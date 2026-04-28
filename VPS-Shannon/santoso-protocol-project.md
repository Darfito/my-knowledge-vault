# Santoso Protocol Project

The main santoso project lives at:
```
/home/santoso/gunawan-agents/santoso-protocol/
```

## Structure

```
santoso-protocol/
├── CLAUDE.md                    ← Claude Code context file
├── README.md
├── IMPLEMENTATION-GUIDE.md
├── SANTOSO-PROTOCOL-PROJECT.md
├── Dockerfile
├── docker-compose.yml           ← Local dev stack
├── agents/                      ← AI agent definitions
├── docs/
├── k8s/
│   ├── manifests/               ← Kubernetes YAMLs
│   ├── docker/paperclip/        ← Dockerfile for the paperclip image
│   ├── scripts/                 ← Helper scripts (snapshot, seed, etc.)
│   └── snapshots/               ← DB snapshots
├── scripts/
├── services/
└── telegram-bridge/             ← Telegram bot bridge
```

## gunawan-agents Repo

```
/home/santoso/gunawan-agents/
├── README.md
├── mobile-gunawan/
├── rust-gunawan/
├── santoso-protocol/    ← main project
└── webapp-gunawan/
```

## Claude Integration

Santoso runs Claude Code directly on the VPS.
Cache is at `/home/santoso/.cache/claude-cli-nodejs/`.
OAuth credentials are at `/home/santoso/.claude/`.

The Paperclip production pod mounts these credentials:
```yaml
volumeMounts:
  - name: claude-credentials
    mountPath: /home/node/.claude      # from /home/santoso/.claude
  - name: claude-config
    mountPath: /home/node/.claude.json # from /home/santoso/.claude.json
```

## Paperclip DB Snapshot

```bash
# Script to snapshot the paperclip database:
~/gunawan-agents/santoso-protocol/k8s/scripts/snapshot-paperclip.sh
```

Snapshots are stored at:
```
~/gunawan-agents/santoso-protocol/k8s/snapshots/
```
