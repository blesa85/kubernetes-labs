# 03 — Pod troubleshooting

## The problem

Labs usually run in a perfect environment. Real clusters don't: pods get stuck in `Pending`, images don't pull, containers keep crashing. Here I broke one pod on purpose, in a different way each time, and used `get`, `describe`, `logs`, `exec` and `events` to find out why.

The method is always the same: `get pods` for the state, `describe` for the reason, `logs` for what the app said, `exec` to look inside.

## What I did

Same three-node cluster as [lab 01](../01-k3s-cluster). The starting point is an ubuntu pod generated with `--dry-run`:

```bash
kubectl run ubuntu --image=ubuntu -o yaml --dry-run=client -- sleep infinity | tee ubuntu.yaml
```

Each step edits `ubuntu.yaml` and recreates the pod with:

```bash
kubectl replace --force=true --grace-period=0 -f ubuntu.yaml; kubectl get pods --watch
```

`replace --force` deletes the pod and creates it again. A plain `apply` wouldn't work here: most of a pod's spec can't be changed once the pod exists.

### 1. A scheduler that doesn't exist

```yaml
spec:
  schedulerName: none
```

```
NAME       READY   STATUS    RESTARTS   AGE
ubuntu     0/1     Pending   0          0s
```

No scheduler called `none`, so nobody picks the pod up and it never gets a node. In `describe`, the field to look at is `Node:`, which stays `<none>`.

### 2. An invalid image name

```yaml
    image: ubuntuX
```

```
ubuntu     0/1     ContainerCreating   0          0s
ubuntu     0/1     InvalidImageName    0          1s
```

```bash
kubectl describe pod/ubuntu
```

```
Node:             worker-2/172.19.0.7
Status:           Pending
[...]
    State:          Waiting
      Reason:       InvalidImageName
[...]
Events:
  Normal   Scheduled      84s               default-scheduler  Successfully assigned default/ubuntu to worker-2
  Warning  InspectFailed  6s (x8 over 84s)  kubelet            spec.containers{ubuntu}: Failed to apply default image tag "ubuntuX": couldn't parse image name "ubuntuX": invalid reference format: repository name (library/ubuntuX) must be lowercase
  Warning  Failed         6s (x8 over 84s)  kubelet            spec.containers{ubuntu}: Error: InvalidImageName
```

Still `Pending`, but this time with a node. And no `Pulling` event at all: the name is rejected before anything is downloaded.

### 3. A valid name for an image that doesn't exist

```yaml
    image: ubuntux
```

```
ubuntu     0/1     ContainerCreating   0          0s
ubuntu     0/1     ErrImagePull        0          2s
ubuntu     0/1     ImagePullBackOff    0          12s
```

```
Events:
  Normal   Scheduled  34s                default-scheduler  Successfully assigned default/ubuntu to worker-2
  Normal   Pulling    23s (x2 over 35s)  kubelet            spec.containers{ubuntu}: Pulling image "ubuntux"
  Warning  Failed     22s (x2 over 34s)  kubelet            spec.containers{ubuntu}: Failed to pull image "ubuntux": [...] docker.io/library/ubuntux:latest: not found
  Warning  Failed     22s (x2 over 34s)  kubelet            spec.containers{ubuntu}: Error: ErrImagePull
  Normal   BackOff    8s (x2 over 33s)   kubelet            spec.containers{ubuntu}: Back-off pulling image "ubuntux"
  Warning  Failed     8s (x2 over 33s)   kubelet            spec.containers{ubuntu}: Error: ImagePullBackOff
```

```bash
kubectl logs ubuntu
```

```
Error from server (BadRequest): container "ubuntu" in pod "ubuntu" is waiting to start: trying and failing to pull image
```

### 4. A container that finishes

The image fixed, but no `args`:

```yaml
  containers:
  - image: ubuntu
    imagePullPolicy: Always
    name: ubuntu
```

```
ubuntu     0/1     ContainerCreating   0             0s
ubuntu     0/1     Completed           0             1s
ubuntu     0/1     Completed           1 (1s ago)    2s
ubuntu     0/1     CrashLoopBackOff    1 (12s ago)   13s
ubuntu     0/1     Completed           2 (13s ago)   14s
ubuntu     0/1     CrashLoopBackOff    2 (20s ago)   34s
ubuntu     0/1     Completed           3 (21s ago)   35s
```

```
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
    Restart Count:  3
[...]
  Normal   Started    16s (x4 over 50s)  kubelet            spec.containers{ubuntu}: Container started
  Warning  BackOff    16s (x3 over 49s)  kubelet            spec.containers{ubuntu}: Back-off restarting failed container ubuntu [...]
```

The default command of the ubuntu image is `bash`. With no terminal attached it has nothing to do, so it exits with 0. `kubectl logs` came back empty, and `kubectl exec` failed with `container not found`.

