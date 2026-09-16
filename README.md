# K3s Homelab

Kubernetes manifests for my three-node K3s homelab.

## Cluster

- k3s-01: 192.168.100.201
- k3s-02: 192.168.100.202
- k3s-03: 192.168.100.203

All three nodes run the control plane, embedded etcd and workloads.

## Applications

### Headlamp

Kubernetes management dashboard available internally at:

https://headlamp.k3s.joeyme.eu

Apply the manifests:

kubectl apply -f apps/headlamp/headlamp.yaml
kubectl apply -f apps/headlamp/ingress.yaml
