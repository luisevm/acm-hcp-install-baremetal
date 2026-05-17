
## Hosted Control Planes installation

Automating the deployment of a Hosted Control Plane (HCP / HyperShift) via RHACM requires a different approach than standard bare-metal clusters. **You do not use ZTP, SiteConfig, or ClusterInstance.** Instead, you sync the native HyperShift Custom Resources alongside your hardware inventory — directly via `oc apply` for a lab, or via OpenShift GitOps (ArgoCD) for production.

> **Architectural Note: Late Binding vs. Early Binding**
> * **Standard AI/ZTP (Early Binding):** A server powers on, and the system explicitly says, *"You belong to Edge-Cluster-01"* (via `InfraEnv.spec.clusterRef`).
> * **HCP Automation (Late Binding):** Your Git repository (or `oc apply` flow) defines a generic hardware inventory pool (`InfraEnv` without a `clusterRef`). Servers sit in a "Ready" state. A `NodePool` later asks for *N workers from inventory*; the HCP provider reaches into the pool and binds matching Agents to the cluster.

### Lab Topology

This procedure simulates bare metal with KVM VMs and **sushy-emulator** providing the Redfish endpoint that BMO drives.

    ┌──────────────────────────────┐         ┌───────────────────────────────┐
    │   Hub / Management cluster   │         │      KVM Host (HP Z420)    │
    │                              │         │        192.168.1.102
                                   │
    │  MCE + hypershift component  │ Redfish │   sushy-emulator :8000        │
    │  Bare Metal Operator (BMO) ──┼───────▶│   libvirt                     │
    │  Assisted Service            │   HTTP  │   br0  (bridged to LAN)       │
    │  HyperShift Operator         │         │                               │
    │  ACM hub                     │         │   ┌────────────────────┐      │
    │                              │  ISO    │   │ VM hcp-worker-0    │      │
    │  Assisted Image Service ─────┼─────────┼─▶│ (empty CDROM,      │      │
    │  serves the discovery ISO    │  mount  │   │  UEFI boot)        │      │
    └──────────────────────────────┘         │   └────────────────────┘      │
                                             └───────────────────────────────┘

Reachability requirements:
- The hub must reach `IP_REDFISH:8000` on the KVM host (BMC commands).
- The KVM VMs must reach the hub's Assisted Image Service route (ISO download) and Assisted Service route (Agent registration).
- DNS must resolve `api.<HCP_NAME>.<DOMAIN>` and `*.apps.<HCP_NAME>.<DOMAIN>` to the hub's API exposure (NodePort target or MetalLB VIP).

---

### Prerequisites

#### Hub-cluster prerequisites

| Requirement | Why it matters |
|---|---|
| MCE installed with the `hypershift` component enabled | Provides the HyperShift Operator. |
| Bare Metal Operator (BMO) available | Reconciles BareMetalHost CRs. Ships with MCE. |
| `Provisioning` CR with `watchAllNamespaces: true` | Wakes BMO. Without it, BMHs outside `openshift-machine-api` are silently ignored. |
| `AgentServiceConfig` deployed | Hosts the Assisted Service that bakes discovery ISOs. |
| DNS for `api.<HCP_NAME>.<DOMAIN>` and `*.apps.<HCP_NAME>.<DOMAIN>` | Workers and clients reach the hosted control plane through these names. |
| MetalLB *or* external LB *or* NodePort + external DNS plan | The HCP API server is exposed from the ACM hub, not from the Hosted cluster workers. |
| Storage class for `AgentServiceConfig` PVCs (LVMO/ODF/etc.) | Holds the database and every generated discovery ISO. |

#### DNS records

Create the following A records in your lab DNS (BIND, dnsmasq, etc.) before applying any manifests:

| Record | Target IP | Purpose |
|---|---|---|
| `api.${HCP_NAME}.${DOMAIN}` (`api.hcpc1.ocp4.home.levmdomain.com`) | `${IP_HUB_API}` = `192.168.3.240` | Hosted-cluster API server. Resolves to the **ACM hub** node IP, because the HCP API server runs as pods on the hub. |
| `*.apps.${HCP_NAME}.${DOMAIN}` (`*.apps.hcpc1.ocp4.home.levmdomain.com`) | `${IP_NODE0}` = `192.168.1.160` | Hosted-cluster Metal-LB IPAdreesPools, used to ingress. Routes resolve here; ingress runs on the hosted cluster worker. |
| `${HOSTNAME_NODE0}.${DOMAIN}` (`hcpc1.ocp4.home.levmdomain.com`) | `${IP_NODE0}` = `192.168.1.247` | Worker node hostname (FQDN). Used by the kubelet for self-identification. |

#### KVM-host prerequisites - install sushy-emulator

KVM has no native Redfish. `sushy-emulator` exposes one against libvirt so BMO can mount the discovery ISO and power-cycle each VM.

```bash
sudo dnf install -y python3-pip libvirt httpd-tools
sudo pip3 install sushy-tools

# Credentials file — must match the BMC Secret created in Step 2c
# (username=admin, password=Admin123!)
sudo htpasswd -bcB /etc/sushy-emulator.htpasswd admin 'Admin123!'
sudo chmod 600 /etc/sushy-emulator.htpasswd

sudo tee /etc/sushy-emulator.conf > /dev/null <<'EOF'
SUSHY_EMULATOR_LIBVIRT_URI = "qemu:///system"
SUSHY_EMULATOR_LISTEN_IP   = "0.0.0.0"
SUSHY_EMULATOR_LISTEN_PORT = 8000
SUSHY_EMULATOR_AUTH_FILE   = "/etc/sushy-emulator.htpasswd"
SUSHY_EMULATOR_BOOT_LOADER_MAP = {
  "Uefi":   {"x86_64": "/usr/share/OVMF/OVMF_CODE.secboot.fd"},
  "Legacy": {"x86_64": None}
}
EOF

sudo tee /etc/systemd/system/sushy-emulator.service > /dev/null <<'EOF'
[Unit]
Description=Sushy Redfish Emulator
After=libvirtd.service
[Service]
ExecStart=/usr/local/bin/sushy-emulator --config /etc/sushy-emulator.conf
Restart=always
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now sushy-emulator

# Open the sushy port on firewalld so the hub can reach it
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload

# Verify locally (anonymous call should now return 401), then with credentials
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:8000/redfish/v1/Systems
# expect: 401

curl -sS -u admin:'Admin123!' http://localhost:8000/redfish/v1/Systems | jq .
# expect: a Members list (possibly empty if no VMs yet)

# From the hub (or any node on 192.168.3.0/24):
#   curl -sS -u admin:'Admin123!' http://192.168.1.102:8000/redfish/v1/Systems | jq .
```

The `SUSHY_EMULATOR_BOOT_LOADER_MAP` is the line most lab guides forget — without it, UEFI guests boot the OVMF firmware menu, never reach the virtual media, and BMH provisioning hangs.

> **⚠️ Operational Gotcha — UUID, not name:** sushy-emulator addresses each system by its libvirt **domain UUID**, not its name. After you create a VM, grab the UUID with `sudo virsh dominfo <name> | awk '/UUID/{print $2}'` and put that UUID in the BMH's `bmc.address`.

---

### Environment variables

Set these once. **Every later step references them — keep names consistent.**

