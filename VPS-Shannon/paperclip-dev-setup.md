# Paperclip Dev Server Setup — VPS Shannon

Complete setup for running the paperclip dev server on the VPS as user `paperclip`.

## One-time Setup (already done — no need to repeat)

### 1. Create the paperclip user
```bash
# As root:
useradd -m -s /bin/bash paperclip
```

### 2. Install Node.js via nvm
```bash
su - paperclip
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
export NVM_DIR="$HOME/.nvm" && [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 22
npm install -g pnpm@9.15.4
```

### 3. Clone & install the project
```bash
# Copy from /home/labs or clone fresh
cp -r /home/labs/paperclip-shannon ~/paperclip-shannon
chown -R paperclip:paperclip ~/paperclip-shannon

# Install dependencies
cd ~/paperclip-shannon
pnpm config set store-dir "~/.pnpm-store"
pnpm install
```

### 4. Create .env
```bash
echo "BETTER_AUTH_SECRET=paperclip-dev-secret" > ~/paperclip-shannon/.env
```

### 5. Onboard (create config & admin account)
```bash
cd ~/paperclip-shannon
pnpm paperclipai onboard
# Follow the prompts — this creates config at ~/.paperclip/instances/default/config.json
```

### 6. Add the VPS hostname
```bash
pnpm paperclipai allowed-hostname 84.247.150.72
```

### 7. Open the firewall (as root)
```bash
ufw allow 3100
```

---

## Running the Dev Server (day-to-day)

```bash
su - paperclip
cd ~/paperclip-shannon

# Start (accessible from the internet):
pnpm dev --bind lan

# Stop:
pnpm dev:stop
# or Ctrl+C
```

Access in browser: **http://84.247.150.72:3100**

---

## Troubleshooting

### Port already in use — check what's on it
```bash
for p in 3100 3101 3102; do pid=$(fuser $p/tcp 2>/dev/null); [ -n "$pid" ] && echo "PORT $p → PID $pid → $(ps -p $pid -o user=,cmd= 2>/dev/null)" || echo "PORT $p → free"; done
```

### PostgreSQL stale pid / shared memory error
```bash
pkill -u paperclip postgres 2>/dev/null; sleep 2
rm -f ~/.paperclip/instances/default/db/postmaster.pid
ipcs -m | grep $(id -u) | awk '{print $2}' | xargs -r ipcrm -m 2>/dev/null
pnpm dev --bind lan
```

### Server starts but jumps to port 3101/3102
Another process is holding port 3100. Find it:
```bash
fuser 3100/tcp
ps aux | grep <PID>
# Kill if needed:
kill <PID>
```

Port 3100 was previously held by `admin_ssh` running `paperclipai onboard --yes` (stuck since April 24).

### "No company access" after login
New accounts created from the UI are **not** automatically granted company access — Paperclip's `CloudAccessGate` (`ui/src/components/CloudAccessGate.tsx`) blocks any session that has no active `companyMemberships` row and no `instance_admin` role. Two ways to fix:

**Quick path — just use the admin account.** Log in with the account created during `pnpm paperclipai onboard`. That account is the instance admin for this VPS instance and bypasses the gate. Email is recorded at `~/.paperclip/instances/default/config.json`.

**Proper path — grant the new account access from the admin UI.**

1. Sign in as the onboard admin account.
2. Go to **`/instance/settings/access`** (the InstanceAccess page).
3. Search for the new account by email or name and select it.
4. In the **Company access** section, tick the checkbox next to the Shannon company and click **Save company access**. This writes a `companyMemberships` row with `status = "active"` and `membershipRole = "operator"` directly — no invite link, no join-request approval needed.
5. Optionally toggle **Promote to instance admin** on the same page if the new account should also manage other users.
6. Sign out, sign in again as the new account — the gate clears.

Server endpoints behind the UI: `PUT /admin/users/{userId}/company-access` and `POST /admin/users/{userId}/promote-instance-admin` (`server/src/routes/access.ts`). The invite-link flow at `/companies/{id}/invites` also works but creates a `pending_approval` join request that still requires admin approval — the InstanceAccess page is the one-step shortcut.

### BETTER_AUTH_SECRET not set
```bash
echo "BETTER_AUTH_SECRET=paperclip-dev-secret" >> ~/paperclip-shannon/.env
```

---

## Development Workflow (laptop → VPS)

```
1. Edit code on laptop
2. git push
3. On VPS (user paperclip):
   git pull
   # Server auto-restarts in watch mode
   # If not: pnpm dev:stop && pnpm dev --bind lan
```

Watch mode (`pnpm dev --bind lan`) automatically restarts on file changes after `git pull`.

---

## Important Notes

- **Do not run as root** — PostgreSQL refuses to run as root
- **Do not run from `/home/labs/`** — that directory belongs to user `labs` and has no write permission here
- Always run from `/home/paperclip/paperclip-shannon/`
- Config is stored at `~/.paperclip/instances/default/config.json`
- Database is at `~/.paperclip/instances/default/db/`
- Automatic backups run every 60 minutes to `~/.paperclip/instances/default/data/backups/`