### 5. A command that doesn't exist

```yaml
  - args:
    - sleeep
    - infinity
```

```
ubuntu     0/1     ContainerCreating   0             0s
ubuntu     0/1     RunContainerError   0             2s
ubuntu     0/1     RunContainerError   1 (1s ago)    3s
ubuntu     0/1     CrashLoopBackOff    1 (2s ago)    4s
ubuntu     0/1     RunContainerError   2 (0s ago)    17s
ubuntu     0/1     CrashLoopBackOff    2 (26s ago)   43s
ubuntu     0/1     RunContainerError   3 (0s ago)    44s
```

```
    State:          Waiting
      Reason:       RunContainerError
    Last State:     Terminated
      Reason:       StartError
      Message:      [...] exec: "sleeep": executable file not found in $PATH
      Exit Code:    128
      Started:      Thu, 01 Jan 1970 00:00:00 +0000
[...]
  Normal   Created    38s (x4 over 81s)  kubelet            spec.containers{ubuntu}: Container created
  Warning  Failed     38s (x4 over 81s)  kubelet            spec.containers{ubuntu}: Error: [...] exec: "sleeep": executable file not found in $PATH
```

`Created`, but never `Started`. The start date is the Unix epoch: the process never existed.

### 6. A container that keeps running, until I stop it

```yaml
  - args:
    - /bin/sh
    - -c
    - touch /running; ps -p 1 -o lstart=; while [ -f /running ]; do sleep 1; done
```

The script prints when PID 1 started and loops while `/running` exists.

```bash
kubectl logs ubuntu
```

```
Thu Oct 29 14:15:20 2026
```

```bash
kubectl exec ubuntu -- rm /running; kubectl get pods --watch
```

```
ubuntu     1/1     Running     0            25s
ubuntu     0/1     Completed   0            26s
ubuntu     1/1     Running     1 (1s ago)   27s
```

```bash
kubectl logs -p ubuntu
```

```
Thu Oct 29 14:15:20 2026
```

`-p` shows the run before the restart. `describe` has both runs side by side:

```
    State:          Running
      Started:      Thu, 29 Oct 2026 14:15:46 +0000
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Thu, 29 Oct 2026 14:15:20 +0000
      Finished:     Thu, 29 Oct 2026 14:15:45 +0000
```

### The whole story: `kubectl events`

```bash
kubectl events
```

```
7m27s                    Normal    Scheduled   Pod/ubuntu   Successfully assigned default/ubuntu to worker-2
5m16s (x12 over 7m26s)   Warning   Failed      Pod/ubuntu   Error: InvalidImageName
4m33s                    Normal    Scheduled   Pod/ubuntu   Successfully assigned default/ubuntu to worker-2
107s (x5 over 4m32s)     Warning   Failed      Pod/ubuntu   Error: ErrImagePull
78s (x11 over 4m31s)     Warning   Failed      Pod/ubuntu   Error: ImagePullBackOff
71s                      Normal    Scheduled   Pod/ubuntu   Successfully assigned default/ubuntu to worker-2
29s (x3 over 69s)        Warning   BackOff     Pod/ubuntu   Back-off restarting failed container ubuntu [...]
```

(Trimmed. The full output also includes the nodes starting up and the other pod.)

## What I learned

`Node:` in `describe` tells you who to blame. At first I read every `Pending` as "the pod and the scheduler can't find each other", including the bad image name. Wrong: with `ubuntuX` the scheduler had done its job and the pod was on worker-2 with an IP. Pending without a node is a scheduling problem. Pending with a node is an image or container problem.

The events tell the image errors apart. `InvalidImageName` has no `Pulling` event: the name is rejected before trying. With `ubuntux` the pull is attempted, fails with `ErrImagePull`, and then `ImagePullBackOff` is just the waiting between retries, each one longer.

A CrashLoopBackOff can mean two opposite things, and `Last State` says which. `Completed` with `Exit Code: 0` is a process that worked and finished; with `restartPolicy: Always` a pod expects it to keep running, so it gets restarted forever. `StartError` with `Exit Code: 128` and a message is a process that never started. I first blamed the first CrashLoop on the `sleeep` typo, but I hadn't applied that file yet. The exit code was the giveaway.

`kubectl logs` has three answers. An error when the container never started (stuck pulling the image). Empty when it ran but wrote nothing (`bash` with no args). Real output when the app said something, and `-p` for the run before the last restart.

`describe` only shows the events of the current object. Since `replace --force` creates a new pod each time, `kubectl events` was the only place to see the whole history.

A typo can still empty a file. I mistyped `cat` as `Acat` in `Acat <<EOF > ubuntu.yaml`. The command didn't exist, but the shell sets up `> ubuntu.yaml` before running anything, so the file was truncated anyway. The next `replace` said `no objects passed to replace` and the old broken pod stayed as it was. Now I `cat` the file before applying it.
