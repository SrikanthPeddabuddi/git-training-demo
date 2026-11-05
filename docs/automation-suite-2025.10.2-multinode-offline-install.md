# UiPath Automation Suite 2025.10.2 – Multi-Node HA Offline Installation (Linux)

**Topology:** 3 server nodes + 3 agent nodes, air-gapped (offline).
**Audience:** Platform / infrastructure engineers.

> ⚠️ **Verify against the official guide before running anything.** Command flags, bundle file names
> and hardware minimums change between releases. Use this runbook together with the
> *Automation Suite on Linux 2025.10 Installation Guide* (docs.uipath.com). If a flag below is
> rejected, run `./bin/uipathctl <command> --help` to check the exact syntax for your build.

---

## 0. Overview of the flow

| # | Phase | Where it runs |
|---|-------|---------------|
| 1 | Plan: IPs, FQDN, load balancer, DNS, certificates, SQL | – |
| 2 | Prepare OS on all 6 nodes (RHEL, disks, firewall, time, swap) | All nodes |
| 3 | Download bundles on an internet machine and copy them in | Jump host → all nodes |
| 4 | Partition & mount disks (`configureUiPathDisks.sh`) | All nodes |
| 5 | Build `cluster_config.json` | Primary server |
| 6 | Run prerequisite checks | All nodes |
| 7 | Install infrastructure (RKE2) – primary server | Server 1 |
| 8 | Join servers 2 & 3 (one at a time) | Server 2, Server 3 |
| 9 | Join agents 1–3 | Agent 1–3 |
| 10 | Hydrate the in-cluster registry with the offline bundle | Server 1 |
| 11 | Install Fabric + UiPath services (`manifest apply`) | Server 1 |
| 12 | Validate, log in, license, post-install hardening | Server 1 / browser |

---

## 1. Planning

### 1.1 Node inventory (fill in)

| Role | Hostname | IP | vCPU | RAM | Notes |
|------|----------|----|------|-----|-------|
| Server 1 (primary) | as-srv-01 | 10.0.0.11 | ≥16 | ≥32/64 GB | Runs installer, holds bundle disk |
| Server 2 | as-srv-02 | 10.0.0.12 | ≥16 | ≥32/64 GB | |
| Server 3 | as-srv-03 | 10.0.0.13 | ≥16 | ≥32/64 GB | |
| Agent 1 | as-agt-01 | 10.0.0.21 | ≥16 | ≥32/64 GB | Workloads |
| Agent 2 | as-agt-02 | 10.0.0.22 | ≥16 | ≥32/64 GB | |
| Agent 3 | as-agt-03 | 10.0.0.23 | ≥16 | ≥32/64 GB | |

Size the nodes with the **UiPath Installation Sizing Calculator** for the products you enable
(Orchestrator, Test Manager, AI Center, Document Understanding, etc.). An HA multi-node
production profile needs at least 3 server nodes (etcd quorum).

### 1.2 Supported OS

- RHEL 8.x / 9.x (supported minor versions per the 2025.10 compatibility matrix), x86_64.
- Same OS version and kernel on all nodes. Hostnames must be lowercase, unique, resolvable.

### 1.3 Disks (per node)

| Disk / mount | Server nodes | Agent nodes | Purpose |
|--------------|-------------|-------------|---------|
| OS disk (`/`, `/var`, `/tmp`, `/home`) | ✅ | ✅ | ≥ 256 GB recommended |
| Cluster disk → `/var/lib/rancher` | ✅ (≥190 GB) | ✅ (≥190 GB) | RKE2, containerd images |
| etcd disk → `/var/lib/rancher/rke2/server/db` | ✅ (≥16 GB, SSD, low latency) | ❌ | etcd |
| Data disk → `/datadisk` | ✅ (per sizing calc) | optional | Local data |
| Ceph raw disk (unformatted) | ✅ (if in-cluster object store) | optional | In-cluster Ceph object storage |
| **UiPath bundle disk → `/uipath`** | ✅ **Server 1 only** (≥512 GB) | ❌ | Offline bundles + extraction tmp |