```bash
# Network
export DOMAIN='ocp4.home.levmdomain.com'
export IP_DNS_SERVER='192.168.1.1'
export IP_DF_GW='192.168.1.1'

# Hub / inventory
export INFRAENV_NAMESPACE='hardware-inventory'
export INFRAENV_NAME='baremetal-pool-1'

# Worker node 0
export HOSTNAME_NODE0='hcpc1'
export MAC_NODE0='52:54:00:AB:CD:01'
export IP_NODE0='192.168.1.247'
export VM_UUID_NODE0=''            # populated after virt-install (Step 5)

# KVM host (remote — Step 5 runs here)
export KVM_HOST_USER='root'                          # SSH user on the KVM host
export KVM_HOST='192.168.1.102'                      # SSH hostname/IP of the KVM host
export VIRSH_URI="qemu+ssh://${KVM_HOST_USER}@${KVM_HOST}/system"   # libvirt remote URI

# sushy-emulator (running on the KVM host)
export IP_REDFISH='192.168.1.102'   # KVM host IP — sushy-emulator listens here
export REDFISH_PORT='8000'

# Hosted cluster
export HCP_NAME='hcpc1'                                                             # HostedCluster name; control-plane pods land in ${HCP_NAMESPACE}-${HCP_NAME}
export HCP_NAMESPACE='clusters'                                                    # Hub namespace holding HostedCluster/NodePool CRs
export OCP_RELEASE='quay.io/openshift-release-dev/ocp-release:4.21.9-x86_64'       # OCP payload for control plane + workers (must agree with AgentServiceConfig osImages)
export SSH_PUB_FILE=~/.ssh/id_ed25519.pub                                              # Public key baked into discovery ISO and final RHCOS install
export IP_HUB_API='192.168.3.240'                                                  # ACM hub node IP that exposes the HCP APIServer NodePort. DNS record api.${HCP_NAME}.${DOMAIN} must point here.
export CLUSTER_NETWORK_CIDR='10.132.0.0/14'                                        # Pod network for the hosted cluster
export SERVICE_NETWORK_CIDR='172.31.0.0/16'                                        # Service network for the hosted cluster
export MACHINE_NETWORK_CIDR='192.168.1.0/24'                                       # Network the worker nodes live on (matches br0/IP_NODE0)

# Hub kubeconfig
export KUBECONFIG_PATH='/mnt/vm/Var/csa-pse/csa/mydocuments/clusters/sno/acmvm/'
export KUBECONFIG_HUB=${KUBECONFIG_PATH}/mgmt-kubeconfig
export KUBECONFIG_HOSTED=${KUBECONFIG_PATH}/hosted-kubeconfig

export CHANNEL='stable-4.21'     # Channel to install from

```

---

### Step 1: ACM Hub-cluster setup (one-time)

#### Step 1a — Enable the `hypershift` component

```bash
oc patch mce multiclusterengine --type=merge -p '
{"spec":{"overrides":{"components":[{"name":"hypershift","enabled":true}]}}}'
```

#### Step 1b - Provisioning CR (wakes BMO across all namespaces)

```bash
cat <<EOF | oc apply -f -
apiVersion: metal3.io/v1alpha1
kind: Provisioning
metadata:
  name: provisioning-configuration
spec:
  provisioningNetwork: Disabled
  watchAllNamespaces: true
EOF
```

#### Step 1c — AgentServiceConfig (deploys Assisted Service)

> The lab uses LVMO via `storageClassName: lvms-vg1`. Production should use ODF or another RWO-capable storage class. `filesystemStorage` holds every generated Discovery ISO — size accordingly.

This will download the ISOs.

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  namespace: multicluster-engine
  name: agent
spec:
  databaseStorage:
    storageClassName: lvms-vg1
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 10Gi
  filesystemStorage:
    storageClassName: lvms-vg1
    accessModes: [ReadWriteOnce]
    resources:
      requests:
        storage: 50Gi
  osImages:
    - openshiftVersion: "4.21"
      url:       "https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.21/latest/rhcos-live.x86_64.iso"
      rootFSUrl: "https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.21/latest/rhcos-live-rootfs.x86_64.img"
      cpuArchitecture: "x86_64"
      version: "<RHCOS_VERSION>"        # fill in from sha256sum.txt in the mirror dir, e.g. 421.94.YYYYMMDDxxxx-0
EOF
```

**parameter configuration:** `osImages`:
- In disconnected/air-gapped environments, it is mandatory to configure this parameter because the operator cannot download images from the internet, so you must point it to your local mirror.
- For connected clusters, it is not required. If the osImages is left empty in the AgentServiceConfig, all OCP versions supported, are downloaded in all architectures. This takes up a lot of unnecessary space for most use cases.

**Version alignment:** `osImages.openshiftVersion`
- (`4.21`) **must match** the major.minor in `OCP_RELEASE` (`4.21.9`). If they mismatch, the install hangs in pre-flight validation with no obvious error.

- **Fill in `<RHCOS_VERSION>`:** the RHCOS patch version that ships with OCP 4.21.9 is not 1:1 with the OCP version. Look it up from the mirror — the value is in the filename listed at `sha256sum.txt`:
    ```bash
    curl -sS https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.21/latest/sha256sum.txt \ | awk '/rhcos-.*-live.x86_64.iso$/{print $2}' | sed 's/.*rhcos-//; s/-x86_64.*//'
    ```
    Paste that string (e.g. `421.94.202511121234-0`) into the `version` field. Alternatively, pin to a specific RHCOS patch directory (`/4.21/4.21.x/rhcos-4.21.x-...`) instead of `latest/` if you want immutable URLs.
    
> **⚠️ iPXE + HTTPS prerequisite:** When the discovery ISO is the default "minimal" type, iPXE fetches the rootfs from the assisted-image-service at boot over HTTPS. iPXE has its own compiled-in trust list (essentially the Mozilla CA bundle) and does **not** read `InfraEnv.spec.additionalTrustBundle` or the cluster's `trustedCA`. If your hub ingress uses the default self-signed certificate, iPXE rejects the HTTPS handshake and the boot stalls at the iPXE prompt with no clear error.
>
> Two acceptable ways to satisfy iPXE:
> 1. **(Recommended)** Front the hub ingress with a certificate that chains to a CA already in iPXE's trust list — e.g. Let's Encrypt via `cert-manager` with the DNS-01 challenge. The InfraEnv ISO will then need to be regenerated (`infraenv.agent-install.openshift.io/recreate-iso` annotation) so iPXE sees the new cert.
> 2. **(Lab escape hatch)** Set `iPXEHTTPRoute: enabled` on this `AgentServiceConfig` to expose the boot artifacts over plain HTTP, bypassing TLS entirely. Acceptable on a trusted lab network; **not** for production.
>
> This procedure assumes option 1 is in place. If you haven't set up cert-manager yet and want to unblock the install temporarily, add `iPXEHTTPRoute: enabled` under `spec:` above and re-apply.

There is a third option to boot the OS, which is using a full ISO rootfs, which is mounted into the CDROM. This options requires configuring the parater `iSOImageType`, on the `InfaEnv` CR.

**Verify:**
This may take a while as it will download the images
```bash
oc --kubeconfig $KUBECONFIG_HUB -n multicluster-engine logs assisted-image-service-0 
```

```bash
oc --kubeconfig $KUBECONFIG_HUB -n multicluster-engine wait --for=condition=DeploymentsHealthy \
   agentserviceconfig/agent --timeout=300s

oc --kubeconfig $KUBECONFIG_HUB -n multicluster-engine get pods \
   --selector 'app in (assisted-image-service,assisted-service)'
# expect: assisted-image-service-0  1/1  Running
# expect: assisted-service-xxxx     2/2  Running
```

---

### Step 2: Hardware-inventory namespace and secrets

#### Step 2a — Namespace

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${INFRAENV_NAMESPACE}
EOF
```

