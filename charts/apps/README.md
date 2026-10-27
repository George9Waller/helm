# App to create all my apps
The common chart for each deployed app from another repo.

The individual app specific charts live in the apps own repo. This parent helm repo simply defines them so they are deployed and synced automatically.

# `data-nvme` storage (local-path on ubuntuserver2)
`templates/local-path-data.yaml` deploys a **second** local-path provisioner as the
`local-path-data` Argo CD Application, separate from the one k3s bundles in
`kube-system`. It backs a non-default StorageClass `data-nvme` whose volumes live
under `/mnt/data/k3s-storage` on the NVMe disk on `ubuntuserver2` only.

The bundled provisioner (StorageClass `local-path`, provisioner
`rancher.io/local-path`) is untouched and stays the cluster default.

## Using it from a workload
1. Create a PVC with `storageClassName: data-nvme`.
2. The consuming pod **must** set `nodeSelector: {size: large}` so it schedules on
   `ubuntuserver2`. `nodePathMap` lists only that node (no
   `DEFAULT_PATH_FOR_NON_LISTED_NODES`), so a pod scheduled on `ubuntuserver`
   would have no backing path and the PVC would stay `Pending`
   (`volumeBindingMode: WaitForFirstConsumer`).

Example PVC:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: data-nvme
  resources:
    requests:
      storage: 10Gi
```
Example pod snippet:
```yaml
spec:
  nodeSelector:
    size: large
  # ... volumes[].persistentVolumeClaim.claimName: my-data
```

A ready-to-apply test manifest lives in `examples/local-path-data/` (not synced by
Argo CD; apply it by hand).

# Node placement across the two nodes
The cluster has a small control-plane node (`ubuntuserver`, 8GB) and a large
worker (`ubuntuserver2`, 32GB, labelled `size=large`). A few things about how the
scheduler behaves here are worth knowing, because the default behaviour can
quietly overload the small node.

**The scheduler places pods on their resource _requests_, not actual usage.** At
creation time it filters out nodes that can't fit the pod's requests (and ones
excluded by `nodeSelector`/taints), then prefers the node with the most
unrequested capacity — which naturally leans towards `ubuntuserver2`. But:

- **A pod with no `requests` counts as costing nothing**, so the scheduler will
  happily pile request-less pods onto `ubuntuserver` even when it's actually at
  high memory. Many charts ship without requests, so this is the usual culprit.
- **Placement only happens at creation.** Nothing rebalances on its own — adding
  `ubuntuserver2` did not move existing pods, and real load growth is invisible to
  the scheduler (the only backstop is the kubelet evicting under memory pressure).

## What to do
1. **Set `requests` on everything, and memory `limits` on the heavy ones.** Base
   them on real usage (`kubectl top pods -A`). A memory limit means a runaway pod
   is OOM-killed and restarted rather than taking the node down. CPU limits are
   often omitted deliberately (CPU throttles rather than fails).
   ```yaml
   resources:
     requests: { cpu: 100m, memory: 256Mi }
     limits:   { memory: 512Mi }
   ```
2. **Add a `LimitRange` per namespace** so anything you miss still gets a sensible
   default request and doesn't look "free" to the scheduler:
   ```yaml
   apiVersion: v1
   kind: LimitRange
   metadata:
     name: defaults
   spec:
     limits:
       - type: Container
         defaultRequest: { cpu: 50m, memory: 128Mi }
         default:        { memory: 512Mi }
   ```
3. **Keep the small node light with a soft taint** rather than adding affinity to
   every app. `PreferNoSchedule` (not `NoSchedule`) means pods avoid `ubuntuserver`
   unless `ubuntuserver2` is full or down, so nothing breaks on failover and the
   control plane / Argo CD keep running there:
   ```sh
   kubectl taint node ubuntuserver dedicated=control-plane:PreferNoSchedule
   ```
4. **Optional: rebalancing.** The Kubernetes
   [`descheduler`](https://github.com/kubernetes-sigs/descheduler) can redistribute
   pods over time (e.g. after `ubuntuserver2` has been down and everything piled
   onto the small node). Add it only _after_ requests are in place — it decides
   from requests too.

After adding the taint, `kubectl rollout restart` each deployment to move existing
pods onto the large node. Heavy workloads that must land on `ubuntuserver2` (like
anything using the `data-nvme` StorageClass above) should still pin themselves with
`nodeSelector: {size: large}` — the taint only expresses a _preference_.

# Create the `ecr-registry-secret`
```
kubectl create secret docker-registry ecr-registry-secret \
  --docker-server=<ecr-registry> \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region eu-west-2) \
  --docker-email=<email>
```

# External secrets
https://external-secrets.io/latest/introduction/getting-started/

## Add repo to helm on cluster machine
`helm repo add external-secrets https://charts.external-secrets.io`

## Create AWS secrets credentials secret
```
echo -n 'KEYID' > ./access-key
echo -n 'SECRETKEY' > ./secret-access-key
kubectl create secret generic awssm-secret --from-file=./access-key --from-file=./secret-access-key
```
