# traffic-lab

A hands-on Kubernetes project on minikube (3 nodes). It covers how pods get a place on nodes, how they scale, how they survive maintenance and updates, and what happens when requests and limits are wrong.

See `CHEATSHEET.md` for a short summary of every topic.

## Setup

```bash
minikube start -p traffic-lab --nodes 3 --cpus 2 --memory 2048
minikube addons enable metrics-server -p traffic-lab
kubectl create namespace traffic-lab
kubectl config set-context --current --namespace=traffic-lab
```

- Three nodes, so the scheduler has a real choice.
- metrics-server is needed for `kubectl top` and HPA.
- The namespace is set as the default, so `-n traffic-lab` is not needed.

Note: with the Docker driver, Kubernetes sees all CPUs of the host machine on every node (12 here), even though `--cpus 2` limits the real CPU time. The scheduler counts the number it sees.

## Requests and limits

File: `deployment.yaml`

- **Request** is the amount a container books on a node. The scheduler uses it to decide where a pod can go.
- **Limit** is the maximum a container may use.
- CPU is measured in millicores: `1000m` = 1 CPU.

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

## CPU throttling

- When a container wants more CPU than its limit, it is not killed. Linux pauses it for part of every 100 ms slice, so it becomes slower.
- `kubectl top` shows an average, so it can hide throttling. The real proof is in the container's cgroup file:

```bash
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu.stat
```

- `nr_throttled` and `throttled_usec` keep rising while the pod is under load.
- Rule: CPU over the limit is throttled. Memory over the limit is killed.

## Taints, tolerations and affinity

File: `batch.yaml`

- **Taint** is a mark on a node: "keep out unless you accept this".
- **Toleration** is on a pod: "I accept that taint". It only gives permission.
- **Node affinity** is on a pod: "I want a node with this label". It pulls the pod there.

```bash
kubectl taint nodes traffic-lab-m03 dedicated=batch:NoSchedule
kubectl label node traffic-lab-m03 workload=batch
```

The `batch` Deployment has a matching toleration and a required node affinity, so its pods land only on that node. The `web` pods have no toleration, so they stay away from it.

## Topology spread

- Keeps pods evenly spread across nodes so one node failure does not take down the whole app.
- `maxSkew: 1` means the pod count of two nodes may differ by at most 1.
- `topologyKey: kubernetes.io/hostname` means each node is one place.
- `whenUnsatisfiable: DoNotSchedule` makes it a hard rule. `ScheduleAnyway` makes it a soft preference.
- `nodeTaintsPolicy: Honor` ignores nodes the pod cannot use (the tainted node), so the spread is calculated correctly.
- The rule is checked only when a pod is scheduled. After a rolling update or a node restart, pods can end up uneven, and nothing moves them. Deleting a pod from the crowded node fixes it.

## HPA (Horizontal Pod Autoscaler)

File: `hpa.yaml`

- Changes the number of pods based on CPU use.
- Needs metrics-server and a CPU request on the container, because usage is measured as a percentage of the request.
- `averageUtilization: 50` with a `100m` request means the target is about 50m per pod.
- Formula: `desired pods = current pods x (current usage / target usage)`.
- Scale-up is quick. Scale-down waits about 5 minutes to avoid going up and down too often.
- When HPA owns the replica count, remove `replicas` from the Deployment file. The HPA overrides it.
- After editing `hpa.yaml`, run `kubectl apply -f hpa.yaml`, otherwise the cluster keeps the old target.

## Load test

```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://web; done"
kubectl get hpa web -w
```

- Pods increased as CPU went above the target, and the new pods spread across the usable nodes.
- One load generator was too light. More generators were needed to push CPU above the target.
- `kubectl describe hpa web` shows each scaling decision in Events.

## Pending pods (insufficient CPU)

File: `hungry.yaml`

- The scheduler counts what pods book (requests), not what they really use.
- Each pod booked 5 CPUs. A node with 12 CPUs fits 2 such pods. A pod must fit on one node, so leftover CPU on different nodes cannot be joined.
- 6 pods asked, 4 were placed, 2 stayed Pending. The tainted node was closed to them.
- Reason shown in events: `Insufficient cpu` and `untolerated taint`.

```bash
kubectl describe pod <pending-pod>
kubectl describe nodes | grep -A 4 "Allocated resources"
```

- `kubectl top nodes` showed almost no real use, while the nodes were about 90% booked. The nodes were idle but booked.
- Fix: reduce the request (to `250m`). All 6 pods then fit.
- On GKE, the cluster autoscaler would add a node when pods are Pending for lack of resources. Minikube has no node autoscaler.