#### Step 2b — Pull secret (copied from hub, renamed)

```bash
oc --kubeconfig $KUBECONFIG_HUB -n openshift-config get secret pull-secret -o json \
  | jq 'del(.metadata.namespace, .metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp)
        | .metadata.name = "pull-secret-inventory"' \
  | oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} apply -f -
```

The HostedCluster (Step 8) needs a separate copy in `${HCP_NAMESPACE}` — we'll handle that there.

#### Step 2c — BMC credentials secret

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: bmc-secret-${HOSTNAME_NODE0}
  namespace: ${INFRAENV_NAMESPACE}
type: Opaque
stringData:
  username: admin
  password: 'Admin123!'
EOF
```

> Use **`stringData`**, not `data`. `data` requires base64; mixing them silently feeds BMO garbage credentials.

> **⚠️ Deletion of BMH deletes BMC Secret:** When you create the BareMetalHost in Step 6, the metal3 controller adds an `ownerReference` on this Secret pointing at the BMH (it does this automatically for credentials secrets that don't already have an owner). Consequence: **deleting the BareMetalHost cascades a delete of this Secret too**, even though you created it separately. If you rebuild the BMH later (e.g. as part of a partial teardown — see [Partial teardown: rebuild inventory only](#partial-teardown-rebuild-inventory-only)), you must re-apply this Secret first or `bmc.credentialsName` will dangle.

---

### Step 3: NMStateConfig (per-host static IP)

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: ${HOSTNAME_NODE0}
  namespace: ${INFRAENV_NAMESPACE}
  labels:
    infraenvs.agent-install.openshift.io: ${INFRAENV_NAME}
spec:
  config:
    interfaces:
      - name: ens3
        type: ethernet
        state: up
        mac-address: ${MAC_NODE0}
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: ${IP_NODE0}
              prefix-length: 24
    dns-resolver:
      config:
        server:
          - ${IP_DNS_SERVER}
    routes:
      config:
        - destination: 0.0.0.0/0
          next-hop-address: ${IP_DF_GW}
          next-hop-interface: ens3
          table-id: 254
  interfaces:
    - name: ens3
      macAddress: ${MAC_NODE0}
EOF
```

The `infraenvs.agent-install.openshift.io: ${INFRAENV_NAME}` label is **the** join key — the InfraEnv's `nmStateConfigLabelSelector` (Step 4) pulls this NMStateConfig into its discovery ISO bundle. The MAC inside `spec.interfaces` matches this NMStateConfig to the booted host at agent registration.

---

### Step 4: InfraEnv (generates Discovery ISO)

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: ${INFRAENV_NAME}
  namespace: ${INFRAENV_NAMESPACE}
spec:
  pullSecretRef:
    name: pull-secret-inventory
  sshAuthorizedKey: "$(cat ${SSH_PUB_FILE})"
  cpuArchitecture: x86_64
  nmStateConfigLabelSelector:
    matchLabels:
      infraenvs.agent-install.openshift.io: ${INFRAENV_NAME}
EOF
```

Note the **omission of `clusterRef`** — this is the late-binding hallmark.

#### What the InfraEnv is responsible for (and what it is not)

InfraEnv owns a **per-InfraEnv discovery ISO** artifact. Everything else used during boot (kernel, rootfs, OS install image) comes from elsewhere.

✅ **The InfraEnv triggers the assisted-image-service to bake a discovery ISO** with these things compiled into its kernel cmdline / initramfs:

| Field on `InfraEnv.spec` | What it injects into the ISO |
|---|---|
| `pullSecretRef` | The container-registry pull secret the agent uses post-boot |
| `sshAuthorizedKey` | Public key authorized for `ssh core@…` on the booted live system |
| `nmStateConfigLabelSelector` | Bundles the matching NMStateConfig(s) so the agent applies static networking on boot |
| `additionalTrustBundle` | CAs trusted by the agent *userspace* (NOT by iPXE — iPXE has its own compile-time CA list) |
| `proxy` | HTTP/HTTPS proxy used by the agent post-boot |
| `clusterRef` | Early-binding marker. Omitting it = late binding (this lab) |
| `cpuArchitecture` | Picks the matching osImages entry from AgentServiceConfig |
| `iSOImageType` | `minimal-iso` (default, ~120 MB) or `full-iso` (~1.2 GB, rootfs bundled in) |

The InfraEnv triggers an ISO bake when the CR is created or when annotated with `infraenv.agent-install.openshift.io/recreate-iso=<timestamp>`. The result is exposed at `status.isoDownloadURL`.

❌ **The InfraEnv does NOT create or own:**

| Artifact | File type | Where it actually comes from |
|---|---|---|
| **RHCOS live rootfs** | `.img` (squashfs) — *not* an ISO | `AgentServiceConfig.spec.osImages[].rootFSUrl`. Pre-built by Red Hat. Downloaded once per OCP version by assisted-image-service, cached on its `filesystemStorage` PVC, served identically to every InfraEnv. |
| **RHCOS live kernel** | kernel image | Same source — comes from osImages alongside the rootfs. |
| **OCP cluster install image** | OCI release payload | `HostedCluster.spec.release.image`. The agent's `coreos-installer` pulls this from `quay.io/openshift-release-dev` and writes it to `/dev/sda` once a NodePool binds the agent. |

#### `minimal-iso` vs `full-iso`: same InfraEnv config, different rootfs delivery

| | `minimal-iso` (default) | `full-iso` |
|---|---|---|
| ISO size | ~120 MB | ~1.2 GB |
| Contents | iPXE + kernel + initramfs + InfraEnv config | Same + rootfs bundled inside |
| Rootfs delivery | Downloaded at boot over HTTPS from `assisted-image-service` | Mounted directly from the CDROM, no network fetch |
| When to use | Production with a trusted (Let's Encrypt etc.) cert on hub ingress | Air-gapped labs, or homelabs without trusted apps cert |

Switch from minimal to full ISO, by patching the InfraEnv:

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE patch infraenv $INFRAENV_NAME \
   --type=merge -p '{"spec":{"iSOImageType":"full-iso"}}'

# Force regeneration of the ISO with the new type
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE annotate infraenv $INFRAENV_NAME \
   "infraenv.agent-install.openshift.io/recreate-iso=$(date +%s)" --overwrite
```

---

### Step 5: Create the KVM "bare metal" VM(s)

Empty CDROM, fixed MAC, no install media, **powered off** — BMO will mount the discovery ISO and power it on via Redfish later (Step 6).

This step runs **on the KVM host**, but your env vars are exported on your laptop. Two ways to do it without leaving your laptop:

#### Step 5a (one-time) — Trust your laptop's SSH key on the KVM host

So later steps don't prompt for a password:

```bash
# On your laptop: create a key if you don't have one
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519

# Push it to the KVM host (you'll be prompted for the password ONCE)
ssh-copy-id -i ~/.ssh/id_ed25519.pub ${KVM_HOST_USER}@${KVM_HOST}

# Verify: should log you in without prompting
ssh ${KVM_HOST_USER}@${KVM_HOST} 'echo OK'
```

If `${KVM_HOST_USER}` is **not** `root` and you want libvirt access without `sudo`, also add the user to the `libvirt` group on the KVM host:

```bash
ssh ${KVM_HOST_USER}@${KVM_HOST} \
   "sudo usermod -aG libvirt ${KVM_HOST_USER} && sudo systemctl restart libvirtd"
# Log out and back in for the group change to take effect, then verify:
ssh ${KVM_HOST_USER}@${KVM_HOST} 'virsh list --all'
```

If you keep `${KVM_HOST_USER}=root`, skip the group step — root already has full libvirt access.

