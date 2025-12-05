# Homelab

My starting Kubernetes cluster.

## 2025-09-26

Free disk space on cluster was getting low. I have configured the `crictl-prune-daemonset` to run `crictl rmi --prune` everyday.
`rmi` - Remove one or more images
`--prune` - Remove all unused images
In order for k3s to work, I added the configuration file `/etc/crictl.yaml`:

```sh
echo 'runtime-endpoint: unix:///run/k3s/containerd/containerd.sock
image-endpoint: unix:///run/k3s/containerd/containerd.sock
timeout: 10
debug: true' > /etc/crictl.yaml
```
