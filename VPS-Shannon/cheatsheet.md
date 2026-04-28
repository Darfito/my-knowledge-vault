# Cheatsheet — VPS Shannon

## SSH & Switch User

```bash
ssh root@84.247.150.72          # log in as root
su - santoso                     # switch to santoso (from root, no password)
su - paperclip                   # switch to paperclip (from root)
```

## k3s

```bash
systemctl start k3s              # start the cluster
systemctl stop k3s               # stop the cluster
systemctl enable k3s             # auto-start on reboot
kubectl get pods -n gunawan      # check production pods
kubectl get ingress -n gunawan   # check domain routing
```

## k9s (must run as santoso)

```bash
su - santoso
k9s                              # open Kubernetes TUI
```

## Paperclip Production (k8s)

```bash
# Make sure k3s is running first, then:
kubectl get pods -n gunawan                          # check status
kubectl logs -n gunawan paperclip-0 -f              # tail logs
kubectl rollout restart statefulset/paperclip -n gunawan  # restart pod

# Re-apply manifests:
kubectl apply -f ~/gunawan-agents/santoso-protocol/k8s/manifests/
```

**Production domain**: https://gunawan-paperclip.digital-lab.ai

## Paperclip Dev Server (user paperclip)

```bash
su - paperclip
cd ~/paperclip-shannon
pnpm dev              # start
pnpm dev:stop         # stop
pnpm dev --bind lan   # start + accessible from outside (port 3100)
```

## Docker Compose (santoso)

```bash
su - santoso
cd ~/gunawan-agents/santoso-protocol
docker-compose up -d --build    # build & start
docker-compose down             # stop
docker ps                       # check containers
```

## Check Running Ports

```bash
ss -tlnp | grep -E '80|443|3100|3101|3102'
fuser -k 3101/tcp               # kill process on port 3101
```

## Nginx

Nginx is **not used** — domain routing goes through **Traefik** (built into k3s).
Nginx status is `inactive (dead)` and `disabled`.

## Key File Locations

| File/Folder | Description |
|-------------|-------------|
| `~/gunawan-agents/santoso-protocol/k8s/manifests/` | All Kubernetes YAMLs |
| `~/gunawan-agents/santoso-protocol/docker-compose.yml` | Docker Compose local stack |
| `/home/paperclip/paperclip-shannon/` | Paperclip dev server |
| `/home/santoso/.claude/` | Claude OAuth credentials (mounted into pod) |
| `/home/santoso/.paperclip/` | Santoso's paperclip instance data |
| `/var/lib/rancher/k3s/server/manifests/traefik.yaml` | k3s Traefik config |
