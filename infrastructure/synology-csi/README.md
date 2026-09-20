# Synology CSI storage

This directory reproduces the tested Synology iSCSI storage integration for
the three-node K3s cluster. The CSI driver dynamically creates a Synology LUN,
iSCSI target and mapping for each PersistentVolumeClaim (PVC).

The deployment uses an ext4 filesystem with ReadWriteOnce access. One node
mounts a volume at a time; Kubernetes can detach it and reattach it to another
healthy node.

## Tested environment

| Component | Tested value |
| --- | --- |
| K3s | v1.36.4+k3s1 |
| K3s nodes | k3s-01, k3s-02, k3s-03 |
| Node addresses | 192.168.100.201-192.168.100.203 |
| Synology | DS423, DSM 7.4.1 |
| DSM address | 192.168.100.61 |
| DSM HTTPS port | 5001 |
| DSM volume | /volume1 |
| CSI driver | synology/synology-csi:v1.3.1 |
| CSI provisioner | registry.k8s.io/sig-storage/csi-provisioner:v3.0.0 |
| CSI attacher | registry.k8s.io/sig-storage/csi-attacher:v3.3.0 |
| CSI resizer | registry.k8s.io/sig-storage/csi-resizer:v1.3.0 |
| CSI node registrar | registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.3.0 |
| Protocol/filesystem | iSCSI / ext4 |

