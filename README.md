# Homelab

My personal k8s homelab running two clusters, managed entirely through GitOps with Flux CD. Everything (apps, infrastructure, secret references) is declarative so if a node gets wiped, `flux bootstrap` and a few manual steps will bring it back.

## What's running

### adstage (staging)

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

Plus kube-prometheus-stack for monitoring (Prometheus + Grafana) and Renovate running as an in-cluster CronJob for dependency updates.

### Not currently deployed

Apps with full manifests in the repo but not active on any cluster:

| App | Purpose | Notes |
|-----|---------|-------|
| Wallabag | Read-it-later | Requires root entrypoint; kept as reference |

### maxipi (production)

| App | Purpose |
|-----|---------|
| Transmission | BitTorrent client |

Plus kube-prometheus-stack for monitoring (Prometheus + Grafana).

## Stack

- **Kubernetes** - k3s, two single-node clusters:
  - `adstage` - Ubuntu on AMD A4-6210, staging cluster for testing new images before promoting to production
  - `maxipi` - Ubuntu on Celeron J3455, production cluster; also serves as NFS server; uses Calico for CNI + NetworkPolicy
- **GitOps** - Flux CD v2 with Kustomize base/overlay pattern
- **Ingress** - Traefik + Cloudflare Tunnels, no ports exposed on the host (adstage); hostPort for local network access (maxipi)
- **Secrets** - SOPS/Age for existing apps; External Secrets Operator + Infisical for new apps
- **Storage** - unified `standard` StorageClass across both clusters: `rancher.io/local-path` on adstage, `nfs.csi.k8s.io` (NFSv3) on maxipi backed by `/mnt/WD1TB/nfs/k8s`; `nfs-csi` (NFSv4.1) available on maxipi for images without chown requirements
- **Monitoring** - kube-prometheus-stack (Prometheus + Grafana) on both clusters

## Repo layout

```
apps/
  _base/<app>/       # Deployment, Service, PVC
  adstage/<app>/     # Cloudflare tunnel, ingress, secrets
  maxipi/<app>/      # hostPort, cluster-specific patches
infrastructure/
  controllers/
    base/<chart>/    # HelmRepository + HelmRelease (adstage shared base)
    adstage/<chart>/ # adstage overlay
    maxipi/<chart>/  # maxipi-specific controllers (calico, nfs-csi, eso)
  configs/
    adstage/<chart>/ # CRD-dependent resources (ClusterSecretStore, StorageClass, etc.)
    maxipi/<chart>/  # maxipi CRD-dependent resources
  daemonsets/        # Node-level housekeeping
monitoring/          # kube-prometheus-stack HelmRelease
clusters/adstage/    # Flux entry point - Kustomization objects
clusters/maxipi/     # Flux entry point - Kustomization objects
```

Every app follows the same base/overlay split. `_base` has the Kubernetes resources. Cluster overlays have environment-specific config: Cloudflare tunnel credentials, ingress rules, secret references, and cluster-specific patches.

## Secrets

- Existing apps use SOPS-encrypted secrets committed to git, decrypted by Flux at apply time via an Age key.
- New apps pull secrets from Infisical through ESO. One `ClusterSecretStore` pointing at the Infisical project per cluster, and per-app `ExternalSecret` resources referencing paths like `/metabase/METABASE_DB_PASSWORD`.

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

Some images are incompatible with this because their entrypoints expect to run as root. The Transmission linuxserver image falls into this bucket and gets `seccompProfile: RuntimeDefault` only, until a rootless alternative exists.

## Adding an app

1. Create `apps/_base/<app>/` - namespace, deployment, service, PVC (use `storageClassName: standard`)
2. Create `apps/adstage/<app>/` - `cloudflare.yaml`, `ingress.yaml`, `externalsecret.yaml`, `configmap.yaml`
3. Register in `apps/adstage/kustomization.yaml`
4. Create a Cloudflare tunnel, store the credentials JSON in Infisical as `CF_TUNNEL_<APP>_CREDENTIALS`
5. Add app secrets to Infisical under `/<app>/`

For maxipi, create `apps/maxipi/<app>/` with cluster-specific patches (hostPort, storage overrides if needed) and register in `apps/maxipi/kustomization.yaml`.

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

### Disk space - container image pruning

k3s does not garbage-collect unused container images automatically. Over time pulled images accumulate and fill the disk. `infrastructure/daemonsets/` runs a DaemonSet (`crictl-prune-daemonset`) on every k3s node that calls `crictl rmi --prune` once every 24 hours to remove images not referenced by any container. It connects to the k3s containerd socket at `/run/k3s/containerd/containerd.sock`. Failures are logged to stderr but do not restart the pod - check with `kubectl logs -n kube-system <crictl-pod>` if you suspect a prune is not running.

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

The CSI Driver NFS Helm chart is installed via Flux (`infrastructure/controllers/maxipi/nfs-csi/`). Two StorageClasses are available:
- `standard` - NFSv3, used by default; compatible with linuxserver images that run `chown` at startup
- `nfs-csi` - NFSv4.1, for images without chown requirements that benefit from NFSv4.1 features

### Grafana PVC ownership on adstage (cluster recreation)

`initChownData` is disabled to avoid running a root init container on every pod start. On a fresh adstage cluster, the local-path provisioner creates the Grafana PVC directory owned by root, which Grafana (UID 472) cannot write to. Fix it once before Grafana starts:

```bash
# Scale down so Grafana doesn't start while you work
kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=0

# Find the PVC path on the node
PV=$(kubectl get pvc -n monitoring kube-prometheus-stack-grafana -o jsonpath='{.spec.volumeName}')
PV_PATH=$(kubectl get pv $PV -o jsonpath='{.spec.local.path}')

chown -R 472:472 $PV_PATH

# Scale back up
kubectl scale deployment -n monitoring kube-prometheus-stack-grafana --replicas=1
```

On maxipi, Grafana uses NFS storage and this issue does not apply.

## TODO:

- Backups
- Add NetworkPolicies
- Grafana Alerts
- PostgreSQL runs as `Deployment` rather than `StatefulSet`. It works, but loses ordered rollout guarantees and stable network identity.