#### create VM - libvirt remote URI
also configure passwordless `sudo` for libvirt operations if you're not root, or stick to root for simplicity.

`virt-install` and `virsh` both accept `--connect <uri>`, so you can target the remote KVM host with locally-expanded env vars. Requires:
- SSH key-based access to `${KVM_HOST_USER}@${KVM_HOST}` (done in Step 5a above).
- The user can talk to `qemu:///system` on the remote (root, or in the `libvirt` group).
- `virt-install` installed **on your laptop** (`sudo dnf install -y virt-install` on Fedora).

```bash
# Pre-flight: prove the remote libvirt is reachable
virsh --connect ${VIRSH_URI} list --all

# Create the VM (defined + powered off) — runs locally, talks to remote libvirt
virt-install --connect ${VIRSH_URI} \
  --name ${HOSTNAME_NODE0} \
  --ram 16384 \
  --vcpus 4 \
  --disk path=/var/lib/libvirt/images/${HOSTNAME_NODE0}.qcow2,size=100,format=qcow2 \
  --disk device=cdrom,bus=sata \
  --os-variant rhel9.7 \
  --network bridge=br0,mac=${MAC_NODE0} \
  --console pty,target_type=serial \
  --boot uefi,hd,cdrom,menu=on \
  --noautoconsole \
  --noreboot \
  --install no_install=yes

# Safety net: shut it down if virt-install left it running
virsh --connect ${VIRSH_URI} destroy ${HOSTNAME_NODE0} 2>/dev/null || true

# Confirm defined and powered off
virsh --connect ${VIRSH_URI} list --all | grep ${HOSTNAME_NODE0}
# expect: "shut off"

# Capture the libvirt UUID (used in the BMH Redfish URL at Step 6)
export VM_UUID_NODE0=$(virsh --connect ${VIRSH_URI} dominfo ${HOSTNAME_NODE0} \
                       | awk '/UUID/{print $2}')
echo "VM UUID: $VM_UUID_NODE0"
```

**Verify sushy can see it:**

```bash
curl -sS -u admin:'Admin123!' http://${IP_REDFISH}:${REDFISH_PORT}/redfish/v1/Systems/${VM_UUID_NODE0} | jq .Name
# expect: "hcpc1"
```

---

### Step 6: BareMetalHost + Role

#### What this step does

A `BareMetalHost` (BMH) is the Kubernetes representation of a physical machine and its BMC. Creating the BMH **triggers the Bare Metal Operator (BMO) and the Bare Metal Agent Controller (BMAC) inside assisted-service to run an automated boot/discovery sequence**:

1. **BMO claims the host.** Reads `spec.bmc.address` and the referenced credentials Secret, opens a Redfish session against sushy-emulator on the KVM host.
2. **BMAC creates a `PreprovisioningImage` CR** owned by this BMH. It picks the discovery ISO URL from the InfraEnv whose name matches the BMH's `infraenvs.agent-install.openshift.io` label and writes it into the PPI's `status.imageUrl`.
3. **BMO calls Redfish `InsertMedia`** with that ISO URL — sushy attaches the ISO as a virtual CDROM to the libvirt domain identified by `${VM_UUID_NODE0}`.
4. **BMO sets `BootSourceOverrideTarget: Cd, Enabled: Continuous`** so the firmware boots the CDROM on every power cycle.
5. **BMO calls Redfish `Reset` (PowerOn).** sushy powers on the libvirt domain.
6. **The VM boots iPXE from the ISO,** fetches the RHCOS kernel + rootfs, and starts the **agent service**.
7. **The agent registers** with assisted-service over HTTPS, sending its hardware inventory.
8. **BMAC matches the registered Agent to this BMH** by MAC address (`spec.bootMACAddress`) and creates the `Agent` CR in the same namespace, owned by the BMH.
9. **BMH transitions through states:** `registering` → `inspecting` → `provisioning` → `provisioned`. The agent stays in this discovery loop, waiting for a NodePool (Step 8) to claim it.

Without the BMH, nothing happens — no ISO is attached, no power-on, no Agent. The BMH is the single trigger that drives steps 2–9 automatically.

#### Boot sequence: three images, not one

A common misconception is that there's a single "RHCOS image" involved. There are actually **three distinct images**, each pulled at a different phase. Understanding which one is currently downloading explains why some phases of provisioning are fast and others slow.

| # | Image | Where it lives | Size | When it's used | Touches disk? |
|---|---|---|---|---|---|
| 1 | **Discovery ISO** (minimal variant) — `.iso` (ISO 9660). **Per-InfraEnv**: baked by `assisted-image-service` with the pull secret, ssh key, NMStateConfig, trust bundle, etc. compiled in (see Step 4). | Sushy's virtual CDROM, attached via Redfish | ~120 MB | UEFI boots from CDROM | No — it's the ISO |
| 2 | **Live rootfs** (RHCOS live root) — `.img` squashfs, **not an ISO**. Pre-built by Red Hat, sourced from `AgentServiceConfig.osImages[].rootFSUrl`, identical for every InfraEnv on this OCP version (see Step 4 for the InfraEnv-vs-rootfs ownership split). | Downloaded by iPXE at boot, over HTTPS from `assisted-image-service` (or bundled in the ISO if `iSOImageType: full-iso`) | ~1.2 GB | iPXE loads it into RAM | No — runs entirely in RAM |
| 3 | **Cluster RHCOS install image** | Downloaded by `coreos-installer` inside the live system, from the OCP release payload on `quay.io` | ~3–4 GB | Written to `/dev/sda` after the agent is bound to a cluster | **Yes** — clones the OS onto the disk |

##### Full sequence, end to end

```
[ Step 6 — BMH applied ]
       │
       ▼
1. BMO calls Redfish InsertMedia(image=ISO #1) on sushy
       │
       ▼
2. BMO calls Redfish PowerOn
       │
       ▼
3. VM firmware (UEFI) boots from CDROM   ─── reads ISO #1
       │
       ▼
4. iPXE script in ISO runs:
   "kernel http://... rootfs=http://..."  ─── downloads rootfs IMAGE #2
                                              (NOT created by InfraEnv — pre-built
                                              by Red Hat, served by assisted-image-service)
       │
       ▼
5. Kernel boots in RAM, mounts rootfs from RAM
       │
       ▼
6. RHCOS live userspace starts; the 'agent' service launches
       │
       ▼
7. Agent registers with assisted-service          [ end state of Step 6: STAGE empty ]
       │
       ▼
[ Step 8 — NodePool created → it claims this Agent ]
       │
       ▼
8. Agent receives "install this OCP release"
       │
       ▼
9. coreos-installer downloads IMAGE #3  ─── writes to /dev/sda
       │
       ▼
10. Agent reports STAGE=writing-image-to-disk → rebooting
       │
       ▼
11. VM reboots — BMO has set BootSourceOverride to Hdd for next boot
       │
       ▼
12. VM boots from /dev/sda (the freshly-written RHCOS install)
       │
       ▼
13. Ignition runs, kubelet starts, node joins the hosted control plane
       │
       ▼
14. STAGE: joined → done
```

##### Three terminology gotchas

