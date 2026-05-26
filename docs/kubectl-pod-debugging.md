# kubectl pod debugging

A field guide for the common pod-debugging loop: find the pod, read its status, inspect logs, jump in for a shell, move files around, forward a port, and tidy up. Every command targets a specific pod in a specific namespace and context — adjust the placeholders to suit your cluster.

Default values used below:

- context: `my-cluster`
- namespace: `default`
- pod: `my-app-7d9b4c5f8-xyz12`
- container: `app`
- local port: `8080`
- remote port: `8080`
- local file: `./debug.log`
- remote file: `/var/log/app.log`
- tail lines: `100`

## 1. List pods and get status

List pods in the namespace with extra detail (node, IP, restart count):

```bash
kubectl --context=my-cluster -n default get pods -o wide
```

Dump the full spec/status YAML for a single pod — the source of truth for restart counts, container states, conditions, and image digests:

```bash
kubectl --context=my-cluster -n default get pod my-app-7d9b4c5f8-xyz12 -o yaml
```

## 2. Describe a pod

A human-readable rollup of spec, status, container states, recent events, and the kubelet's view of conditions. Usually your first stop when a pod is stuck in `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff`:

```bash
kubectl --context=my-cluster -n default describe pod my-app-7d9b4c5f8-xyz12
```

## 3. Stream and fetch logs

Snapshot the last 100 lines from the pod (single container or default container):

```bash
kubectl --context=my-cluster -n default logs my-app-7d9b4c5f8-xyz12 --tail=100
```

Follow logs for a specific container, including the previous container instance — handy after a crash loop where the live container has restarted but you want the logs that caused the crash:

```bash
kubectl --context=my-cluster -n default logs my-app-7d9b4c5f8-xyz12 -c app --previous --tail=100 -f
```

## 4. Exec into a container

Open an interactive shell. Most distroless images don't ship a shell; try `/bin/bash`, `/bin/sh`, or `/busybox/sh` as fallbacks:

```bash
kubectl --context=my-cluster -n default exec -it my-app-7d9b4c5f8-xyz12 -c app -- /bin/sh
```

Run a one-shot command without an interactive session — useful in scripts and CI:

```bash
kubectl --context=my-cluster -n default exec my-app-7d9b4c5f8-xyz12 -c app -- ls -la /var/log
```

## 5. Copy files in and out

Pull a file from the pod down to your local machine. Requires `tar` to be present in the container image:

```bash
kubectl --context=my-cluster -n default cp my-app-7d9b4c5f8-xyz12:/var/log/app.log ./debug.log -c app
```

Push a local file into the pod (e.g. a config snippet or debug tool):

```bash
kubectl --context=my-cluster -n default cp ./debug.log my-app-7d9b4c5f8-xyz12:/var/log/app.log -c app
```

## 6. Port-forward

Forward a local port to a pod's port — open a private tunnel without exposing the pod via a Service. Leave it running in a separate terminal:

```bash
kubectl --context=my-cluster -n default port-forward pod/my-app-7d9b4c5f8-xyz12 8080:8080
```

## 7. Events and resource usage

Recent events sorted by time — this surfaces scheduler decisions, image pulls, OOM kills, probe failures, and evictions across the namespace:

```bash
kubectl --context=my-cluster -n default get events --sort-by=.lastTimestamp
```

Live CPU/memory usage per container in the pod (requires metrics-server):

```bash
kubectl --context=my-cluster -n default top pod my-app-7d9b4c5f8-xyz12 --containers
```

## 8. Delete (force-restart)

Delete the pod. If it's owned by a Deployment / StatefulSet / ReplicaSet the controller will recreate it — effectively a forced restart. The default `--cascade=background` means dependents are GC'd asynchronously:

```bash
kubectl --context=my-cluster -n default delete pod my-app-7d9b4c5f8-xyz12
```
