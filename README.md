# k3s-gitops

Flux setup for the k3s box (Ubuntu desktop). Same layout as
[homelab-gitops](https://github.com/bentastic27/homelab-gitops), trimmed down to the
minecraft servers and the CoreDNS server that lets the Switch reach them.

## What's exposed

| Service | Type | Port(s) on the node |
|---|---|---|
| minecraft (survival) | NodePort | java `30565/tcp`, bedrock `30132/udp` |
| minecraft2 (creative) | NodePort | java `30566/tcp`, bedrock `30133/udp` |
| bedrock (kid) | NodePort | bedrock `30134/udp` |
| bedrockconnect | LoadBalancer (k3s servicelb) | `19132/udp` |
| coredns | LoadBalancer (k3s servicelb) | `53/udp`, `53/tcp` |

bedrockconnect and coredns can't be NodePorts: the Switch only talks to them on their
standard ports (53 and 19132), which sit below the NodePort range. They use k3s's
built-in servicelb, so don't install k3s with `--disable servicelb`.

## Switch setup

Point the Switch's primary DNS at the node IP. The featured servers resolve to the node,
which lands on bedrockconnect, whose server list points at the NodePorts above.

## Init

```
kubectl create ns flux

# node IP used by the coredns hosts entries and bedrockconnect's server list
kubectl -n flux create configmap cluster-vars --from-literal=NODE_IP=192.168.1.x

flux bootstrap -n flux github \
  --owner=bentastic27 \
  --repository=k3s-gitops \
  --private=false \
  --personal=true \
  --path=flux
```