> Use the sizing calculator for exact numbers. Raw Ceph disks must have **no partitions or filesystem**.

### 1.4 Load balancer

Place an L4 (TCP) load balancer in front of the **server nodes** (and agents for 443 if you
route ingress to all nodes):

| Port | Protocol | Backend | Purpose |
|------|----------|---------|---------|
| 443 | TCP | all nodes | HTTPS (UiPath UI/API) |
| 6443 | TCP | server nodes | Kubernetes API |
| 9345 | TCP | server nodes | RKE2 node registration |
| 80 | TCP | all nodes (optional) | HTTP → HTTPS redirect |

The LB address (FQDN or IP) becomes **`fixed_rke_address`** in `cluster_config.json`.
Health-check: TCP on 6443/9345, HTTPS on 443.

### 1.5 DNS

Create A/CNAME records pointing to the load balancer:

```
automationsuite.corp.local          -> LB VIP
*.automationsuite.corp.local        -> LB VIP   (wildcard; or the individual sub-domains below)
```

If no wildcard is allowed, create explicit records for at least:
`alm.`, `monitoring.`, `registry.`, `objectstore.`, `insights.`, `apps.`, `sso.` (plus any
product-specific sub-domains listed in the 2025.10 docs) under the FQDN.

### 1.6 Certificates

- Server certificate (PFX or PEM + key) whose SAN covers `automationsuite.corp.local` and
  `*.automationsuite.corp.local`, plus the full CA chain.
- Identity token-signing certificate (optional; self-signed is generated if not provided).
- You can install with self-signed certs and replace later with
  `uipathctl config tls-certificates update`.

### 1.7 External SQL Server

- SQL Server 2016+ (per compatibility matrix), reachable from **all** nodes on 1433.
- A login with `dbcreator` (installer creates databases) or pre-created databases per product.
- Collation `SQL_Latin1_General_CP1_CI_AS`; enable Full-Text Search if Document Understanding / Test Manager need it.
- Record the connection strings (ADO.NET, ODBC, JDBC) – the config file needs them.

### 1.8 Network ports between nodes (open in firewall / security groups)

| Port | Proto | Between | Purpose |
|------|-------|---------|---------|
| 22 | TCP | admin → nodes | SSH |
| 80, 443 | TCP | clients/LB → nodes | Ingress |
| 2379-2380 | TCP | server ↔ server | etcd |
| 6443 | TCP | all → servers/LB | Kubernetes API |
| 9345 | TCP | all → servers/LB | RKE2 supervisor |
| 10250 | TCP | all ↔ all | kubelet |
| 8472 | UDP | all ↔ all | Canal/Flannel VXLAN |
| 9099 | TCP | all ↔ all | Canal health |
| 9100 | TCP | all ↔ all | Node exporter |
| 30000-32767 | TCP | all ↔ all | NodePort range |
| 7946 | TCP/UDP | all ↔ all | (if required by your release) |
| 1433 | TCP | nodes → SQL | SQL Server |

> Easiest approach for an isolated subnet: allow **all traffic between cluster nodes**, and
> restrict only inbound from outside.

---

## 2. OS preparation (run on **all 6 nodes** as root)