- **"PXE" usually means iPXE here.** Traditional PXE (DHCP + TFTP) isn't used in this flow — the iPXE bootloader is baked into the discovery ISO and runs from the CDROM.
- **The live rootfs (#2) never touches the hard disk.** It runs entirely in RAM. Don't expect a disk-write event in phases 4–7.
- **Only ONE reboot happens in the normal flow** — between phases 10 and 11, after `coreos-installer` finishes writing image #3 to disk. If you see multiple reboots, something's wrong (UEFI NVRAM stuck, boot order issue, or install failure).

#### The manifest

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: ${HOSTNAME_NODE0}
  namespace: ${INFRAENV_NAMESPACE}
  labels:
    infraenvs.agent-install.openshift.io: ${INFRAENV_NAME}
  annotations:
    inspect.metal3.io: disabled
    bmac.agent-install.openshift.io/hostname: ${HOSTNAME_NODE0}
spec:
  online: true
  automatedCleaningMode: disabled
  bootMACAddress: ${MAC_NODE0}
  bmc:
    address: redfish-virtualmedia+http://${IP_REDFISH}:${REDFISH_PORT}/redfish/v1/Systems/${VM_UUID_NODE0}
    credentialsName: bmc-secret-${HOSTNAME_NODE0}
    disableCertificateVerification: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: capi-provider-role
  namespace: ${INFRAENV_NAMESPACE}
rules:
  - apiGroups: [agent-install.openshift.io]
    resources: [agents]
    verbs: ['*']
EOF
```

**Why `redfish-virtualmedia+http://`:** sushy-emulator runs plain HTTP in the lab config above. For real hardware (Dell iDRAC, HPE iLO) use `+https://`. The `+<scheme>` suffix is mandatory — generic `redfish-virtualmedia://` silently fails on most implementations.

**Verify BMH progress:**

```bash
watch "oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} get bmh,agent"

# Expected progression over ~5–10 min:
#   bmh:   registering → inspecting → provisioning → provisioned
#   agent: appears after the discovery ISO boots and registers
```

##### Monitoring tips — what to watch at this stage

At this point in the procedure only the **BareMetalHost** and (once it registers) the **Agent** exist in the inventory namespace. The HostedCluster and NodePool are created later in Step 8, so we monitor those in Step 9 — not here.

###### A. Inventory dashboard (BMH + Agent + recent events)

```bash
watch -n 5 "
echo '=== BMH ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get bmh ${HOSTNAME_NODE0}
echo
echo '=== Agent (note the STAGE column — this is the real progress signal) ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents -o wide
echo
echo '=== Last 5 events ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get events --sort-by=.lastTimestamp | tail -5
"
```

The Agent's `STAGE` column is the single most useful progress indicator. Expected sequence:

| STAGE | What's happening | Typical duration |
|---|---|---|
| (empty) | Agent has registered but is not bound to a cluster yet (waiting for NodePool in Step 8) | until you reach Step 8 |
| `discovering` | Bound — gathering hardware info | 30s–2 min |
| `installing` | Install scheduled | a few seconds |
| `writing-image-to-disk` | RHCOS payload being written to `/dev/sda` | 1–3 min |
| `rebooting` | Agent reboots from disk | 30s |
| `configuring` | Ignition + kubelet starting | 1–3 min |
| `joined` | Node has joined the hosted control plane | a few seconds |
| `done` | Install complete | terminal state |

At this point in the procedure, expect the Agent to appear with **STAGE blank** (registered, not yet bound). It won't advance past blank/`discovering` until Step 8 creates the NodePool — that's normal.

###### B. Detailed Agent progress (sub-stage info `get agents` doesn't show)

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents \
   -o jsonpath='{range .items[*]}{.metadata.name}{"  stage="}{.status.progress.currentStage}{"  stageStarted="}{.status.progress.stageStartTime}{"  percent="}{.status.progress.installationPercentage}{"\n"}{end}'
```

Tells you exactly when the current stage started and (during install) how far through it the agent is. Re-run every 30s once the install is actually running.

###### C. Live install log from the agent itself (via VM console)

The most ground-truth view during boot and the install stages — see what the agent's own service is doing:

```bash
virsh --connect ${VIRSH_URI} console ${HOSTNAME_NODE0}
# In the VM:
sudo journalctl -fu agent
# Ctrl+] to detach the console
```

###### D. Hub-side: assisted-service log filtered to this host

If you don't want to console in:

```bash
oc --kubeconfig $KUBECONFIG_HUB -n multicluster-engine logs deploy/assisted-service -c assisted-service \
   --tail=200 -f | grep -iE "${HOSTNAME_NODE0}|$(oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents -o name | head -1 | cut -d/ -f2)"
```

###### E. Just the events, sorted (cheapest signal)

```bash
watch -n 5 "oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get events --sort-by=.lastTimestamp | tail -10"
```

###### Rule of thumb at this stage

- BMH reaches `provisioning` and Agent appears (STAGE blank) within ~5–10 min → healthy. Proceed to Step 7.
- BMH stuck in `registering` or `inspecting` for >5 min → jump to "Diagnostic checks" below.
- Agent never appears after 10 min → VM didn't reach assisted-service; check Step 6.5 (VM console) in the diagnostics section.

##### Expected end state of Step 6

This is what "Step 6 finished successfully" looks like — the BMH is permanently parked in `provisioning` and the Agent in empty `STAGE`, waiting for Step 8 to give it work. **Do not wait for STAGE to advance here — it won't.**

```text
NAME                            STATE          CONSUMER   ONLINE   ERROR   AGE
baremetalhost.metal3.io/hcpc1   provisioning              true             22m

NAME                                                                    CLUSTER   APPROVED   ROLE          STAGE
agent.agent-install.openshift.io/<uuid>                                           true       auto-assign
```

Key signals to look for before moving on:

| Field | Expected value | What it means |
|---|---|---|
| `bmh.STATE` | `provisioning` | BMO has mounted the ISO and powered the VM on; the agent is alive |
| `bmh.ONLINE` | `true` | BMC reports the host as powered on |
| `bmh.ERROR` | (empty) | No registration/credential errors |
| `agent.<uuid>` | row exists | The agent inside the VM phoned home to assisted-service |
| `agent.APPROVED` | `true` | BMAC auto-approved via the BMH→Agent association |
| `agent.CLUSTER` | (empty) | **Expected** — no cluster has claimed this agent yet |
| `agent.STAGE` | (empty) | **Expected** — agent waiting in discovery until Step 8 binds it |

If your output matches this shape, **proceed to Step 7** to formally confirm, then **Step 8** to create the HostedCluster + NodePool that actually puts the agent to work.

> **⚠️ Operational Gotcha — Hardware Validation Black Hole:** If a VM has an undersized disk or missing NIC, the Agent registers but the NodePool hangs forever with no surface error. Inspect with `oc describe agent <id> -n ${INFRAENV_NAMESPACE}` and look at `Validations Info`.

#### Step 6 — Diagnostic checks if provisioning stalls

If the BMH sits in `provisioning` for more than ~10 minutes, walk through these layered checks. Stop where something looks wrong.

##### 6.1 — Single-pane "what's stuck" dashboard

```bash
watch -n 5 "
echo '=== BMH ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get bmh $HOSTNAME_NODE0
echo
echo '=== Agent ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents -o wide
echo
echo '=== VM ==='
virsh --connect $VIRSH_URI domstate $HOSTNAME_NODE0
echo
echo '=== Last 5 events ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get events \
  --sort-by=.lastTimestamp --field-selector involvedObject.name=$HOSTNAME_NODE0 | tail -5
"
```

##### 6.2 — BMH details + events

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE describe bmh ${HOSTNAME_NODE0} | tail -40
```

Look at:
- `Status.Operational Status` (should be `OK`)
- `Status.Powered On` (should match what `virsh domstate` reports)
- `Events:` — anything `Warning` in the last few minutes

##### 6.3 — PreprovisioningImage chain (BMAC-side)

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get preprovisioningimage
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE describe preprovisioningimage ${HOSTNAME_NODE0} | tail -30
```

- `READY=True` with a `Status.Image URL` → BMAC populated the ISO URL.
- `READY=False` or empty URL → BMAC didn't reconcile; check assisted-service:
  ```bash
  oc --kubeconfig $KUBECONFIG_HUB -n multicluster-engine logs deploy/assisted-service -c bmac --tail=200 \
    | grep -iE "${HOSTNAME_NODE0}|hcpc1|error|fail"
  ```

##### 6.4 — Sushy / Redfish layer

```bash
# Is sushy reachable and authenticated?
curl -sS -u admin:'Admin123!' \
  http://${IP_REDFISH}:${REDFISH_PORT}/redfish/v1/Systems/${VM_UUID_NODE0} \
  | jq '{PowerState, Boot}'

# Did sushy actually accept the InsertMedia call?
curl -sS -u admin:'Admin123!' \
  http://${IP_REDFISH}:${REDFISH_PORT}/redfish/v1/Systems/${VM_UUID_NODE0}/VirtualMedia/Cd \
  | jq '{Image, Inserted, ConnectedVia}'
```

Confirm the libvirt domain actually has the ISO attached:

```bash
virsh --connect ${VIRSH_URI} dumpxml ${HOSTNAME_NODE0} | grep -B2 -A4 cdrom
# expect: <source file="/var/lib/libvirt/images/boot-<provisioning-id>-iso-<vm-uuid>.img"/>
```

Sushy logs while reproducing:

```bash
ssh ${KVM_HOST_USER}@${KVM_HOST} \
  'journalctl -u sushy-emulator --since "10 minutes ago" | grep -iE "insert|virtual|error" | tail -40'
```

##### 6.5 — KVM VM console (what's the VM actually doing?)

```bash
If you only see a blank screen, try the graphical console:
virt-viewer --connect ${VIRSH_URI} ${HOSTNAME_NODE0}

virsh --connect ${VIRSH_URI} console ${HOSTNAME_NODE0}
# Ctrl+] to detach
```

##### 6.6 — Agent validations (if Agent CR exists but install hasn't progressed)

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents -o name \
  | xargs -I{} oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE describe {} \
  | grep -A40 -E "Validations Info|Conditions:"
```

Look for `"status":"failure"`. Common culprits and fixes:

| Validation | Fix |
|---|---|
| `sufficient-installation-disk-size` | Grow the qcow2: `qemu-img resize /var/lib/libvirt/images/${HOSTNAME_NODE0}.qcow2 +50G` |
| `sufficient-cpu-cores` / `sufficient-min-cpu-cores-for-role` | Bump `--vcpus` on the VM |
| `sufficient-min-memory-for-role` | Bump `--ram` on the VM |
| `belongs-to-machine-cidr` | Worker IP is outside `${MACHINE_NETWORK_CIDR}` — fix `IP_NODE0` or the CIDR |
| `dns-wildcard-not-configured` | Hub-side DNS wildcard test failing — check `*.apps.${HCP_NAME}.${DOMAIN}` records |
| `ntp-synced` | Worker clock drift — set an NTP server in the NMStateConfig or attach via `chrony` |

##### 6.7 — Assisted-service logs (deeper inspection)

```bash
oc --kubeconfig $KUBECONFIG_HUB -n multicluster-engine logs deploy/assisted-service -c assisted-service \
  --tail=300 | grep -iE "${HOSTNAME_NODE0}|hcpc1|error|fail" | tail -30
```

##### 6.8 — Ironic / metal3 logs (only if BMC layer is the suspect)

Ironic is what BMO uses to talk to the BMC. Ironic = the BMC remote control. Assisted-service = the OS installer.
- Handles four things via Redfish: open BMC session, mount the discovery ISO as virtual media, set boot device to CD, power the host on.
- Normalizes vendor quirks (Dell iDRAC vs HPE iLO vs sushy) so BMO doesn't have to.
- Does not install the OS. The customDeploy: start_assisted_install annotation tells Ironic to do the BMC dance, then hand off to assisted-service.

```bash
oc --kubeconfig $KUBECONFIG_HUB -n openshift-machine-api logs deploy/metal3 -c metal3-ironic \
  --tail=300 | grep -iE "${HOSTNAME_NODE0}|hcpc1|error|fail|virtual.media" | tail -30
```

##### 6.9 — Clean reset if all else fails

```bash
# 1. Force VM off and wipe UEFI + disk state
virsh --connect ${VIRSH_URI} destroy ${HOSTNAME_NODE0} 2>/dev/null || true
ssh ${KVM_HOST_USER}@${KVM_HOST} \
   "rm -f /var/lib/libvirt/qemu/nvram/${HOSTNAME_NODE0}_VARS.fd && \
    qemu-img create -f qcow2 /var/lib/libvirt/images/${HOSTNAME_NODE0}.qcow2 100G"

# 2. Delete the BMH so BMO starts fresh
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE delete bmh ${HOSTNAME_NODE0}

# 3. Force a new ISO if the InfraEnv may be cached
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE annotate infraenv $INFRAENV_NAME \
   "infraenv.agent-install.openshift.io/recreate-iso=$(date +%s)" --overwrite

# 4. Re-apply the BMH manifest from above
```

> **Triage rule of thumb:** if `oc describe bmh` shows a recent event (last 1–2 min), give it a few more minutes — Ironic and BMO retry on their own. If the latest event is over 5 minutes old, **it's stuck** and one of 6.2–6.6 will tell you why.

---

### Step 7: Confirm Agent registration and approval

```bash
oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} get agents
# expect:
#   NAME                                 APPROVED   ROLE          STAGE
#   <uuid>                               true       auto-assign
```

BMO booted the VM via Redfish, so the Agent is auto-approved through the BMH/Agent association — **no manual step is needed in this procedure**.

> **Fallback (only if you booted the ISO manually, without BMO):** approve every unapproved Agent in the inventory namespace in one command. You don't have to copy the UUID by hand.
> ```bash
> for a in $(oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} get agents \
>             -o jsonpath='{range .items[?(@.spec.approved==false)]}{.metadata.name}{"\n"}{end}'); do
>   oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} patch agent "$a" \
>      --type merge -p '{"spec":{"approved":true}}'
> done
> ```

---

### Step 8: HostedCluster + NodePool (ACM-driven, declarative)

ACM deploys HCP by reconciling **HostedCluster** and **NodePool** CRs on the hub — the `hypershift-addon` (installed automatically when the `hypershift` MCE component is enabled in Step 1a) does the rest. The manifests below contain every required field; missing any of them is the most common cause of a stuck install.

#### Step 8a — HCP namespace + pull-secret + ssh-key secret

```bash
# Namespace that holds the HostedCluster/NodePool CRs
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: ${HCP_NAMESPACE}
EOF

# Pull secret in the HCP namespace (HostedCluster.spec.pullSecret references it)
oc --kubeconfig $KUBECONFIG_HUB -n openshift-config get secret pull-secret -o json \
  | jq 'del(.metadata.namespace, .metadata.uid, .metadata.resourceVersion, .metadata.creationTimestamp)
        | .metadata.name = "pull-secret"' \
  | oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} apply -f -

# SSH public key as a Secret (HostedCluster.spec.sshKey references it).
# HyperShift's Agent provider expects the public key under the data key `id_rsa.pub`.
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ${HCP_NAME}-ssh-key
  namespace: ${HCP_NAMESPACE}
type: Opaque
stringData:
  id_rsa.pub: |
    $(cat ${SSH_PUB_FILE})
EOF
```

#### Step 8b — HostedCluster

```bash
cat <<EOF> hc.yaml
apiVersion: hypershift.openshift.io/v1beta1
kind: HostedCluster
metadata:
  name: ${HCP_NAME}
  namespace: ${HCP_NAMESPACE}
  annotations:
    # Auto-import into ACM: the hypershift-addon creates ManagedCluster +
    # KlusterletAddonConfig automatically once the first worker joins.
    cluster.open-cluster-management.io/managedcluster-name: ${HCP_NAME}
spec:
  release:
    image: ${OCP_RELEASE}
  channel: ${CHANNEL}
  pullSecret:
    name: pull-secret
  sshKey:
    name: ${HCP_NAME}-ssh-key
  dns:
    baseDomain: ${DOMAIN}
  networking:
    networkType: OVNKubernetes
    clusterNetwork:
      - cidr: ${CLUSTER_NETWORK_CIDR}
    serviceNetwork:
      - cidr: ${SERVICE_NETWORK_CIDR}
    machineNetwork:
      - cidr: ${MACHINE_NETWORK_CIDR}
  platform:
    type: Agent
    agent:
      agentNamespace: ${INFRAENV_NAMESPACE}     # where BMHs / Agents live
  controllerAvailabilityPolicy: SingleReplica   # lab; production: HighlyAvailable
  infrastructureAvailabilityPolicy: SingleReplica
  etcd:
    managementType: Managed
    managed:
      storage:
        type: PersistentVolume
        persistentVolume:
          size: 8Gi
          # storageClassName: lvms-vg1   # uncomment to pin a specific storage class
  services:
    - service: APIServer
      servicePublishingStrategy:
        type: NodePort                          # lab default; switch to LoadBalancer if MetalLB is on the hub
        nodePort:
          address: ${IP_HUB_API}
    - service: OAuthServer
      servicePublishingStrategy:
        type: Route
    - service: Konnectivity
      servicePublishingStrategy:
        type: Route
    - service: Ignition
      servicePublishingStrategy:
        type: Route
    - service: OIDC
      servicePublishingStrategy:
        type: Route
EOF
```

oc apply -f hc.yaml

#### Step 8c — NodePool

```bash
cat <<EOF | oc --kubeconfig $KUBECONFIG_HUB apply -f -
apiVersion: hypershift.openshift.io/v1beta1
kind: NodePool
metadata:
  name: ${HCP_NAME}
  namespace: ${HCP_NAMESPACE}
spec:
  clusterName: ${HCP_NAME}
  replicas: 1
  release:
    image: ${OCP_RELEASE}                       # MUST match HostedCluster.spec.release.image
  management:
    autoRepair: false                           # lab; production: true
    upgradeType: InPlace                        # or Replace
  platform:
    type: Agent
    agent:
      agentLabelSelector: {}                    # empty selector → claim any Agent in agentNamespace
EOF
```

The empty `agentLabelSelector: {}` matches **any** Agent in `${INFRAENV_NAMESPACE}`. To pin specific hardware, label the BMH or Agent (e.g. `oc label agent <id> role=worker`) and set `matchLabels: {role: worker}` here.

> **API exposure choice:**
> * **`NodePort`** (shown) — works without MetalLB. Requires `nodePort.address` set to a reachable hub-node IP. Your DNS for `api.${HCP_NAME}.${DOMAIN}` must resolve to this IP, and the NodePort range (30000-32767) must be reachable from your workers and clients.
> * **`LoadBalancer`** — change `services[0].servicePublishingStrategy.type` to `LoadBalancer` and remove the `nodePort:` block. Requires MetalLB (or a cloud LB) on the hub. `api.${HCP_NAME}.${DOMAIN}` then points to the assigned VIP.

> **Lab vs. production:**
> * `controllerAvailabilityPolicy` / `infrastructureAvailabilityPolicy: SingleReplica` is appropriate for a single-hub lab. Production uses `HighlyAvailable` (3 replicas of each control-plane component) and requires the hub to have enough capacity.
> * `management.autoRepair: false` keeps a failed worker visible for inspection. Flip to `true` in production for self-healing.

---

### Step 9: Watch the install

Once HostedCluster + NodePool are applied (Step 8), three things happen in parallel:
- HyperShift Operator spins up the control-plane pods in `${HCP_NAMESPACE}-${HCP_NAME}`.
- The CAPI agent provider claims an Agent from `${INFRAENV_NAMESPACE}` and binds it to the NodePool.
- The bound Agent's `STAGE` advances through `installing` → `writing-image-to-disk` → `rebooting` → `configuring` → `joined` → `done`.

#### Quick health check

```bash
# Control plane spin-up (~10–15 min)
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} get hostedcluster ${HCP_NAME} \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}{"\n"}'
# expect: True

# Control-plane pods
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE}-${HCP_NAME} get pods

# NodePool claim status
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} get nodepool ${HCP_NAME}
# expect: DESIRED 1, CURRENT 1, READY 1
```

#### Full progress dashboard (all four layers)

```bash
watch -n 5 "
echo '=== BMH ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get bmh ${HOSTNAME_NODE0}
echo
echo '=== Agent (STAGE column is the real progress signal) ==='
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents -o wide
echo
echo '=== NodePool ==='
oc --kubeconfig $KUBECONFIG_HUB -n $HCP_NAMESPACE get nodepool $HCP_NAME
echo
echo '=== HostedCluster ==='
oc --kubeconfig $KUBECONFIG_HUB -n $HCP_NAMESPACE get hostedcluster $HCP_NAME
echo
echo '=== Control-plane pods ==='
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE}-${HCP_NAME} get pods 2>/dev/null | tail -10
"
```

#### HostedCluster conditions in detail

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $HCP_NAMESPACE get hostedcluster $HCP_NAME \
   -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" "}{.reason}{"\n"}{end}'
