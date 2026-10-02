# Introduction to NRP

Sethurman Lab Meeting Technical Demo Oct,2 2026.

Meeting recording:

<video width="750" height="500" controls>
  <source src="https://drive.google.com/embeddedfolderview?id=1hJK8taPneG3XYZnrcq650Bk9qIGe4RU4#list">
</video>

## Overview

The National Research Platform (NRP) pools distributed compute, storage, and open-weight LLM hosting resources. Nautilus is its federated Kubernetes cluster. Participating institutions contribute hardware and receive priority on their contributed resources; the preparation notes describe access for US-based nonprofit research and education institutions.

**Workshop goal:** submit a Pod, run single and parallel batch Jobs, inspect logs, and preserve output using a PersistentVolumeClaim (PVC).

**Command status:** these are instructional examples, not an execution log. The source page does not record command outputs or confirm which commands were run. Submission, inspection, and cleanup commands below were added to make the examples usable end to end.

## 1. Containers, Kubernetes, and the NRP portal

**Summary:** a container packages an application and its dependencies. Kubernetes schedules and manages containerized workloads across compute nodes.

| Topic | Purpose |
| --- | --- |
| Container | Packages code and dependencies in a reusable image. |
| Kubernetes (K8s) | Schedules workloads and manages resources and desired state. |
| Pod | Runs one or more containers together. |
| Job | Runs finite batch work and tracks successful completions. |
| Deployment | Maintains replicas for long-running applications and supports rolling updates. |
| PVC | Requests persistent storage that can be mounted by Pods. |

Use the portal to inspect resources, namespaces, running Pods, and LLM API keys:

