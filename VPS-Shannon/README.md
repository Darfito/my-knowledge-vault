# VPS Shannon — Knowledge Vault

Contabo VPS used as the primary server for the Gunawan AI Company stack deployment.

## Basic Info

| Item | Value |
|------|-------|
| Provider | Contabo |
| IP | 84.247.150.72 |
| OS | Debian/Ubuntu Linux |
| Internal name | vmi3229394 |

## Users

| User | UID | Role |
|------|-----|------|
| `root` | 0 | Full admin |
| `santoso` | 1000 | Main operator — has sudo, docker, k9s |
| `admin_ssh` | 1001 | SSH admin |
| `paperclip` | 1002 | Runs the paperclip dev server (non-root) |
| `labs` | — | Old project at `/home/labs/paperclip-shannon` |

## Login

```bash
ssh root@84.247.150.72
# or
ssh santoso@84.247.150.72
```

Switch from root to santoso without a password:
```bash
su - santoso
```

---

## Important Notes

- **Do not run anything as root** that requires PostgreSQL — postgres refuses to run as root
- User `paperclip` is a dedicated user for running the dev server at `/home/paperclip/paperclip-shannon`
- `/home/labs/paperclip-shannon` is the old installation — do not use it, use the `paperclip` user's copy instead