```

Look for these to flip to `True` in roughly this order: `EtcdAvailable`, `KubeAPIServerAvailable`, `InfrastructureReady`, `ValidConfiguration`, `Available`, `ClusterVersionSucceeding`.

#### NodePool replicas (data-plane readiness)

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $HCP_NAMESPACE get nodepool $HCP_NAME \
   -o jsonpath='{"desired="}{.spec.replicas}{"  current="}{.status.replicas}{"  ready="}{.status.readyReplicas}{"\n"}'
```

`desired=current=ready` means the worker has joined and is healthy. `current<desired` means the CAPI provider hasn't claimed an Agent yet (usually the Agent's STAGE is still pre-`done`, or RBAC is wrong — see the gotcha below).

#### Detailed Agent progress

```bash
oc --kubeconfig $KUBECONFIG_HUB -n $INFRAENV_NAMESPACE get agents \
   -o jsonpath='{range .items[*]}{.metadata.name}{"  stage="}{.status.progress.currentStage}{"  stageStarted="}{.status.progress.stageStartTime}{"  percent="}{.status.progress.installationPercentage}{"\n"}{end}'
```

#### Rule of thumb

- `HostedCluster.Available=True` and `NodePool ready=1` and Agent `STAGE=done` → install complete; proceed to Step 10.
- A condition or stage advances every 1–2 min → healthy, keep waiting.
- No movement for >5 min → run Step 6's "Diagnostic checks" subsection.

