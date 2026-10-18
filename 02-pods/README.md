# 02 — Pods

## The problem

The pod is the smallest thing Kubernetes deploys, and it's easy to read it as "one container". It isn't: it's a wrapper for one or more containers that share an IP. I wanted to run pods by hand, reach them across nodes, move from commands to YAML, and then put two containers in the same pod and break one of them on purpose.

## What I did

Same layout as [lab 01](../01-k3s-cluster): a control plane and two workers.

### A pod, imperatively

```bash
kubectl run nginx --image=nginx
kubectl get pods -o wide
```

```
NAME    READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          56s   10.42.1.2   worker-1   <none>           <none>
```

`kubectl logs pod/nginx` shows the nginx entrypoint output, which is just whatever the container writes to stdout.

### Reaching it from every node

```bash
NGINX_IP=$(kubectl get pods -o wide | awk '/nginx/ { print $6 }'); echo $NGINX_IP
ping -c 3 $NGINX_IP
ssh worker-1 ping -c 3 $NGINX_IP
ssh worker-2 ping -c 3 $NGINX_IP
```

```
# from control-plane
64 bytes from 10.42.1.2: icmp_seq=1 ttl=63 time=0.631 ms
# from worker-1
64 bytes from 10.42.1.2: icmp_seq=1 ttl=64 time=0.056 ms
# from worker-2
64 bytes from 10.42.1.2: icmp_seq=1 ttl=63 time=0.214 ms
```

All three nodes reach the pod directly by its IP, with no NAT and nothing configured. Worth looking at the TTL: 64 from worker-1, where the pod lives, and 63 from the other two, one routing hop away.

### From another pod

A throwaway pod that runs `curl` once and deletes itself:

```bash
kubectl run -it --rm curl --image=curlimages/curl:8.4.0 --restart=Never -- http://$NGINX_IP
```

```
<title>Welcome to nginx!</title>
...
pod "curl" deleted from default namespace
```

And a longer-lived one, with the IP passed in as an environment variable:

```bash
kubectl run --image=ubuntu ubuntu --env="NGINX_IP=$NGINX_IP" -- sleep infinity
kubectl get pods -o wide
```

```
NAME     READY   STATUS    RESTARTS   AGE     IP          NODE       NOMINATED NODE   READINESS GATES
nginx    1/1     Running   0          3m15s   10.42.1.2   worker-1   <none>           <none>
ubuntu   1/1     Running   0          14s     10.42.2.3   worker-2   <none>           <none>
```

```bash
kubectl exec -it ubuntu -- bash
ps -ef
```

```
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 16:42 ?        00:00:00 sleep infinity
root           7       0  0 16:42 pts/0    00:00:00 bash
root          14       7  0 16:42 pts/0    00:00:00 ps -ef
```

PID 1 is `sleep infinity` and nothing else. No init system, no services: the container is the process. After `apt update && apt install -y curl`, `curl $NGINX_IP` from worker-2 returned the nginx page served from worker-1.

### From imperative to declarative

