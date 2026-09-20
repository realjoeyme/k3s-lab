# Three-Node K3s Homelab

This repository documents my hands-on Kubernetes learning environment: a
highly available, three-node K3s cluster running embedded etcd, kube-vip,
Traefik, Headlamp and dynamically provisioned Synology iSCSI storage.

The tracked manifests are the declarative record for cluster infrastructure
and applications. Cluster administration is performed from openclaw.lan
rather than from a K3s node. Credentials and machine-specific secret material
are deliberately excluded.

## Architecture

~~~mermaid
flowchart TD
    GH[Public GitHub repository] <-->|git pull / push| MGMT[openclaw.lan<br/>kubectl + private kubeconfig]
    MGMT -->|Kubernetes API| API[API VIP<br/>api.k3s.joeyme.eu<br/>192.168.100.200:6443]

    API --> N1[k3s-01<br/>192.168.100.201]
    API --> N2[k3s-02<br/>192.168.100.202]
    API --> N3[k3s-03<br/>192.168.100.203]

    CLIENT[Internal client] -->|headlamp.k3s.joeyme.eu| LB[Traefik LoadBalancer VIP<br/>192.168.100.204]
    LB --> HEADLAMP[Headlamp Service and Pod]

    N1 -->|Synology CSI<br/>iSCSI 3260| NAS[Synology DS423<br/>192.168.100.61<br/>volume1]
    N2 -->|Synology CSI<br/>iSCSI 3260| NAS
    N3 -->|Synology CSI<br/>iSCSI 3260| NAS
    MGMT -->|DSM API HTTPS 5001| NAS
~~~

All three K3s nodes run:

- The K3s control plane
- Embedded etcd
- Schedulable workloads
- A kube-vip DaemonSet instance
- A Synology CSI node plugin

| Component | Purpose | Version in manifests |
| --- | --- | --- |
| K3s | Kubernetes distribution | v1.36.4+k3s1 (tested) |
| kube-vip | Control-plane and service virtual IPs | v1.2.3 |
| kube-vip cloud provider | Allocates LoadBalancer addresses | v0.0.12 |
| Traefik | Ingress controller | Managed by K3s |
| Headlamp | Kubernetes dashboard | v0.45.0 |
| Synology CSI | Dynamic iSCSI storage | v1.3.1 |

The Synology CSI integration was validated against K3s v1.36.4+k3s1 and DSM
7.4.1 on a DS423.

## Repository layout

~~~text
.
├── apps
│   └── headlamp
│       ├── headlamp.yaml
│       └── ingress.yaml
└── infrastructure
    ├── kube-vip
    │   ├── address-pool.yaml
    │   ├── cloud-controller.yaml
    │   ├── daemonset.yaml
    │   ├── k3s-config.yaml.example
    │   └── rbac.yaml
    └── synology-csi
        ├── README.md
        ├── client-info.yml.example
        ├── storageclass.yaml
        ├── tests
        │   ├── pod-k3s-01.yaml
        │   ├── pod-k3s-02.yaml
        │   └── pvc.yaml
        └── upstream-v1.3.1
            ├── LICENSE
            ├── controller.yaml
            ├── csi-driver.yaml
            ├── namespace.yaml
            └── node.yaml
~~~

## Design decisions

### Highly available control plane

The Kubernetes API is reached through api.k3s.joeyme.eu at
192.168.100.200, not through an individual node. kube-vip advertises this
address and moves it between control-plane nodes when leadership changes.
Both the DNS name and VIP are present in the K3s API certificate.

| DNS record | Address |
| --- | --- |
| api.k3s.joeyme.eu | 192.168.100.200 |
| k3s-01.k3s.joeyme.eu | 192.168.100.201 |
| k3s-02.k3s.joeyme.eu | 192.168.100.202 |
| k3s-03.k3s.joeyme.eu | 192.168.100.203 |

### LoadBalancer services

The built-in K3s ServiceLB is disabled. kube-vip allocates addresses for
Kubernetes Services of type LoadBalancer; the current pool contains
192.168.100.204, which is assigned to Traefik.

### Shared storage

The official Synology CSI driver provisions LUNs, iSCSI targets and mappings
on /volume1. The custom synology-iscsi-retain StorageClass:

- Is not the default StorageClass
- Uses ext4 and ReadWriteOnce
- Supports online expansion
- Uses Retain to protect application data from accidental PVC deletion

The K3s local-path StorageClass remains the default for disposable and
node-local data. Workloads must explicitly opt in to Synology storage.

### Encrypted Kubernetes Secrets

K3s Secrets encryption is enabled on every server. Existing Secrets were
re-encrypted, the encryption key was rotated, and all server encryption hashes
were verified to match. A disposable marker Secret was readable through the
Kubernetes API but absent as plaintext from raw etcd storage.

The encryption configuration and key material are never stored in this
repository.

### Separate management host

Git, kubectl and the administrative kubeconfig live on openclaw.lan. The K3s
nodes contain only the runtime and node-local configuration. This keeps
cluster nodes focused on running the cluster and provides one clear Git
workflow.