## Memory limit and OOMKilled

File: `oom-demo.yaml`

- A container that goes over its memory limit is killed immediately.
- The `stress` container tried to use 150 MB with a 100Mi limit. It got `OOMKilled`, `Exit Code: 137`, then `CrashLoopBackOff`.
- The limit protects the node: only the container that goes over is killed.
- Fix: raise the memory limit (to `256Mi`).

```bash
kubectl describe pod -l app=oom-demo | grep -A 8 "Last State"
```

## VPA (Vertical Pod Autoscaler)

Files: `vpa-demo.yaml`, `vpa.yaml`

- HPA changes the number of pods. VPA looks at real usage and suggests better request values.
- VPA is not built into minikube. It was installed from the kubernetes/autoscaler repository with `./hack/vpa-up.sh`.
- `updateMode: "Off"` only gives recommendations and changes nothing. `Recreate` deletes and recreates pods with new values.
- The demo container really used about 1 CPU and 100 MB, but requested 50m CPU and 1Gi memory. VPA recommended a higher CPU and a lower memory request.
- `kubectl describe vpa vpa-demo` shows Lower Bound, Target and Upper Bound. Target is the suggested request.
- Do not let VPA and HPA both act on CPU for the same Deployment.

## PodDisruptionBudget and node drain

File: `pdb.yaml`

- `kubectl drain <node>` empties a node by evicting its pods, for example before maintenance.
- A PDB tells the drain: keep at least this many pods of my app running.
- `minAvailable` counts the pods of the app in the whole cluster, not per node.
- Formula: `allowed disruptions = running pods - minAvailable`.
- With 2 pods and `minAvailable: 2`, the drain was blocked with `Cannot evict pod as it would violate the pod's disruption budget`.
- With `minAvailable: 1`, the drain evicted pods one at a time and waited for replacements.
- A PDB protects only against voluntary disruptions such as a drain. It does not block `kubectl delete pod`, HPA scale-down or a node crash.
- A hard topology spread rule and a cordoned node can leave a replacement pod Pending.

```bash
kubectl drain traffic-lab-m02 --ignore-daemonsets --delete-emptydir-data --timeout=120s
kubectl uncordon traffic-lab-m02
```

- Drain a worker node, not the control-plane node.

## Graceful shutdown

File: `graceful.yaml`

- When a pod stops: it is marked Terminating, the `preStop` hook runs, SIGTERM is sent, Kubernetes waits for the grace period, and SIGKILL is sent only if the app is still running.
- An app that ignores SIGTERM took the full 30 seconds to stop. An app that caught SIGTERM stopped in about 0.5 seconds.
- `terminationGracePeriodSeconds` sets how long Kubernetes waits after SIGTERM. The default is 30.
- A `preStop` hook (for example `sleep 5`) gives the Service time to stop sending traffic to the pod before the app starts shutting down.

```yaml
terminationGracePeriodSeconds: 15
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]
```

## Rolling updates

File: `rolling.yaml`

- A rolling update replaces pods a few at a time, so the app stays up.
- `maxSurge` is how many extra pods may exist above the replica count. `maxUnavailable` is how many pods may be missing.
- With 4 replicas, `maxSurge: 1` and `maxUnavailable: 1` give up to 5 pods in total and at least 3 ready.
- `maxSurge: 1` with `maxUnavailable: 0` keeps all 4 pods ready, but needs room for 1 extra pod and is slower.
- The readiness probe decides when a new pod is ready, so it controls the speed of the update.
- Kubernetes keeps the old ReplicaSet, so `kubectl rollout undo deployment <name>` goes back to the previous version.

```bash
kubectl rollout status deployment rolling
kubectl rollout history deployment rolling
kubectl rollout undo deployment rolling
```

## Fixing poor requests (right-sizing)

- Compare what pods really use with what they book:

```bash
kubectl top pods -l app=web
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

- The `web` pods used about 11Mi of memory but requested 64Mi (limit 128Mi), so memory was booked far too big. Under load, CPU hit the 200m limit and was throttled, so the CPU limit was too small.
- New values:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 24Mi
  limits:
    cpu: 500m
    memory: 64Mi
```

- CPU request stays at 100m because HPA measures usage as a percentage of it.
- Result: less throttling under load, and lower memory requests on the nodes, so more pods can fit.
- A request that is too big wastes room and causes Pending pods. A limit that is too small causes throttling or OOMKills. Use real usage (and VPA recommendations) to choose the numbers.

## Clean up

```bash
minikube delete -p traffic-lab
```