```bash
kubectl delete pod/nginx pod/ubuntu --now
kubectl run nginx --image=nginx --dry-run=client -o yaml | tee nginx.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

`--dry-run=client -o yaml` builds the object locally without sending it to the API server. `kubectl run` adds a `run: nginx` label by itself, and fills in `restartPolicy: Always`. `kubectl explain pod.spec.restartPolicy` confirms the three options (`Always`, `Never`, `OnFailure`) and that `Always` is the default.

```bash
kubectl create -f nginx.yaml
kubectl create -f nginx.yaml
```

```
pod/nginx created
Error from server (AlreadyExists): error when creating "nginx.yaml": pods "nginx" already exists
```

```bash
kubectl apply -f nginx.yaml
kubectl apply -f nginx.yaml
```

```
Warning: resource pods/nginx is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. [...] The missing annotation will be patched automatically.
pod/nginx configured
pod/nginx unchanged
```

`create` is imperative and fails if the object exists. `apply` is declarative and idempotent: the second run is just `unchanged`.

The same trick for the ubuntu pod, then both in one file separated by `---`:

```bash
kubectl run --image=ubuntu --dry-run=client -o yaml ubuntu sleep infinity | tee ubuntu.yaml
{ cat nginx.yaml; echo "---"; cat ubuntu.yaml; } | tee combined.yaml
kubectl apply -f combined.yaml
```

```
pod/nginx created
pod/ubuntu created
```

`kubectl delete -f combined.yaml --now` removes both the same way.

### Two containers, one pod

A webserver plus a sidecar that logs every 5 seconds, and exits with an error if `/tmp/crash` exists:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mypod
  name: mypod
spec:
  containers:
  - image: nginx
    name: webserver
    resources: {}
  - image: ubuntu
    name: sidecar
    args:
    - /bin/sh
    - -c
    - while true; do echo "$(date +'%T') - Hello from the sidecar"; sleep 5; if [ -f /tmp/crash ]; then exit 1; fi; done
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

```bash
kubectl apply -f combined.yaml
kubectl get pods -o wide
```

```
NAME    READY   STATUS              RESTARTS   AGE   IP       NODE       NOMINATED NODE   READINESS GATES
mypod   0/2     ContainerCreating   0          3s    <none>   worker-1   <none>           <none>
```

```bash
kubectl describe pod/mypod
```

```
IP:               10.42.1.5
[...]
Events:
  Normal  Scheduled  8s    default-scheduler  Successfully assigned default/mypod to worker-1
  Normal  Pulled     7s    kubelet            spec.containers{webserver}: Successfully pulled image "nginx" in 347ms [...]
  Normal  Started    7s    kubelet            spec.containers{webserver}: Container started
  Normal  Pulled     2s    kubelet            spec.containers{sidecar}: Successfully pulled image "ubuntu" in 4.497s [...]
  Normal  Started    2s    kubelet            spec.containers{sidecar}: Container started
```

One pod, one IP, two containers. Curling `10.42.1.5` from a throwaway pod returns the nginx page, served by the `webserver` container. The sidecar is reached with `-c`:

```bash
kubectl logs pod/mypod -c sidecar
```

```
16:44:44 - Hello from the sidecar
16:44:49 - Hello from the sidecar
16:44:54 - Hello from the sidecar
16:44:59 - Hello from the sidecar
```

### Breaking the sidecar

```bash
kubectl exec -it mypod -c sidecar -- touch /tmp/crash
kubectl get pods -o wide
```

```
NAME    READY   STATUS    RESTARTS     AGE   IP          NODE       NOMINATED NODE   READINESS GATES
mypod   2/2     Running   1 (3s ago)   34s   10.42.1.5   worker-1   <none>           <none>
```

```bash
kubectl logs pod/mypod -c sidecar
kubectl logs pod/mypod -p -c sidecar
```

```
# current
16:45:09 - Hello from the sidecar
16:45:14 - Hello from the sidecar
# previous (-p)
16:44:44 - Hello from the sidecar
[...]
16:45:04 - Hello from the sidecar
```

## What I learned

The restart happens at container level, not pod level. The sidecar died, the kubelet started it again, and `RESTARTS` went to 1. The pod itself didn't change: same IP, same age, still `2/2`. The current logs start from scratch, and `-p` is the only way to see what the dead container wrote before it exited. In a real crash loop that's the useful half.

Pod IPs don't survive the pod. nginx was `10.42.1.2`, then `10.42.1.3`, then `10.42.1.4` each time I recreated it. Passing the IP by hand in an environment variable, as I did with the ubuntu pod, works in a lab and is exactly what you can't do for real. That's the problem Services solve.

Each node hands out IPs from its own range: everything on worker-1 got `10.42.1.x`, everything on worker-2 `10.42.2.x`. You can tell where a pod lives from its address.

The arguments after the pod name in `kubectl run` go into `args`, not `command`. The ubuntu YAML has `args: [sleep, infinity]` and no `command`, so they replace the image's default `CMD` and leave any `ENTRYPOINT` alone.

Mixing `create` and `apply` leaves a trace. `apply` works on an object made with `create`, but warns about the missing `last-applied-configuration` annotation and patches it in. Picking one approach per object avoids it.