```bash
# 2.1 Hostname (example for server 1)
hostnamectl set-hostname as-srv-01

# 2.2 /etc/hosts or DNS – every node must resolve every other node
cat >> /etc/hosts <<'EOF'
10.0.0.11 as-srv-01
10.0.0.12 as-srv-02
10.0.0.13 as-srv-03
10.0.0.21 as-agt-01
10.0.0.22 as-agt-02
10.0.0.23 as-agt-03
EOF

# 2.3 Time sync (local NTP server, since offline)
dnf install -y chrony      # from local repo / ISO
sed -i 's/^pool .*/server ntp.corp.local iburst/' /etc/chrony.conf
systemctl enable --now chronyd && chronyc tracking

# 2.4 Swap off
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab

# 2.5 Firewall: either open the ports from §1.8 or disable firewalld (UiPath recommends disabled)
systemctl disable --now firewalld

# 2.6 nm-cloud-setup must be disabled (RKE2 requirement), and NetworkManager must ignore CNI interfaces
systemctl disable --now nm-cloud-setup.service nm-cloud-setup.timer 2>/dev/null || true
cat > /etc/NetworkManager/conf.d/rke2-canal.conf <<'EOF'
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:flannel*
EOF
systemctl reload NetworkManager

# 2.7 SELinux: enforcing is supported; leave as configured by your security baseline.

# 2.8 Required base packages (from your local RHEL repo/ISO, since the node is offline)
dnf install -y tar unzip curl jq lvm2 iptables nfs-utils

# 2.9 Proxy: none (offline). Make sure no stale http_proxy/https_proxy env vars exist.
env | grep -i proxy
```

Also ensure:
- `umask 0022` for root (installer files must be world-readable).
- No antivirus/EDR blocking `/var/lib/rancher`, `/var/lib/kubelet`, `/opt/UiPathAutomationSuite`
  (add exclusions if you run one).
- All nodes are in the **same subnet/region** with low latency (< 10 ms between servers for etcd).

---

## 3. Download the offline artifacts (on an internet-connected machine)

From the Automation Suite 2025.10.2 download page, download:

| File | Purpose | Copy to |
|------|---------|---------|
| `as-installer.zip` | Installer + `uipathctl` + `configureUiPathDisks.sh` + `installUiPathAS.sh` | **All 6 nodes** |
| `sf-infra.tar.gz` | Offline infra bundle (RKE2, RPMs, infra images) | **All 6 nodes** (or shared path) |
| `sf.tar.gz` | Main offline bundle (UiPath product images, Helm charts) | **Server 1 only** (`/uipath`) |
| Optional bundles (e.g. `du-ondemand.tar.gz`, `cv-ondemand.tar.gz`, ML packages) | Only if those products are enabled | Server 1 (`/uipath`) |

> File names may differ slightly in 2025.10.2 – follow the download page. Verify checksums
> (`sha256sum`) after copying.

Copy via your approved transfer method (SCP from a jump host, USB, artifact repository):

```bash
# example from jump host
for h in as-srv-01 as-srv-02 as-srv-03 as-agt-01 as-agt-02 as-agt-03; do
  scp as-installer.zip sf-infra.tar.gz root@$h:/tmp/
done
scp sf.tar.gz root@as-srv-01:/tmp/        # move into /uipath after step 4
```

---

## 4. Extract installer and configure disks (all nodes)

### 4.1 Extract installer

```bash
mkdir -p /opt/UiPathAutomationSuite/2025.10.2/installer
cd /opt/UiPathAutomationSuite/2025.10.2
unzip /tmp/as-installer.zip -d installer
chmod -R 755 installer
cd installer
```

### 4.2 Identify disks

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
```

### 4.3 Run `configureUiPathDisks.sh`

**Server 1 (primary – includes bundle disk):**

```bash
./configureUiPathDisks.sh --node-type server --install-type offline \
  --cluster-disk-name /dev/sdb \
  --etcd-disk-name    /dev/sdc \
  --data-disk-name    /dev/sdd \
  --bundle-disk-name  /dev/sde
```

**Server 2 & 3:**

```bash
./configureUiPathDisks.sh --node-type server --install-type offline \
  --cluster-disk-name /dev/sdb \
  --etcd-disk-name    /dev/sdc \
  --data-disk-name    /dev/sdd
```

**Agent 1–3:**

```bash
./configureUiPathDisks.sh --node-type agent --install-type offline \
  --cluster-disk-name /dev/sdb
