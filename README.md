# Trendz on your own Kubernetes

Manifests for running Trendz next to an existing ThingsBoard in any Kubernetes cluster: Trendz, the Python executor
and Valkey (cache, locks and rate limiter shared by the Trendz replicas). No monitoring, no cloud-specific parts.
The PostgreSQL database is yours: these files only point Trendz at it.

| File | What it is |
|---|---|
| `trendz-namespace.yml` | namespace `trendz`; its default service account mounts no API token |
| `trendz-db-app-secret.yml` | database URL, user and password |
| `trendz-python-executor-secret.yml` | shared secret of the Trendz ↔ executor channel |
| `trendz-app-env-config.yml` | Trendz environment: heap, load limits, Valkey, executor address |
| `trendz-app-files-config.yml` | `trendz.conf` (JVM flags) and `logback.xml` mounted into Trendz |
| `trendz-app-shared-pvc.yml` | `/data` of all Trendz pods: one license file per pod, heap dumps (ReadWriteMany) |
| `trendz-app-statefulset.yml` | Trendz StatefulSet, its Service and the StatefulSet's headless Service |
| `trendz-cache-deployment.yml` | Valkey: config, Deployment, Service, NetworkPolicy |
| `trendz-python-executor-env-config.yml` | executor environment: heap, memory guard, message size |
| `trendz-python-executor-files-config.yml` | Python `requirements.txt`, JVM flags and `logback.xml` of the executor |
| `trendz-python-executor-deployment.yml` | executor Deployment and Service |
| `trendz-python-executor-network-policy.yml` | only Trendz may call the executor; the executor may call nothing |
| `trendz-ingress.yml` | NGINX ingress for `/trendz` and `/apiTrendz`, `/trendz/actuator` refused |

## Before applying

1. `trendz-db-app-secret.yml`: the real database connection (or create the Secret with `kubectl`, see the file).
2. `trendz-python-executor-secret.yml`: replace `placeholder` with a long random value, for example
   `openssl rand -hex 32`. Trendz and the executor read the same Secret.
3. `trendz-app-statefulset.yml`: `TB_API_URL` (the ThingsBoard service as seen from the `trendz` namespace) and the
   license values.
4. `trendz-app-shared-pvc.yml`: `storageClassName` of a ReadWriteMany class of your cluster (NFS, EFS, Azure Files,
   CephFS, ...). A ReadWriteOnce volume does not work: replicas may run on different nodes.

## Install

```bash
kubectl apply -f trendz-namespace.yml
kubectl apply -f trendz-db-app-secret.yml -f trendz-python-executor-secret.yml
kubectl apply -f trendz-app-env-config.yml -f trendz-app-files-config.yml
kubectl apply -f trendz-python-executor-env-config.yml -f trendz-python-executor-files-config.yml
kubectl apply -f trendz-app-shared-pvc.yml
kubectl apply -f trendz-cache-deployment.yml
kubectl apply -f trendz-python-executor-deployment.yml -f trendz-python-executor-network-policy.yml
kubectl apply -f trendz-app-statefulset.yml
kubectl apply -f trendz-ingress.yml
```

## Scaling

Raise `replicas` of `trendz-app-statefulset.yml` and of `trendz-python-executor-deployment.yml` together: one executor
per Trendz pod. Each Trendz pod sends at most `SCRIPT_ENGINE_MAX_CONNECTIONS` (2) Python scripts at once, and the
executor memory (2500M, `PYTHON_SCRIPTS_MAX_TOTAL_MEMORY_MB` 1500) is sized for that. Raise both memory values together
if you raise the script count. Valkey stays at one replica.

Trendz runs as a StatefulSet: each pod keeps a stable name (`trendz-app-0`, `trendz-app-1`, ...) and its own
file in `/data`, `license-<pod name>.data`. Pods must never share this file.

A rollout restarts the Trendz pods one at a time: with one replica Trendz is unavailable while its pod restarts.

Per pod: Trendz 1600m CPU / 3500M memory (heap 2000m), executor 1600m / 2500M (heap 384m), Valkey 200m / 300M.

## Upgrading from the previous version of these files

The file and object names changed. Trendz moved from the Deployment `trendz-app-deployment` to the StatefulSet
`trendz-app`, and the license file is now per pod: `license.data` becomes `license-trendz-app-0.data`. Stop the old
pods and move the file before the StatefulSet starts.

1. Apply the new files as in "Install", except `trendz-app-statefulset.yml`.
2. Stop the old Trendz pods:
   ```bash
   kubectl -n trendz delete deployment trendz-app-deployment
   kubectl -n trendz wait --for=delete pod -l app=trendz-app-pod-label --timeout=180s
   ```
3. Move the license file. If `/data` is already `trendz-app-shared-pvc`:
   ```bash
   kubectl -n trendz run trendz-license-move --rm -i --restart=Never --image=busybox:1.37 --overrides='{"spec":{"securityContext":{"runAsUser":799,"runAsGroup":799},"containers":[{"name":"move","image":"busybox:1.37","command":["sh","-c","cd /data && if [ -f license.data ] && [ ! -f license-trendz-app-0.data ]; then mv license.data license-trendz-app-0.data; fi; ls -ln"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"trendz-app-shared-pvc"}}]}}'
   ```
   If `/data` is still the ReadWriteOnce `trendz-app-pvc`, copy the file to the new volume instead, then delete
   `trendz-app-pvc`:
   ```bash
   kubectl -n trendz run trendz-license-move --rm -i --restart=Never --image=busybox:1.37 --overrides='{"spec":{"containers":[{"name":"move","image":"busybox:1.37","command":["sh","-c","if [ -f /old/license.data ] && [ ! -f /data/license-trendz-app-0.data ]; then cp /old/license.data /data/license-trendz-app-0.data && chown 799:799 /data/license-trendz-app-0.data; fi; ls -ln /data"],"volumeMounts":[{"name":"old","mountPath":"/old"},{"name":"data","mountPath":"/data"}]}],"volumes":[{"name":"old","persistentVolumeClaim":{"claimName":"trendz-app-pvc"}},{"name":"data","persistentVolumeClaim":{"claimName":"trendz-app-shared-pvc"}}]}}'
   ```
   Without `license.data` nothing is moved.
4. Start Trendz:
   ```bash
   kubectl apply -f trendz-app-statefulset.yml
   ```

Also:

- Delete the ConfigMaps the new files replace:
  ```bash
  kubectl -n trendz delete configmap trendz-app-config trendz-app-db-config trendz-python-executor-config
  ```
- The database settings moved from the ConfigMap `trendz-app-db-config` to the Secret `trendz-db-app-secret`.
- Trendz and the executor run as user 799 (no root). Volumes are given to group 799 on mount (`fsGroup`).