> **⚠️ NodePool stuck at `CURRENT 0`:** The CAPI agent provider running in `${HCP_NAMESPACE}-${HCP_NAME}` needs RBAC to read `Agent` CRs in `${INFRAENV_NAMESPACE}`. The HCP operator generates the matching `RoleBinding` automatically when `agentNamespace` is set on the HostedCluster, paired with the `capi-provider-role` Role from Step 6. If it sits empty, check: `oc -n ${HCP_NAMESPACE}-${HCP_NAME} get rolebinding -A | grep capi`.

---

### Step 10: Extract kubeconfig and verify the hosted cluster

The HCP operator generates a kubeconfig Secret named `${HCP_NAME}-admin-kubeconfig` in `${HCP_NAMESPACE}`. Extract it with plain `oc`:

```bash
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} \
   extract secret/${HCP_NAME}-admin-kubeconfig --to=- --keys=kubeconfig \
   > /tmp/${HCP_NAME}.kubeconfig

oc --kubeconfig /tmp/${HCP_NAME}.kubeconfig get nodes
# expect: 1 node Ready, name = hcpc1

oc --kubeconfig /tmp/${HCP_NAME}.kubeconfig get clusterversion
# expect: VERSION 4.21.9, AVAILABLE True
```

---

### Step 11: Confirm ACM auto-import

