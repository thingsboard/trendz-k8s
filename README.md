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
| `trendz-app-shared-pvc.yml` | `/data` of all Trendz pods: license instance file, heap dumps (ReadWriteMany) |
| `trendz-app-deployment.yml` | Trendz Deployment and Service |
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
3. `trendz-app-deployment.yml`: `TB_API_URL` (the ThingsBoard service as seen from the `trendz` namespace) and the
   license values.
4. `trendz-app-shared-pvc.yml`: `storageClassName` of a ReadWriteMany class of your cluster (NFS, EFS, Azure Files,
   CephFS, ...). A ReadWriteOnce volume does not work: a rolling update starts the new pod before the old one stops,
   and replicas may run on different nodes.

## Install

```bash
kubectl apply -f trendz-namespace.yml
kubectl apply -f trendz-db-app-secret.yml -f trendz-python-executor-secret.yml
kubectl apply -f trendz-app-env-config.yml -f trendz-app-files-config.yml
kubectl apply -f trendz-python-executor-env-config.yml -f trendz-python-executor-files-config.yml
kubectl apply -f trendz-app-shared-pvc.yml
kubectl apply -f trendz-cache-deployment.yml
kubectl apply -f trendz-python-executor-deployment.yml -f trendz-python-executor-network-policy.yml
kubectl apply -f trendz-app-deployment.yml
kubectl apply -f trendz-ingress.yml
```

## Scaling

Raise `replicas` of `trendz-app-deployment.yml` and of `trendz-python-executor-deployment.yml` together: one executor
per Trendz pod. Each Trendz pod sends at most `SCRIPT_ENGINE_MAX_CONNECTIONS` (2) Python scripts at once, and the
executor memory (2500M, `PYTHON_SCRIPTS_MAX_TOTAL_MEMORY_MB` 1500) is sized for that. Raise both memory values together
if you raise the script count. Valkey stays at one replica.

Per pod: Trendz 1600m CPU / 3500M memory (heap 2000m), executor 1600m / 2500M (heap 384m), Valkey 200m / 300M.

## Upgrading from the previous version of these files

The file and object names changed. After applying the new files as in "Install":

- `/data` moved from the ReadWriteOnce `trendz-app-pvc` to `trendz-app-shared-pvc`. Copy `license.data` from the old
  volume to the new one before Trendz starts (or activate the license again), then delete `trendz-app-pvc`.
- Delete the ConfigMaps the new files replace:
  ```bash
  kubectl -n trendz delete configmap trendz-app-config trendz-app-db-config trendz-python-executor-config
  ```
- The database settings moved from the ConfigMap `trendz-app-db-config` to the Secret `trendz-db-app-secret`.
- Trendz and the executor run as user 799 (no root). Volumes are given to group 799 on mount (`fsGroup`).