## Deployment

> These manifests describe this lab. Review and adapt addresses, interface
> names, hostnames, certificates, RBAC and versions before using them
> elsewhere.

On openclaw.lan, select the private administrative kubeconfig explicitly:

~~~bash
export KUBECONFIG="$HOME/.kube/k3s-lab.yaml"
kubectl cluster-info
~~~

### 1. Configure every K3s server

infrastructure/kube-vip/k3s-config.yaml.example represents
/etc/rancher/k3s/config.yaml:

~~~yaml
tls-san:
  - 192.168.100.200
  - api.k3s.joeyme.eu

disable:
  - servicelb

secrets-encryption: true
~~~

This is an example rather than an automatically applied manifest because it
configures the K3s process itself. Changes are rolled through one server at a
time while preserving etcd quorum.

### 2. Deploy kube-vip

~~~bash
kubectl apply -f infrastructure/kube-vip/rbac.yaml
kubectl apply -f infrastructure/kube-vip/cloud-controller.yaml
kubectl apply -f infrastructure/kube-vip/daemonset.yaml
kubectl apply -f infrastructure/kube-vip/address-pool.yaml
~~~

### 3. Deploy Synology CSI

Follow
[infrastructure/synology-csi/README.md](infrastructure/synology-csi/README.md).
It documents node and DSM prerequisites, TLS verification, private Secret
creation, install order, the complete storage test and cleanup.

The real client-info.yml and client-info-secret are intentionally absent.

### 4. Deploy Headlamp

~~~bash
kubectl apply -f apps/headlamp/headlamp.yaml
kubectl apply -f apps/headlamp/ingress.yaml
~~~

Headlamp is available internally at:

~~~text
https://headlamp.k3s.joeyme.eu
~~~

## Validation

Useful cluster checks:

~~~bash
kubectl get nodes
kubectl -n kube-system get daemonset kube-vip-ds
kubectl -n kube-system get service traefik
kubectl -n headlamp get deployment,pod,service,ingress -o wide
kubectl -n synology-csi get pods -o wide
kubectl get csidriver csi.san.synology.com
kubectl get storageclass
kubectl get pvc -A
kubectl get pv
kubectl get volumeattachment
~~~

Before applying a manifest:

~~~bash
kubectl apply --dry-run=server -f path/to/manifest.yaml
kubectl diff -f path/to/manifest.yaml
~~~

## Completed resilience tests

### Workload and ingress

I cordoned k3s-01 and k3s-03, deleted the running Headlamp pod and watched
Kubernetes recreate it on k3s-02. Headlamp remained reachable through
Traefik's 192.168.100.204 LoadBalancer address. Both nodes were uncordoned
after the test.

This demonstrated the separation between a stable service endpoint and a
replaceable workload pod.

### Synology storage

A disposable iSCSI PVC was used to prove:

1. Dynamic LUN, target, mapping and PV creation
2. ext4 mount and checksum-protected writes
3. Data persistence across pod deletion and recreation
4. Clean detach from k3s-01 and reattach to k3s-02
5. Online expansion from 1 GiB to 2 GiB
6. Data integrity after expansion
7. Retain behavior after PVC deletion
8. Controlled backend cleanup through the CSI provisioner

Kubernetes attachment state, node iSCSI sessions and DSM objects agreed at
each stage. The disposable PV, LUN, target and test namespace were removed
after validation.

## Security notes

- This repository is public. Kubeconfigs, passwords, private keys, environment
  files, populated client-info files and Secret manifests must never be
  committed.
- .gitignore reduces the chance of adding new secret files, but it does not
  remove an already tracked secret or erase Git history.
- The dedicated DSM CSI account requires DSM administrative API capability to
  manage storage. Its unrelated application and shared-folder access is
  denied.
- The Headlamp administrative service account is intentionally bound to
  cluster-admin for this personal, access-restricted learning lab. Its token
  is generated only when needed and is not stored in Git.
- Private RFC1918 addresses document the lab topology and are not
  Internet-routable.
- Retained volumes and Synology snapshots are not independent backups.

## Lessons learned

- A Kubernetes namespace deletion does not remove cluster-scoped resources
  such as a ClusterRoleBinding.
- A LoadBalancer VIP and Service status are related but separate states.
- Cordoning a node prevents new scheduling but does not move existing pods.
- Cloned nodes can share an iSCSI initiator IQN; every node must have a unique
  identity before LUNs are mapped.
- A reachable DSM API does not by itself prove that TCP 3260 is listening.
- A ReadWriteOnce volume can move between nodes after a clean detach.
- Retain deliberately separates PVC deletion from backend data deletion.
- Kubernetes Secret values are base64-encoded by the API; K3s Secrets
  encryption provides the separate at-rest protection in etcd.
- Git history is useful only when credentials remain outside the repository
  from the beginning.

This is a learning repository and a reproducible record of the cluster. It
does not justify keeping the cluster by itself; future operation depends on
whether a genuinely useful workload is selected.
