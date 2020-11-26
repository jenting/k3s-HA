# k3s-HA

The control plane nodes _must be_ odd number.
```shell
# 1st control plane node
systemctl disable firewalld --now
curl -sfL https://get.k3s.io | sh -s - server --cluster-init
SERVER_NODE_TOKEN=`sudo cat /var/lib/rancher/k3s/server/node-token`

# 2nd, 3rd control plane nodes
systemctl disable firewalld --now
curl -sfL https://get.k3s.io | K3S_URL=https://<1st_CONTROL_PLANE_NODE_IP_FQDN>:6443 K3S_TOKEN=$SERVER_NODE_TOKEN sh -s - server

# worker nodes
systemctl disable firewalld --now
curl -sfL https://get.k3s.io | K3S_URL=https://<1st_CONTROL_PLANE_NODE_IP_FQDN>:6443 K3S_TOKEN=$SERVER_NODE_TOKEN sh -
```

Install the system-upgrade-controller.
```shell
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.2/system-upgrade-controller.yaml

cat <<EOF | kubectl apply -f -
# Server plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  channel: https://update.k3s.io/v1-release/channels/stable
---
# Agent plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 2
  cordon: true
  drain:
    force: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: NotIn
      values:
      - "true"
  prepare:
    args:
    - prepare
    - server-plan
    image: rancher/k3s-upgrade
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  channel: https://update.k3s.io/v1-release/channels/stable
EOF
```

References:
1. https://rancher.com/docs/k3s/latest/en/
2. https://github.com/rancher/system-upgrade-controller
3. https://github.com/rancher/k3s-upgrade