The vendored files under upstream-v1.3.1/ are the unmodified Kubernetes
v1.20 manifests from Synology CSI tag
[v1.3.1](https://github.com/SynologyOpenSource/synology-csi/tree/v1.3.1),
commit d6e51755fb09dc8e6979aa4f7e14649158c1b6a3. The upstream Apache-2.0
license is included beside them. Snapshot-controller components are not
installed.

## Repository contents

~~~text
synology-csi/
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

## Security model

- A dedicated DSM account named k3s-iscsi performs LUN and target lifecycle
  operations through the DSM API.
- The account must belong to DSM's administrators group and be allowed to use
  DSM. Shared-folder permissions and unrelated DSM applications are denied.
- DSM credentials live only in the synology-csi/client-info-secret Kubernetes
  Secret.
- K3s Secrets encryption is enabled before the DSM credential is stored, so
  Secret payloads are encrypted before being written to embedded etcd.
- DSM HTTPS is verified with the Synology CA certificate and
  tlsServerName: synology. TLS verification is not disabled.
- The populated client-info.yml, certificate files, private keys, kubeconfigs
  and K3s encryption configuration must never be committed.

The CA certificate is public material, but it is environment-specific and is
supplied locally. The DSM CA private key and leaf private key must never leave
protected storage.

## 1. K3s server configuration

Every server uses /etc/rancher/k3s/config.yaml. The tracked example is:

~~~yaml
tls-san:
  - 192.168.100.200
  - api.k3s.joeyme.eu

disable:
  - servicelb

secrets-encryption: true
~~~

The API certificate must contain both the VIP and DNS name:

~~~bash
openssl s_client \
  -connect api.k3s.joeyme.eu:6443 \
  -servername api.k3s.joeyme.eu </dev/null 2>/dev/null |
openssl x509 -noout -ext subjectAltName
~~~

### Enabling Secrets encryption on an existing cluster

Take an on-demand etcd snapshot before changing encryption:

~~~bash
sudo k3s etcd-snapshot save \
  --name pre-secrets-encryption-$(date +%Y%m%d-%H%M%S)
~~~

Enable encryption once from one server:

~~~bash
sudo k3s secrets-encrypt enable
~~~

Merge secrets-encryption: true into the existing config on all three servers.
Restart one server at a time, waiting for it to become Ready before continuing
to the next, so etcd quorum remains available.

After all servers load the initial encryption configuration:

~~~bash
sudo k3s secrets-encrypt status
sudo k3s secrets-encrypt rotate-keys
~~~

Perform another rolling restart and require:

~~~text
Encryption Status: Enabled
Current Rotation Stage: reencrypt_finished
Server Encryption Hashes: All hashes match
~~~

Never copy or display
/var/lib/rancher/k3s/server/cred/encryption-config.json; it contains the
encryption keys.

The remaining kubectl commands are run from openclaw.lan with the private
administrative kubeconfig selected explicitly:

~~~bash
export KUBECONFIG="$HOME/.kube/k3s-lab.yaml"
kubectl cluster-info
~~~

## 2. Node iSCSI prerequisites

On every K3s node:

1. Install open-iscsi.
2. Confirm iscsid.socket is active.
3. Confirm /etc/iscsi/initiatorname.iscsi contains a unique IQN.
4. Confirm the DSM HTTPS endpoint is reachable.
5. Confirm there are no unexpected active sessions.

Example checks:

~~~bash
dpkg-query -W open-iscsi
systemctl is-active iscsid.socket
cat /etc/iscsi/initiatorname.iscsi
sudo iscsiadm -m session
timeout 3 bash -c '</dev/tcp/192.168.100.61/5001'
~~~

Cloned machines can inherit the same initiator IQN. Correct duplicate IQNs
before presenting any LUN. Do not copy a tracked initiator file between nodes.

On the tested DS423, TCP 3260 began accepting connections after CSI created
the first target. After provisioning a PVC, verify it from every node:

~~~bash
timeout 3 bash -c '</dev/tcp/192.168.100.61/3260'
~~~

## 3. DSM preparation

1. Install and open SAN Manager.
2. Create the dedicated k3s-iscsi account.
3. Add it to administrators and allow DSM access.
4. Deny shared-folder access and unrelated applications.
5. Do not manually create a LUN or target for dynamic provisioning.
6. Export only the DSM leaf certificate and CA certificate needed to verify
   HTTPS. Do not export or distribute private keys.

Verify the certificate chain locally:

~~~bash
openssl verify -CAfile syno-ca-cert.pem cert.pem
openssl x509 -in syno-ca-cert.pem -noout -subject -issuer -text |
grep -E 'subject=|issuer=|CA:TRUE'
~~~

The tested DSM leaf certificate is issued to synology, which is why the
client configuration uses tlsServerName: synology while connecting to the NAS
by IP address.

## 4. Create the private client configuration

The example file is intentionally unusable until its placeholders are
replaced. Copy it to a private, Git-ignored location:

~~~bash
install -d -m 700 "$HOME/.config/k3s"
install -m 600 \
  infrastructure/synology-csi/client-info.yml.example \
  "$HOME/.config/k3s/client-info.yml"
~~~

Edit the private copy and replace:

- REPLACE_LOCALLY with the DSM account password.
- The certificate placeholder with the complete PEM-encoded DSM CA
  certificate, including the BEGIN/END lines.

Do not use insecureSkipVerify.

## 5. Install the driver

Create the namespace first:

~~~bash
kubectl apply -f \
  infrastructure/synology-csi/upstream-v1.3.1/namespace.yaml
~~~

Create the Secret from the private file. The password never appears in a
command argument or tracked manifest:

~~~bash
kubectl -n synology-csi create secret generic client-info-secret \
  --from-file=client-info.yml="$HOME/.config/k3s/client-info.yml"
~~~

Install the minimum components required for dynamic provisioning:

~~~bash
kubectl apply -f \
  infrastructure/synology-csi/upstream-v1.3.1/csi-driver.yaml
kubectl apply -f \
  infrastructure/synology-csi/upstream-v1.3.1/controller.yaml
kubectl apply -f \
  infrastructure/synology-csi/upstream-v1.3.1/node.yaml
kubectl apply -f \
  infrastructure/synology-csi/storageclass.yaml
~~~

local-path remains the default StorageClass. Workloads must explicitly
request synology-iscsi-retain.

## 6. Validate the installation

~~~bash
kubectl -n synology-csi get pods -o wide
kubectl get csidriver csi.san.synology.com
kubectl get storageclass
kubectl -n synology-csi get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .spec.containers[*]}{"  "}{.name}{" = "}{.image}{"\n"}{end}{end}'
~~~

Expected state:

- Controller StatefulSet: 4/4 Running
- One node pod on each K3s node: 2/2 Running
- CSI driver: csi.san.synology.com
- Plugin image everywhere: synology/synology-csi:v1.3.1
- synology-iscsi-retain is non-default, expandable and uses Retain

## 7. End-to-end storage test

The test intentionally creates and later destroys a disposable Synology LUN.

Create the namespace and 1 GiB claim:

~~~bash
kubectl apply -f infrastructure/synology-csi/tests/pvc.yaml
kubectl wait \
  --namespace synology-csi-test \
  --for=jsonpath='{.status.phase}'=Bound \
  pvc/iscsi-smoke-test \
  --timeout=180s