```

**In-cluster Ceph object store (servers, if using in-cluster storage):**

```bash
./configureUiPathDisks.sh --ceph-raw-disk-name /dev/sdf
```

Verify:

```bash
df -h /var/lib/rancher /var/lib/rancher/rke2/server/db /datadisk /uipath 2>/dev/null
lsblk
```

### 4.4 Move bundles to the bundle disk (Server 1)

```bash
mv /tmp/sf.tar.gz /tmp/sf-infra.tar.gz /uipath/
mkdir -p /uipath/tmp
```

On the other nodes keep `sf-infra.tar.gz` somewhere with enough space (e.g. `/opt/UiPathAutomationSuite/`)
or mount the same path via NFS read-only.

---

## 5. Create `cluster_config.json` (Server 1)

### Option A – Interactive wizard (recommended)

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer
./installUiPathAS.sh
```

Choose: **Multi-node HA-ready production** → **Offline** → enter FQDN, LB address, SQL details,
certificates, product selection. **Stop the wizard after the config file is generated**
(the wizard prints where `cluster_config.json` was written) and continue manually, because
in multi-node you must install node by node.

### Option B – Manual file (key fields)

```jsonc
{
  "fqdn": "automationsuite.corp.local",
  "fixed_rke_address": "automationsuite-lb.corp.local",   // LB in front of servers
  "admin_username": "admin",
  "admin_password": "<StrongPassword>",
  "rke_token": "<random GUID – identical on all nodes>",
  "profile": "ha",
  "install_type": "offline",
  "telemetry_optout": false,
  "accept_license_agreement": true,

  "server_certificate": {
    "ca_cert_file": "/opt/certs/rootCA.crt",
    "tls_cert_file": "/opt/certs/server.crt",
    "tls_key_file": "/opt/certs/server.key"
  },

  "sql": {
    "server_url": "sql.corp.local",
    "port": "1433",
    "username": "uipathas",
    "password": "<SqlPassword>",
    "create_db": true
  },
  "sql_connection_string_template":        "Server=tcp:sql.corp.local,1433;Initial Catalog=DB_NAME_PLACEHOLDER;Persist Security Info=False;User Id=uipathas;Password='<pwd>';MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=True;Connection Timeout=30;Max Pool Size=100;",
  "sql_connection_string_template_jdbc":   "jdbc:sqlserver://sql.corp.local:1433;database=DB_NAME_PLACEHOLDER;user=uipathas;password={<pwd>};encrypt=true;trustServerCertificate=true;loginTimeout=30;hostNameInCertificate=sql.corp.local",
  "sql_connection_string_template_odbc":   "SERVER=sql.corp.local,1433;DATABASE=DB_NAME_PLACEHOLDER;DRIVER={ODBC Driver 17 for SQL Server};UID=uipathas;PWD={<pwd>};MultipleActiveResultSets=False;Encrypt=YES;TrustServerCertificate=YES;Connection Timeout=30;",
  "sql_connection_string_template_sqlalchemy_pyodbc": "mssql+pyodbc://uipathas:<pwd>@sql.corp.local:1433/DB_NAME_PLACEHOLDER?driver=ODBC+Driver+17+for+SQL+Server",

  "infra": {
    "docker_registry": { "username": "admin", "password": "<RegistryPassword>" },
    "custom_dns_resolver": ""
  },

  "external_object_storage": { "enabled": false },   // false = in-cluster Ceph

  "orchestrator":     { "enabled": true },
  "action_center":    { "enabled": true },
  "insights":         { "enabled": false },
  "test_manager":     { "enabled": false },
  "aicenter":         { "enabled": false },
  "documentunderstanding": { "enabled": false }
  // … other products as required
}
```

