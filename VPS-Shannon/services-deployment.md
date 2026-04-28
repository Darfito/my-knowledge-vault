# Services & Deployment — VPS Shannon

## Production Stack (via k3s)

All production services run in the Kubernetes namespace `gunawan`.

---

### 1. Paperclip (AI Platform)

| Item | Value |
|------|-------|
| Kind | StatefulSet |
| Image | `docker.io/iannn07/paperclip-gunawan:2026.403.0-patched` |
| Port | 3100 |
| Domain | `https://gunawan-paperclip.digital-lab.ai` |
| Data dir | PersistentVolume at `/home/node/.paperclip` |

**Claude Auth**: Mounts hostPath from `/home/santoso/.claude` → `/home/node/.claude`
The paperclip pod uses santoso's Claude OAuth session (not an API key).

**Resource limits**:
- CPU: 300m request / 2000m limit
- Memory: 1536Mi request / 4Gi limit

---

### 2. Telegram Bridge

| Item | Value |
|------|-------|
| Kind | Deployment |
| Image | `docker.io/iannn07/santoso-telegram-bridge:latest` |
| Port | 3200 |
| Domain | `https://gunawan-bridge.digital-lab.ai` |

---

### Traefik Ingress

Traefik is the ingress controller built into k3s v3.6.10.
Ingress config is at `/home/santoso/gunawan-agents/santoso-protocol/k8s/manifests/50-ingress.yaml`.

```yaml
# Domain routing:
gunawan-paperclip.digital-lab.ai  →  service/paperclip:3100
gunawan-bridge.digital-lab.ai     →  service/telegram-bridge:3200

# TLS: cert-manager + Let's Encrypt (letsencrypt-prod)
# Secret: gunawan-tls
```

k3s Traefik manifest is at:
```
/var/lib/rancher/k3s/server/manifests/traefik.yaml
```

---

### Docker Compose (non-k8s, santoso-protocol)

A docker-compose stack also exists for local/dev at:
```
/home/santoso/gunawan-agents/santoso-protocol/docker-compose.yml
```

| Service | Container | Port |
|---------|-----------|------|
| `advisor` | `xlsmart-advisor` | 3000 |
| `telegram-bridge` | `santoso-bridge` | 3200 |

```bash
# Run as santoso:
cd ~/gunawan-agents/santoso-protocol
docker-compose up -d --build
```

---

## Dev Server (user paperclip, non-production)

The development paperclip instance runs as user `paperclip`:

```bash
su - paperclip
cd ~/paperclip-shannon
pnpm dev            # start (watch mode)
pnpm dev:stop       # stop
```

Server runs at `127.0.0.1:3100` (or 3101/3102 if the port is taken).
**Not connected to any domain** — accessible only locally or via SSH tunnel.

To expose externally:
```bash
pnpm dev --bind lan   # bind to 0.0.0.0
# + open firewall: ufw allow 3100
```