PV=$(kubectl -n synology-csi-test get pvc iscsi-smoke-test \
  -o jsonpath='{.spec.volumeName}')
printf '%s\n' "$PV"
~~~

Confirm the matching LUN and target appear in DSM.

### Write and verify data on k3s-01

~~~bash
kubectl apply -f infrastructure/synology-csi/tests/pod-k3s-01.yaml
kubectl -n synology-csi-test wait \
  --for=condition=Ready pod/iscsi-smoke --timeout=180s
kubectl -n synology-csi-test logs iscsi-smoke
kubectl -n synology-csi-test exec iscsi-smoke -- \
  sh -c 'cd /data && sha256sum -c proof.sha256'
~~~

Delete and recreate the same pod once to prove that pod replacement does not
remove the data:

~~~bash
kubectl -n synology-csi-test delete pod iscsi-smoke --wait=true
kubectl apply -f infrastructure/synology-csi/tests/pod-k3s-01.yaml
kubectl -n synology-csi-test wait \
  --for=condition=Ready pod/iscsi-smoke --timeout=180s
kubectl -n synology-csi-test exec iscsi-smoke -- \
  sh -c 'cd /data && sha256sum -c proof.sha256'
~~~

### Move the volume to k3s-02

~~~bash
kubectl -n synology-csi-test delete pod iscsi-smoke --wait=true
kubectl apply -f infrastructure/synology-csi/tests/pod-k3s-02.yaml
kubectl -n synology-csi-test wait \
  --for=condition=Ready pod/iscsi-smoke --timeout=180s
kubectl -n synology-csi-test get pod iscsi-smoke -o wide
kubectl -n synology-csi-test exec iscsi-smoke -- \
  sh -c 'cd /data && sha256sum -c proof.sha256'
kubectl get volumeattachment -o wide
~~~

The session must be absent from k3s-01 and logged in on k3s-02. Run this
node-local command on both nodes:

~~~bash
sudo iscsiadm -m session
~~~

### Expand the mounted filesystem

~~~bash
kubectl -n synology-csi-test patch pvc iscsi-smoke-test \
  --type=merge \
  -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
kubectl -n synology-csi-test get pvc iscsi-smoke-test -w
~~~

After capacity reports 2Gi:

~~~bash
kubectl -n synology-csi-test exec iscsi-smoke -- df -h /data
kubectl -n synology-csi-test exec iscsi-smoke -- \
  sh -c 'cd /data && sha256sum -c proof.sha256'
~~~

The filesystem should report approximately 1.9-2.0 GiB, DSM should show a
2 GiB LUN, and the checksum must remain valid.

### Prove Retain and clean up

Keep the previously captured PV value. Delete the consumer and claim:

~~~bash
kubectl -n synology-csi-test delete pod iscsi-smoke --wait=true
kubectl -n synology-csi-test delete pvc iscsi-smoke-test
kubectl get pv "$PV" -o wide
~~~

The PV must enter Released with reclaim policy Retain; the LUN and target must
still exist in DSM, and no node should have an active session.

After identifying the exact disposable backend object, change only this test
PV to Delete. This permanently removes its test data:

~~~bash
kubectl patch pv "$PV" \
  --type=merge \
  -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
kubectl wait --for=delete "pv/$PV" --timeout=180s
kubectl delete namespace synology-csi-test
~~~

Confirm that the matching LUN and target disappear from DSM. Do not issue a
separate kubectl delete pv; allow the external provisioner to complete the
backend cleanup.

## Operational notes

- Retain protects a volume from automatic backend deletion when its PVC is
  removed. It does not replace backups.
- A Synology snapshot remains on the same NAS and is not an independent
  backup.
- An iSCSI filesystem PVC is single-writer storage. Use NFS/RWX only for a
  workload that genuinely needs simultaneous mounts from multiple nodes.
- Before removing the CSI driver, verify that no PVCs, PVs or
  VolumeAttachments still use csi.san.synology.com.
- After removing the cluster or driver, remove the DSM service account if it
  is no longer required.

Useful checks:

~~~bash
kubectl get pvc -A
kubectl get pv
kubectl get volumeattachment
kubectl -n synology-csi get pods -o wide
kubectl -n synology-csi describe secret client-info-secret
~~~

The final command displays Secret metadata and key sizes, not the credential
contents.
