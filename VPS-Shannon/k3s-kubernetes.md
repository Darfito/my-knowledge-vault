# k3s & Kubernetes — VPS Shannon

## Overview

This VPS runs **k3s** (lightweight Kubernetes) for the Gunawan production stack.

- **Traefik** as ingress controller (built into k3s)
- **cert-manager** for automatic TLS via Let's Encrypt
- All services are in the **`gunawan`** namespace

## k3s Status

k3s is currently **disabled** (does not auto-start on reboot). Last active: 15 April 2026.

### Start / Stop / Status

```bash
# As root:
systemctl start k3s      # start
systemctl stop k3s       # stop
systemctl status k3s     # check status
systemctl enable k3s     # enable auto-start on reboot
systemctl disable k3s    # disable auto-start
```

### After starting k3s, wait ~30 seconds then check:

```bash
kubectl get pods --all-namespaces
```

---

## k9s — Terminal UI for Kubernetes

k9s is installed via **Linuxbrew** at `/home/linuxbrew/.linuxbrew/bin/k9s`.

**Must be run as user `santoso`** — not root.

```bash
su - santoso
k9s
```

If not found from root:
```bash
/home/linuxbrew/.linuxbrew/bin/k9s
```

### Useful k9s shortcuts

| Key | Action |
|-----|--------|
| `:pod` | List all pods |
| `:deploy` | List deployments |
| `:ing` | List ingress |
| `l` | View pod logs |
| `d` | Describe resource |
| `ctrl+d` | Delete resource |
| `q` | Quit / go back |

---

## Namespace & Manifests

All manifests are at:
```
/home/santoso/gunawan-agents/santoso-protocol/k8s/manifests/
```

| File | Contents |
|------|----------|
| `00-namespace.yaml` | `gunawan` namespace |
| `10-secrets.yaml.tmpl` | Secrets template (API keys) |
| `20-configmap.yaml` | Config — PAPERCLIP_API_URL etc. |
| `30-paperclip-statefulset.yaml` | Paperclip StatefulSet |
| `31-paperclip-service.yaml` | Paperclip service (ClusterIP:3100) |
| `40-bridge-deployment.yaml` | Telegram bridge Deployment |
| `41-bridge-service.yaml` | Telegram bridge service (ClusterIP:3200) |
| `50-ingress.yaml` | Ingress + domain routing |
| `60-cert-manager-issuer.yaml` | Let's Encrypt ClusterIssuer |

### Apply all manifests:

```bash
# As santoso (k3s must be running first):
kubectl apply -f ~/gunawan-agents/santoso-protocol/k8s/manifests/
```
