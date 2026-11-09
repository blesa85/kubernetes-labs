# 04 — Namespaces

## The problem

A cluster is shared: system components, my own pods, other teams' pods. Namespaces split it into logical spaces. I wanted to see what's already there out of the box, which resources live inside a namespace and which don't, and how kubectl decides which namespace I'm looking at.

## What I did

Same three-node cluster as [lab 01](../01-k3s-cluster).

### What's already running

```bash
kubectl get all -A
```

```
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-54bf7cdff9-4nc4p                  1/1     Running   0          3m6s
kube-system   pod/local-path-provisioner-69879d7dd7-lqqmb   1/1     Running   0          3m6s
kube-system   pod/metrics-server-77dbbf84b-xq2v8            1/1     Running   0          3m6s

NAMESPACE     NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes       ClusterIP   10.43.0.1      <none>        443/TCP                  3m14s
kube-system   service/kube-dns         ClusterIP   10.43.0.10     <none>        53/UDP,53/TCP,9153/TCP   3m9s
kube-system   service/metrics-server   ClusterIP   10.43.139.79   <none>        443/TCP                  3m9s
[...]
kube-system   deployment.apps/coredns                  1/1     1            1           3m9s
[...]
```

Three things stood out. The DNS runs as `coredns` (pod, deployment) behind a service called `kube-dns`. There's a `service/kubernetes` in `default` that I didn't create. And there's no pod for the API server, scheduler or kube-proxy.

```bash
kubectl get ns
```

```
NAME              STATUS   AGE
default           Active   5m40s
kube-node-lease   Active   5m40s
kube-public       Active   5m40s
kube-system       Active   5m40s
```

### Namespaced or not

```bash
kubectl api-resources | more
```

```
NAME                     SHORTNAMES   APIVERSION   NAMESPACED   KIND
configmaps               cm           v1           true         ConfigMap
namespaces               ns           v1           false        Namespace
nodes                    no           v1           false        Node
persistentvolumes        pv           v1           false        PersistentVolume
pods                     po           v1           true         Pod
resourcequotas           quota        v1           true         ResourceQuota
services                 svc          v1           true         Service
[...]
leases                                coordination.k8s.io/v1   true    Lease
[...]
clusterroles                          rbac.authorization.k8s.io/v1   false   ClusterRole
roles                                 rbac.authorization.k8s.io/v1   true    Role
```

The `SHORTNAMES` column is where `ns`, `po`, `svc`, `cm` come from.

### Choosing a namespace: `-n`

```bash
kubectl create namespace mynamespace
kubectl -n mynamespace run nginx --image=nginx
kubectl get pods
```

```
No resources found in default namespace.
```

```bash
kubectl -n mynamespace get pods
```

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          16s
```

### Choosing a namespace: the context

```bash
kubectl config view
```

```yaml
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
```

No `namespace` key at all, and kubectl still uses `default`.

```bash
kubectl config set-context --current --namespace=mynamespace
kubectl config view
```

```yaml
contexts:
- context:
    cluster: default
    namespace: mynamespace
    user: default
  name: default
```

```bash
kubectl get pods
```

```
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          58s
```

Back to `default` with `kubectl config set-context --current --namespace=default`, and `kubectl delete namespace/mynamespace --now` removes the namespace and the pod inside it.

### Same name, two namespaces

```bash
kubectl create namespace mynamespace
kubectl -n mynamespace run nginx --image=nginx
kubectl run nginx --image=nginx
kubectl get pods -A
```

```
NAMESPACE     NAME                                      READY   STATUS              RESTARTS   AGE
default       nginx                                     0/1     ContainerCreating   0          7s
kube-system   coredns-54bf7cdff9-hg2x5                  1/1     Running             0          39s
kube-system   local-path-provisioner-69879d7dd7-qvn4j   1/1     Running             0          39s
kube-system   metrics-server-77dbbf84b-29c64            1/1     Running             0          39s
mynamespace   nginx                                     1/1     Running             0          13s
```

Two pods called `nginx`, no conflict.

### What's in `kube-node-lease`

```bash
kubectl -n kube-node-lease get leases
```

```
NAME            HOLDER          AGE
control-plane   control-plane   40s
worker-1        worker-1        34s
worker-2        worker-2        34s
```

### Deleting the namespace I'm standing in

```bash
kubectl config set-context --current --namespace=mynamespace
kubectl delete namespace/mynamespace --now
kubectl get pods
```

```
No resources found in mynamespace namespace.
```

```bash
kubectl config set-context --current --namespace=default
kubectl delete pod/nginx --now
```

## What I learned

`get all` isn't all. It shows pods, services, deployments, replicasets and a few more, but not leases, configmaps or secrets. `kube-node-lease` looked empty and wasn't: it holds one lease per node, renewed by each kubelet as a heartbeat to the control plane.

On K3s the core of Kubernetes isn't visible as pods. I expected kube-proxy, the API server and the scheduler in `kube-system`. K3s runs them inside the single `k3s` process, so only the add-ons show up as pods: CoreDNS, metrics-server and local-path-provisioner. On a kubeadm cluster they would be pods.

A Service name is an address, not the software behind it. I first guessed `kube-dns` had something to do with the kubelet. It doesn't: the DNS server is CoreDNS, and the Service keeps the old name `kube-dns` because every pod already points at that IP. The same idea explains `service/kubernetes`: it's the API server, published inside the cluster so pods can reach it.

`-n` is for one command. `set-context --namespace` writes a `namespace:` line into the kubeconfig, and it stays until I change it. Without that line, kubectl falls back to `default`.

Namespaced or not depends on whether the thing makes sense scoped to one space. I first thought authorization-related resources were cluster-level, but `roles` and `rolebindings` are namespaced, while `clusterroles` aren't. Nodes, persistent volumes and namespaces themselves belong to the whole cluster. A namespace can't live inside a namespace.

Names are only unique within a namespace, so two `nginx` pods can coexist. And `-A` means all namespaces, not "system ones too": I predicted it would show only `default` and got everything, with a `NAMESPACE` column that only appears with `-A`.

The kubeconfig lives on my machine and the namespace lives in the cluster. Deleting `mynamespace` didn't touch my context. I predicted it would fall back to `default`, but kubectl kept looking in a namespace that no longer existed and answered "No resources found" instead of an error. Listing doesn't check whether the namespace exists. In a real cluster that's how you lose half an hour looking for a pod that's alive somewhere else. The fix is to read the end of the message: `in mynamespace namespace`.
