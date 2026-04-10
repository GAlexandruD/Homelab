# Homelab

My personal k8s cluster running on a single bare-metal node, managed entirely through GitOps with Flux CD. Everything (apps, infrastructure, secret references) is declarative so if the node gets wiped, `flux bootstrap` and a few manual steps will bring it back.

## What's running

| App | Purpose |
|-----|---------|
| Audiobookshelf | Audiobook and podcast server |
| Commafeed | Self-hosted RSS reader |
| Homepage | Cluster dashboard |
| Linkding | Bookmark manager |
| Metabase | Analytics and BI |
| n8n | Workflow automation |
| Transmission | BitTorrent client |
| Uptime Kuma | Uptime monitoring |
| Quartz | Personal blog |
| Wallabag | Read-it-later |

Plus kube-prometheus-stack for monitoring (Prometheus + Grafana) and Renovate running as an in-cluster CronJob for dependency updates.

## Stack

- **Kubernetes** - k3s, two single-node clusters:
  - `adstage` — Ubuntu 22.04, main workload cluster
  - `maxipi` — Ubuntu 24.04, also serves as NFS server; uses Calico for CNI + NetworkPolicy
- **GitOps** - Flux CD v2 with Kustomize base/overlay pattern
- **Ingress** - Traefik + Cloudflare Tunnels, no ports exposed on the host
- **Secrets** - SOPS/Age for existing apps; External Secrets Operator + Infisical for new apps
- **Storage** - CSI Driver NFS on maxipi; StorageClass `nfs-csi` backed by `/mnt/WD1TB/nfs/k8s` on the node
- **Monitoring** - kube-prometheus-stack (Prometheus + Grafana); alerting beyond kube-state-metrics defaults is still a work in progress

## Repo layout

```
apps/
  _base/<app>/       # Deployment, Service, PVC
  adstage/<app>/     # Cloudflare tunnel, ingress, secrets
infrastructure/
  controllers/       # Helm chart installations (ESO, etc.)
  configs/           # CRD-dependent resources (ClusterSecretStore, etc.)
  daemonsets/        # Node-level housekeeping
monitoring/          # kube-prometheus-stack HelmRelease
clusters/adstage/    # Flux entry point - Kustomization objects
```

Every app follows the same base/overlay split. `_base` has the Kubernetes resources. `adstage` has environment-specific config: Cloudflare tunnel credentials, ingress rules, and secret references.

## Secrets

- I started with SOPS-encrypted secrets committed to git, decrypted by Flux at apply time via an Age key.
- New apps pull secrets from Infisical through ESO. One `ClusterSecretStore` pointing at the Infisical project, and per-app `ExternalSecret` resources referencing paths like `/metabase/METABASE_DB_PASSWORD`.

The only secret that lives in the cluster as a manually applied Kubernetes Secret is the Infisical machine identity credential. Everything else is either encrypted in git or fetched from Infisical at runtime.

## Security hardening

All deployments run with:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop:
      - ALL
```

Where a container needs to write to the filesystem (JVM tmp dirs, PostgreSQL sockets, app caches), an `emptyDir` volume covers just that path rather than making the whole root writable. PostgreSQL deployments always get emptyDirs at `/tmp` and `/var/run/postgresql`.

Some images are incompatible with this because their entrypoints expect to run as root. Wallabag and the Transmission linuxserver image fall into this bucket. Those get `seccompProfile: RuntimeDefault` only, until rootless alternatives exist.

## Adding an app

1. Create `apps/_base/<app>/` - namespace, deployment, service, PVC
2. Create `apps/adstage/<app>/` - `cloudflare.yaml`, `ingress.yaml`, `externalsecret.yaml`, `configmap.yaml`
3. Register in `apps/adstage/kustomization.yaml`
4. Create a Cloudflare tunnel, store the credentials JSON in Infisical as `CF_TUNNEL_<APP>_CREDENTIALS`
5. Add app secrets to Infisical under `/<app>/`

## Adding a Helm chart

Charts that install CRDs need two separate Flux Kustomizations with a `dependsOn` between them. One for the HelmRelease (which creates the CRDs) and one for any CRD-dependent resources like `ClusterSecretStore`. Putting both in the same Kustomization causes dry-run failures because Flux validates all resources in a single pass before any are applied.

## Node configuration

### inotify limits

Quartz (and Flux itself) use fsnotify to watch files. On a node running many pods the default inotify limits are too low and builds fail with `failed to create fsnotify watcher: too many open files`. Fix:

```bash
cat >> /etc/sysctl.d/99-inotify.conf <<EOF
fs.inotify.max_user_instances=512
fs.inotify.max_user_watches=524288
EOF
sysctl -p /etc/sysctl.d/99-inotify.conf
```

### Disk space — container image pruning

k3s does not garbage-collect unused container images automatically. Over time pulled images accumulate and fill the disk. `infrastructure/daemonsets/` runs a DaemonSet (`crictl-prune-daemonset`) on every k3s node that calls `crictl rmi --prune` once every 24 hours to remove images not referenced by any container. It connects to the k3s containerd socket at `/run/k3s/containerd/containerd.sock`. Failures are logged to stderr but do not restart the pod — check with `kubectl logs -n kube-system <crictl-pod>` if you suspect a prune is not running.

### NFS server (maxipi)

maxipi doubles as the NFS server for its own cluster. One-time host setup:

```bash
apt install nfs-kernel-server
mkdir -p /mnt/WD1TB/nfs/k8s
chown nobody:nogroup /mnt/WD1TB/nfs/k8s
chmod 755 /mnt/WD1TB/nfs/k8s
echo '/mnt/WD1TB/nfs/k8s 192.168.1.14(rw,sync,no_subtree_check,no_root_squash) 10.42.0.0/16(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports
exportfs -rav
systemctl enable --now nfs-server
```

The CSI Driver NFS Helm chart is installed via Flux (`infrastructure/controllers/maxipi/nfs-csi/`). The `StorageClass` (`nfs-csi`, set as default) lives in `infrastructure/configs/maxipi/nfs-csi/storageclass.yaml`, pointing at `192.168.1.14`.

### Grafana PVC ownership (on cluster recreation)

`initChownData` is disabled to avoid running a root init container on every pod start. On a fresh cluster, the local-path provisioner creates the Grafana PVC directory owned by root, which Grafana (UID 472) cannot write to. Fix it once before Grafana starts:

```bash
# Scale down so Grafana doesn't start
kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=0

# Find the PVC path on the node
PV=$(kubectl get pvc -n monitoring kube-prometheus-stack-grafana -o jsonpath='{.spec.volumeName}')
PV_PATH=$(kubectl get pv $PV -o jsonpath='{.spec.local.path}')  # for adstage
PV_PATH=$(kubectl get pv $PV -o jsonpath='{.spec.hostPath.path}')  # for maxipi
chown -R 472:472 $PV_PATH

# Scale back up
kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=1
```

## TODO:

- Backups
- Add NetworkPolicies
- Grafana Alerts
- PostgreSQL runs as `Deployment` rather than `StatefulSet`. It works, but loses ordered rollout guarantees and stable network identity.