The annotation injected in Step 8 tells the `hypershift-addon` to create the `ManagedCluster` automatically once a worker joins.

```bash
oc --kubeconfig $KUBECONFIG_HUB get managedcluster ${HCP_NAME}
# expect: HUB ACCEPTED True, JOINED True, AVAILABLE True

oc --kubeconfig $KUBECONFIG_HUB get klusterletaddonconfig -A | grep ${HCP_NAME}
```

If `ManagedCluster` does not appear within ~5 min after worker join, clear any disable annotation:

```bash
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} annotate hostedcluster ${HCP_NAME} \
   import.open-cluster-management.io/disable-auto-import-
```

(Trailing dash removes the annotation if it was set.)

---

### Accessing the Hosted Cluster using the Kubeconfig
Getting the Kubeconfig and Kubeadmin via the CLI

1.Get pull-secret and kubeadmin
```bash
#Get HCPc1 pull-secret
oc --kubeconfig ${KUBECONFIG_HUB} -n clusters \
    extract secret/hcpc1-admin-kubeconfig --to=- \
    > ${KUBECONFIG_HOSTED}

#The kubeadmin password can be retrieved with the command below.
oc --kubeconfig ${KUBECONFIG_HUB} -n clusters \
    extract secret/hcpc1-kubeadmin-password --to=-
```

2.We can access the HostedCluster now.

```bash
#We need to use the --insecure-skip-tls-verify=true due to the lab setup we have, in a real scenario this shouldn’t be required. 
oc --kubeconfig ${KUBECONFIG_HOSTED} get nodes

#If we check the ClusterVersion it complains about some non-available operators.

oc --insecure-skip-tls-verify=true --kubeconfig ${KUBECONFIG_HOSTED} get clusterversion

#The ClusterOperators list will let us know which operators are not ready.
oc --insecure-skip-tls-verify=true --kubeconfig ${KUBECONFIG_HOSTED} get clusteroperators
```

### Configuring the Hosted Cluster Ingress
In order to provide ingress capabilities to our Hosted Cluster we will use a LoadBalancer service. We will use MetalLB.

Note that the Ingress Controller, is deployed OOTB on the Hosted cluster and MetalLB will be used to ingress traffic to the HCP cluster. 

The MetalLB will be configured for L2, but one could as well use a L3, using BGP to advertize the Hosted cluster  ingress IPs.

1. Let’s get MetalLB Operator deployed.

  - subscription
    ```bash
    cat <<EOF > metallb-deployment.yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: metallb-operator
      namespace: openshift-operators
    spec:
      channel: "stable"
      name: metallb-operator
      source: redhat-operators
      sourceNamespace: openshift-marketplace
    EOF
    
    oc --kubeconfig ${KUBECONFIG_HOSTED} apply -f metallb-deployment.yaml
    sleep 30
    ```

  - Wait for application to be installed
    ```bash
    oc --kubeconfig ${KUBECONFIG_HOSTED} \
        -n openshift-operators wait --for=jsonpath='{.status.state}'=AtLatestKnown \
        subscriptions.operators.coreos.com/metallb-operator --timeout=300s

    oc --kubeconfig ${KUBECONFIG_HOSTED} \
        -n openshift-operators wait --for=condition=Ready pod -l component=webhook-server \
        --timeout=300s
    ```

2. Create the MetalLB CR and configure the Advertizement and pools

    ```bash
    cat <<EOF > metallb-config.yaml
    apiVersion: metallb.io/v1beta1
    kind: MetalLB
    metadata:
      name: metallb
      namespace: openshift-operators
    ---
    apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: lab-network
      namespace: openshift-operators
    spec:
      autoAssign: true
      addresses:
      - 192.168.1.160-192.168.1.165
    ---
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: advertise-lab-network
      namespace: openshift-operators
    spec:
      ipAddressPools:
      - lab-network
    EOF
    
    oc --kubeconfig ${KUBECONFIG_HOSTED} apply -f metallb-config.yaml
    ```

3. Create the LoadBalancer service that exposes the OpenShift Routers.

    ```bash
    cat <<EOF > metallb-svc.yaml
    kind: Service
    apiVersion: v1
    metadata:
      annotations:
        metallb.universe.tf/address-pool: lab-network
        metallb.universe.tf/loadBalancerIPs: 192.168.1.160
      name: metallb-ingress
      namespace: openshift-ingress
    spec:
      ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 80
        - name: https
          protocol: TCP
          port: 443
          targetPort: 443
      selector:
        ingresscontroller.operator.openshift.io/deployment-ingresscontroller: default
      type: LoadBalancer
    EOF
    
    oc --kubeconfig ${KUBECONFIG_HOSTED} apply -f metallb-svc.yaml
    ```

    ```bash
    oc --kubeconfig ${KUBECONFIG_HOSTED} -n openshift-ingress get svc
    ```
    
4. If we check the ClusterVersion again, it should show a finished cluster deployment now.

    ```bash
    oc --kubeconfig ${KUBECONFIG_HOSTED} get clusterversion
    ```

5. Additionally we can check the HostedCluster state on the management cluster.

    ```bash
    oc --kubeconfig ${KUBECONFIG_HUB} -n clusters get hostedcluster hcpc1
    ```
    It can take up to 5 minutes for the hosted cluster to move to completed. 
    
---
### Destroying Lab

Tear down in **reverse dependency order** — skipping a step leaks resources.

```bash
# 1. Delete NodePool first — releases Agents back to inventory
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} delete nodepool ${HCP_NAME}

# 2. Delete HostedCluster — tears down the control-plane namespace
oc --kubeconfig $KUBECONFIG_HUB -n ${HCP_NAMESPACE} delete hostedcluster ${HCP_NAME}

# 3. Wait for the control-plane namespace to disappear
oc --kubeconfig $KUBECONFIG_HUB wait --for=delete namespace/${HCP_NAMESPACE}-${HCP_NAME} --timeout=600s

# 4. Deprovision BMHs (BMO powers down the VMs via Redfish)
oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} delete bmh ${HOSTNAME_NODE0}

# 5. Destroy the VMs
sudo virsh destroy   ${HOSTNAME_NODE0} || true
sudo virsh undefine  ${HOSTNAME_NODE0} --nvram
sudo rm -f /var/lib/libvirt/images/${HOSTNAME_NODE0}.qcow2

# 6. (Optional) Clean inventory artifacts
oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} delete infraenv     ${INFRAENV_NAME}
oc --kubeconfig $KUBECONFIG_HUB -n ${INFRAENV_NAMESPACE} delete nmstateconfig ${HOSTNAME_NODE0}
oc --kubeconfig $KUBECONFIG_HUB delete namespace ${INFRAENV_NAMESPACE}
```