- [NRP portal](https://nrp.ai/)
- [Available resources](https://nrp.ai/viz/resources/)
- [Your namespaces](https://nrp.ai/namespaces/)

Check resource availability before requesting specialized GPUs or large memory allocations.

## 2. Prerequisites and command conventions

**Summary:** cluster access, namespace membership, and local Kubernetes tools must be ready before submitting workloads.

- NRP access through Authentik.
- Membership in at least one namespace.
- `kubectl` and the `kubelogin` authentication plugin installed.
- Kubernetes configuration at `~/.kube/config`.
- Access to the NRP portal.

The original notes mention installation and configuration steps but do not contain them. Complete these separately before the hands-on exercises.

**Replace every placeholder** before running commands or saving YAML:

- `<namespace>`: your authorized namespace.
- `<username>`: a unique, Kubernetes-compatible identifier using lowercase letters, numbers, and hyphens.
- `<pod-name>`: an actual Pod name returned by `kubectl get pods -n your-namespace`.

Save each manifest under the filename given below. Angle-bracket placeholders are not valid runnable shell arguments; do not paste them unchanged. Every cluster command specifies `-n <namespace>` to avoid using the wrong namespace.

```bash
# Added readiness checks
kubectl config get-contexts
# Should print
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         nautilus   nautilus   oidc

```

## 3. Pod: Hello World

**Summary:** a Pod is Kubernetes’ smallest deployable unit. This example prints a greeting and sleeps for one hour so it can be inspected. A standalone Pod does not provide a Job controller’s completion tracking; restart behavior depends on its restart policy.

### Manifest — `hello-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-<username>
spec:
  containers:
  - name: mypod
    image: ubuntu
    resources:
      limits:
        memory: 100Mi
        cpu: 100m
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sh", "-c", "echo 'Hello from NRP!' && sleep 3600"]
```

**Key fields:** `apiVersion` and `kind` identify the object; `metadata.name` names it; `spec.containers` defines the image, resources, and startup command.

- `requests` influence scheduling and resource allocation.
- `limits` cap usage: CPU can be throttled; exceeding the memory limit can cause an out-of-memory termination.
- `100m` CPU = 0.1 CPU core; `100Mi` memory = 100 mebibytes.

**Command inside the container:**

```bash
echo 'Hello from NRP!' && sleep 3600
```

### Submit, inspect, and clean up

```bash
kubectl apply -f hello-pod.yaml -n <namespace>
kubectl get pods -n <namespace>
kubectl logs test-pod-<username> -n <namespace>
kubectl describe pod test-pod-<username> -n <namespace>
kubectl delete pod test-pod-<username> -n <namespace>
```

**Expected behavior:** logs show `Hello from NRP!`; the Pod stays running during the sleep. Delete it when inspection is complete.

## 4. Job: a single batch task

**Summary:** a Job tracks finite work until the required successful completion is reached. With `restartPolicy: Never`, failed containers are not restarted inside the same Pod; the Job controller can create replacement Pods subject to its retry limit.

### Manifest — `simple-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: simple-job-<username>
spec:
  template:
    spec:
      containers:
      - name: simple-container
        image: ubuntu
        command: ["sh", "-c", "echo 'Hello from NRP Job!' && date && exit 0"]
        resources:
          limits:
            memory: 100Mi
            cpu: 100m
          requests:
            memory: 100Mi
            cpu: 100m
      restartPolicy: Never
  backoffLimit: 1
  ttlSecondsAfterFinished: 300
```

- `backoffLimit: 1` sets a small retry budget before the Job is marked failed.
- `ttlSecondsAfterFinished: 300` makes a finished Job eligible for automatic cleanup after five minutes, including its dependent Pods. Collect logs promptly.

**Command inside the container:**

```bash
echo 'Hello from NRP Job!' && date && exit 0
```

### Submit, inspect, and clean up

```bash
kubectl apply -f simple-job.yaml -n <namespace>
kubectl get jobs -n <namespace>
kubectl get pods -l job-name=simple-job-<username> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl describe job simple-job-<username> -n <namespace>
# Optional manual cleanup before the TTL controller removes the Job
kubectl delete job simple-job-<username> -n <namespace>
```

**Expected behavior:** the container prints a greeting and date, then exits successfully. The Job records one successful completion.

## 5. Job: multiple Pods in parallel

**Summary:** `completions` sets the total successful completions needed; `parallelism` sets the desired maximum number of concurrently active Pods. Pods can remain Pending when resources are unavailable.

### Manifest — `parallel-job.yaml`

This version uses a separate Job name to avoid conflicting with the single-task example.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job-<username>
spec:
  # --- Add these two lines to submit 5 pods ---
  completions: 5
  parallelism: 5
  # ---------------------------
  template:
    spec:
      containers:
      - name: simple-container
        image: ubuntu
        command: ["sh", "-c", "echo 'Hello from NRP Job!' && date && exit 0"]
        resources:
          limits:
            memory: 100Mi
            cpu: 100m
          requests:
            memory: 100Mi
            cpu: 100m
      restartPolicy: Never
  backoffLimit: 1
  ttlSecondsAfterFinished: 300
```

Each Pod runs the same greeting-and-date command. This example repeats identical work; it does not partition an input dataset.

### Submit, inspect, and clean up

```bash
kubectl apply -f parallel-job.yaml -n <namespace>
kubectl get jobs -n <namespace>
kubectl get pods -l job-name=parallel-job-<username> -n <namespace>
# Repeat for each Pod whose output you want to inspect
kubectl logs <pod-name> -n <namespace>
kubectl delete job parallel-job-<username> -n <namespace>
```

**Expected behavior:** the Job seeks five successful completions, with up to five Pods active concurrently. Cleanup eligibility starts five minutes after it finishes.

## 6. Deployments: long-running applications

**Summary:** Deployments maintain a desired number of replicas and support rolling application updates. Use them for services rather than finite batch tasks.

The source notes introduce Deployments but include no manifest or command exercise. A full Deployment walkthrough is outside this session’s scope.

## 7. Persistent storage with a PVC

**Summary:** a container’s writable filesystem on the compute node is ephemeral and should not be used to preserve results. A PVC requests persistent storage that can remain after its consuming Pod is deleted.

### Storage manifest — `cephfs-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc-<username>
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
```

- `storage: 1Gi` requests one gibibyte of storage.
- `ReadWriteMany` permits read/write mounts from multiple nodes, allowing multiple Pods to share the volume. Applications must still coordinate concurrent writes.
- `rook-cephfs` is the storage class used for object storage on the NRP.


### Consumer manifest — `cephfs-pod.yaml`

The example mounts the PVC at `/shared-data`, writes a file, prints it, and sleeps for inspection. CPU/memory requests and limits were added for consistency with the earlier examples. Double quotes replace the original single quotes so `$(date)` expands to a timestamp instead of being written literally.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs-pod-<username>
spec:
  containers:
  - name: cephfs-container
    image: ubuntu
    command: ["sh", "-c", "echo \"Data written at $(date)\" > /shared-data/test.txt && cat /shared-data/test.txt && sleep 3600"]
    resources:
      requests:
        memory: 100Mi
        cpu: 100m
      limits:
        memory: 100Mi
        cpu: 100m
    volumeMounts:
    - name: cephfs-volume
      mountPath: /shared-data
  volumes:
  - name: cephfs-volume
    persistentVolumeClaim:
      claimName: cephfs-pvc-<username>
```

**Commands executed inside the container:**

```bash
echo "Data written at $(date)" > /shared-data/test.txt
cat /shared-data/test.txt
sleep 3600
```

### Create storage, run the Pod, and inspect output

```bash
kubectl apply -f cephfs-pvc.yaml -n <namespace>
kubectl get pvc cephfs-pvc-<username> -n <namespace>
# Check that the claim is Bound before continuing
kubectl apply -f cephfs-pod.yaml -n <namespace>
kubectl get pods -n <namespace>
kubectl logs cephfs-pod-<username> -n <namespace>
kubectl exec cephfs-pod-<username> -n <namespace> -- cat /shared-data/test.txt
kubectl delete pod cephfs-pod-<username> -n <namespace>
```

**Expected behavior:** the PVC binds, the Pod prints the timestamped file contents, and deleting the Pod leaves the PVC in place.

**Data safety:** the example uses `>` and overwrites `test.txt` on each run. Keep the PVC if you need the results. Only remove the claim after backing up data and checking the storage retention policy; deleting it may delete the underlying data.

```bash
# Destructive storage cleanup — run only when the data is no longer needed
kubectl delete pvc cephfs-pvc-<username> -n <namespace>
```

## 8. Command quick reference

| Action | Command pattern |
| --- | --- |
| Submit a manifest | `kubectl apply -f <file.yaml> -n <namespace>` |
| List Pods | `kubectl get pods -n <namespace>` |
| List Jobs | `kubectl get jobs -n <namespace>` |
| Find a Job’s Pods | `kubectl get pods -l job-name=<job-name> -n <namespace>` |
| Read output | `kubectl logs <pod-name> -n <namespace>` |
| Inspect scheduling/events | `kubectl describe pod <pod-name> -n <namespace>` |
| Check storage | `kubectl get pvc -n <namespace>` |
| Read a file in a running container | `kubectl exec <pod-name> -n <namespace> -- cat /shared-data/test.txt` |
| Remove compute resources | `kubectl delete pod <pod-name> -n <namespace>` or `kubectl delete job <job-name> -n <namespace>` |

## 9. Troubleshooting checklist

- Confirm authentication, namespace membership, and local tool setup.
- Replace placeholders and save each YAML manifest separately.
- Submit the Hello World Pod and inspect its logs.
- Run the single and parallel Job examples; collect logs before automatic cleanup.
- Create the PVC and storage Pod; inspect the written file.
- Remove unneeded compute resources; retain storage needed for results.
- Record actual commands, outputs, and errors separately if documenting a completed session.
