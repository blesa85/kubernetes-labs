# 01 — Building a multi-node K3s cluster

## The problem

Most local Kubernetes setups give you a single node, which hides half of what the scheduler does. K3s is light enough to run a control plane plus two workers, so you can actually watch pods being placed across machines.

## What I did

### Control plane

```bash
curl -sfLk https://get.k3s.io | \
  INSTALL_K3S_VERSION=v1.34.1+k3s1 \
  K3S_TOKEN=KCNA \
  INSTALL_K3S_EXEC="--disable traefik --kubelet-arg=eviction-hard=imagefs.available<1%,nodefs.available<1%" \
  sh -
```

Two flags worth noting. `--disable traefik` skips the ingress controller K3s ships by default, which I don't need here. The `eviction-hard` values stop the kubelet from evicting pods on a lab box with very little free disk.

`K3S_TOKEN` is the shared secret the workers use to join later. It has to be the same value on every node.

The installer symlinks its own binary as `kubectl`:

```bash
ls -altrh /usr/local/bin/kubectl
# /usr/local/bin/kubectl -> k3s
```

### Kubeconfig

K3s writes its config to `/etc/rancher/k3s/k3s.yaml`. I moved it to `~/.kube/config`, where every other Kubernetes distribution expects to find it — K3s works with either location.

```bash
mkdir -p ~/.kube && mv /etc/rancher/k3s/k3s.yaml ~/.kube/config
kubectl config view
kubectl get nodes
```

```
NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   22s   v1.34.1+k3s1
```

I use `kubectl config view` rather than `cat` on the file: it shows the same structure with the certificates and private key replaced by `DATA+OMITTED`. That file holds cluster-admin credentials and should never leave the machine.

### Joining the workers

```bash
ssh worker-1 'curl -sfLk https://get.k3s.io | \
  INSTALL_K3S_VERSION=v1.34.1+k3s1 \
  K3S_URL=https://control-plane:6443 \
  K3S_TOKEN=KCNA \
  INSTALL_K3S_EXEC="--kubelet-arg=eviction-hard=imagefs.available<1%,nodefs.available<1%" \
  sh -'
```

Same command again for `worker-2`. The only difference from the control plane is `K3S_URL`, and that single variable is what makes the installer set the node up as an agent instead of a server — it creates `k3s-agent.service` instead of `k3s.service`.

```bash
kubectl get nodes
```

```
NAME            STATUS   ROLES           AGE     VERSION
control-plane   Ready    control-plane   4m16s   v1.34.1+k3s1
worker-1        Ready    <none>          3m33s   v1.34.1+k3s1
worker-2        Ready    <none>          5s      v1.34.1+k3s1
```

## What I learned

Joining is fast. The control plane was `Ready` in 22 seconds and each worker in about 5. Useful as a baseline: next time a node takes minutes to register, something is wrong rather than slow.

The `<none>` role on the workers isn't an error. K3s doesn't label agents with a role, and the label is cosmetic anyway — `kubectl label node worker-1 node-role.kubernetes.io/worker=` sets it if you want it in the output.

In K3s the control plane also runs workloads, because it doesn't carry the `NoSchedule` taint that kubeadm applies by default. Worth remembering before assuming pods will only land on the workers.

A wrong token fails quietly. The agent installs and the service starts, but the node never appears in `kubectl get nodes` — the authentication error only shows up in `journalctl -u k3s-agent`. Which is a good reminder that the node list is not where you diagnose a node that never arrived.