> Generate `rke_token` with `uuidgen`. Validate JSON with `jq . cluster_config.json`.
> The exact schema for 2025.10 is in the docs ("Manual: Advanced installation experience →
> cluster_config.json"); fields not listed there will fail validation.

### 5.1 Copy the same config file to every node

```bash
for h in as-srv-02 as-srv-03 as-agt-01 as-agt-02 as-agt-03; do
  scp cluster_config.json root@$h:/opt/UiPathAutomationSuite/2025.10.2/installer/
done
```

The file **must be identical** on all nodes (same `rke_token`, `fixed_rke_address`, FQDN).

---

## 6. Prerequisite checks (all nodes)

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer
./bin/uipathctl prereq create --help        # confirm flags for your build
./bin/uipathctl prereq run cluster_config.json --versions versions/helm-charts.json
```

Fix every `FAIL` (disk sizes, ports, DNS, SQL reachability, time sync, kernel params) before
continuing. Warnings should be reviewed.

---

## 7. Install infrastructure on **Server 1** (primary)

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer

./bin/uipathctl rke2 install \
  -i ./cluster_config.json \
  -o ./output.json \
  --offline-bundle /uipath/sf-infra.tar.gz \
  --offline-tmp-folder /uipath/tmp \
  --install-type offline \
  --accept-license-agreement
```

Verify the first server is up:

```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin
kubectl get nodes -o wide        # as-srv-01  Ready  control-plane,etcd,master
kubectl get pods -A | grep -v Running | grep -v Completed
```

---

## 8. Join **Server 2**, then **Server 3** (sequentially!)

On **as-srv-02** (wait until it's `Ready`, then repeat on **as-srv-03**):

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer

./bin/uipathctl rke2 install \
  -i ./cluster_config.json \
  -o ./output.json \
  -j server \
  --offline-bundle /path/to/sf-infra.tar.gz \
  --offline-tmp-folder /path/to/tmp \
  --install-type offline \
  --accept-license-agreement
```

Check from Server 1:

```bash
kubectl get nodes          # 3 servers Ready
kubectl -n kube-system get pods -l component=etcd
```

> ⚠️ Never join two server nodes in parallel – etcd membership changes must be serialized.

---

## 9. Join **Agent 1, 2, 3** (can run in parallel)

On each agent:

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer

./bin/uipathctl rke2 install \
  -i ./cluster_config.json \
  -o ./output.json \
  -j agent \
  --offline-bundle /path/to/sf-infra.tar.gz \
  --offline-tmp-folder /path/to/tmp \
  --install-type offline \
  --accept-license-agreement
```

Check from Server 1:

```bash
kubectl get nodes -o wide
# NAME        STATUS  ROLES                       
# as-srv-01   Ready   control-plane,etcd,master
# as-srv-02   Ready   control-plane,etcd,master
# as-srv-03   Ready   control-plane,etcd,master
# as-agt-01   Ready   <none>
# as-agt-02   Ready   <none>
# as-agt-03   Ready   <none>
```

Optional (only if you plan specialized agents, e.g. GPU for AI Center or Task Mining):
label/taint them according to the docs before the next step.

---

## 10. Hydrate the in-cluster registry (Server 1)

Uploads all product images/charts from `sf.tar.gz` into the in-cluster Docker registry:

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer

./bin/uipathctl rke2 registry upload \
  --offline-bundle /uipath/sf.tar.gz \
  --offline-tmp-folder /uipath/tmp
```

Repeat for any optional bundles (DU on-demand, CV, etc.) you downloaded. This step can take
1–2+ hours depending on disk speed.

---

## 11. Install Fabric + UiPath services (Server 1)

```bash
cd /opt/UiPathAutomationSuite/2025.10.2/installer

# Re-run prereqs against the live cluster
./bin/uipathctl prereq run cluster_config.json --versions versions/helm-charts.json

# Deploy shared components and all enabled products
./bin/uipathctl manifest apply cluster_config.json --versions versions/helm-charts.json
```

Monitor progress (separate shell):

```bash
watch -n 20 'kubectl get pods -A | grep -Ev "Running|Completed"'
kubectl -n argocd get applications
```

The command exits successfully once all ArgoCD applications are `Synced` / `Healthy`.

---

## 12. Validation and post-installation

### 12.1 Health check

```bash
./bin/uipathctl health check
./bin/uipathctl health bundle     # generates a support bundle if anything fails
```

### 12.2 Access

| Portal | URL | Credentials |
|--------|-----|-------------|
| Automation Suite (host org) | `https://automationsuite.corp.local/` → *Host* | `admin` / `admin_password` from config |
| Default org | `https://automationsuite.corp.local/` | `orgadmin` / same password (change at first login) |
| ArgoCD (read-only) | `https://alm.automationsuite.corp.local` | `uipathctl config argocd ...` (see docs) |
| Monitoring (Grafana/Prometheus) | `https://monitoring.automationsuite.corp.local` | see docs |

### 12.3 Post-install tasks

1. **Activate license** (offline activation in Host portal → Licenses).
2. **Replace self-signed certs** (if used):
   `./bin/uipathctl config tls-certificates update --tls-cert server.crt --tls-key server.key --ca-cert ca.crt`
3. **Configure backup** (etcd snapshots + NFS backup target) per "Backing up and restoring the cluster".
4. **Configure SMTP**, identity providers (AD/Azure AD/SAML), alerting receivers.
5. **Keep the `/uipath` bundle disk** until you've validated – it's also needed for adding nodes later.
6. Store `cluster_config.json` **securely** (contains passwords) and back it up off-cluster.
7. Connect Robots/Studio to `https://automationsuite.corp.local/<org>/<tenant>/orchestrator_`.

---

## 13. Troubleshooting quick reference

| Symptom | Check |
|---------|-------|
| Node stuck joining | `journalctl -u rke2-server -f` (servers) / `journalctl -u rke2-agent -f` (agents); LB 9345/6443 reachable; same `rke_token` |
| etcd unhealthy | Disk latency on etcd disk (`fio`), clock skew (`chronyc tracking`) |
| Pods `ImagePullBackOff` | Registry hydration incomplete → rerun step 10 |
| ArgoCD app `Degraded` | `kubectl -n argocd get app <name> -o yaml`, then pod logs in its namespace |
| SQL errors | Connection strings, firewall 1433 from **every** node, `dbcreator` role |
| DNS failures in pods | `custom_dns_resolver`, `/etc/resolv.conf` has no unreachable nameservers / `search` loops |
| Generic | `./bin/uipathctl health check`, `./bin/uipathctl health bundle` → open UiPath support ticket |

---

## 14. Execution checklist

- [ ] 6 VMs provisioned, sized via calculator, RHEL version supported
- [ ] LB configured (443, 6443, 9345, 80) – health checks green
- [ ] DNS: FQDN + wildcard → LB
- [ ] Certificates ready (SAN includes FQDN + wildcard)
- [ ] SQL Server reachable from all nodes, login has `dbcreator`
- [ ] NTP, swap off, firewall, nm-cloud-setup disabled on all nodes
- [ ] `as-installer.zip`, `sf-infra.tar.gz` on all nodes; `sf.tar.gz` on Server 1 `/uipath`
- [ ] `configureUiPathDisks.sh` run on all nodes (bundle disk on Server 1)
- [ ] Identical `cluster_config.json` on all nodes
- [ ] `uipathctl prereq run` passes
- [ ] RKE2 on Server 1 → Server 2 → Server 3 (sequential)
- [ ] RKE2 on Agents 1–3
- [ ] `uipathctl rke2 registry upload` completed
- [ ] `uipathctl manifest apply` completed, all ArgoCD apps Healthy
- [ ] `uipathctl health check` green, portal login works, license activated
- [ ] Backup configured, config file archived securely
