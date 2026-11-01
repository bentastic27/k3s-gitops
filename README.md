# k3s-gitops

Flux setup for the k3s box (Ubuntu desktop). Same layout as
[homelab-gitops](https://github.com/bentastic27/homelab-gitops), trimmed down to the
minecraft servers and the CoreDNS server that lets the Switch reach them.

## What's exposed

| Service | Type | Port(s) on the node |
|---|---|---|
| survival | NodePort | java `30565/tcp`, bedrock `30132/udp` |
| creative | NodePort | java `30566/tcp`, bedrock `30133/udp` |
| bedrockconnect | hostPort | `19132/udp` |
| coredns | host network, bound to `NODE_IP` | `53/udp`, `53/tcp` |

bedrockconnect and coredns can't be NodePorts: the Switch only talks to them on their
standard ports (53 and 19132), which sit below the NodePort range. bedrockconnect uses
a hostPort.

coredns runs on the host network bound to `NODE_IP` only. A hostPort (or servicelb) on 53
hijacks the host's own DNS (127.0.0.53), which breaks image pulls whenever the pod is
down. If the node's IP changes, update the `cluster-vars` ConfigMap or coredns won't
start.

## Switch setup

Point the Switch's primary DNS at the node IP. The featured servers resolve to the node,
which lands on bedrockconnect, whose server list points at the NodePorts above.

## Init

```
kubectl create ns flux

# node IP used by the coredns hosts entries and bedrockconnect's server list
kubectl -n flux create configmap cluster-vars --from-literal=NODE_IP=192.168.1.x

# public repo, so no PAT or deploy key: flux pulls over anonymous HTTPS
kubectl apply -f https://raw.githubusercontent.com/bentastic27/k3s-gitops/main/flux/flux/gotk-components.yaml
kubectl apply -f https://raw.githubusercontent.com/bentastic27/k3s-gitops/main/flux/flux/gotk-sync.yaml
```
