# Claude Credentials & claude-swap Setup — VPS Shannon

## Problem

User `paperclip` has neither Claude CLI installed nor credentials.
Claude is already logged in under user `santoso`.
The Paperclip server needs to run Claude agents — so credentials must be available under user `paperclip`.

---

## Step 1: Copy Claude Binary to System Path

The `santoso` Claude binary cannot be symlinked directly because `/home/santoso/` has permissions `750` (not accessible by other users).
Solution: **copy the binary** directly to `/usr/local/bin/`.

```bash
# As root — remove old symlink if present, copy binary:
rm -f /usr/local/bin/claude
cp /home/santoso/.local/share/claude/versions/2.1.119 /usr/local/bin/claude
chmod +x /usr/local/bin/claude

# Verify:
su - paperclip -c "claude --version"
```

> **Note**: The version may change. Check the latest version under santoso:
> ```bash
> ls /home/santoso/.local/share/claude/versions/
> ```

---

## Step 2: Copy Santoso's Credentials to Paperclip

Santoso is already logged into Claude. Copy the credentials to user `paperclip`.

```bash
# As root:
cp -r /home/santoso/.claude /home/paperclip/.claude
cp /home/santoso/.claude.json /home/paperclip/.claude.json 2>/dev/null
chown -R paperclip:paperclip /home/paperclip/.claude
chown paperclip:paperclip /home/paperclip/.claude.json 2>/dev/null

# Verify as paperclip:
su - paperclip -c "claude --version"
```

> **Note**: If santoso logs out or refreshes the token, the credentials must be re-copied using the same commands above.

---

## Step 3: Install claude-swap (for account switching)

claude-swap is a third-party tool for switching between multiple Claude accounts without logging out.
Repo: https://github.com/realiti4/claude-swap

```bash
su - paperclip

# Install uv (Python package manager):
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc

# Install claude-swap:
uv tool install claude-swap

# Verify:
cswap --list
```

### claude-swap commands

```bash
cswap --list              # list all saved accounts
cswap --add-account       # save the currently active account
cswap --switch            # switch to the next account
cswap --switch-to 2       # switch to account number 2
cswap --purge             # delete all claude-swap data
```

### Workflow to add a new account:
```bash
claude login              # log in to new account
cswap --add-account       # save to claude-swap
cswap --list              # verify
```

---

## Manual Alternative (without claude-swap)

Back up and restore credentials manually:

```bash
# Save account A:
cp -r ~/.claude ~/.claude-account-a

# Switch to account B:
cp -r ~/.claude-account-b ~/.claude

# Switch back to account A:
cp -r ~/.claude-account-a ~/.claude
```

---

## Important Notes

- Claude credentials are stored in `~/.claude/` and `~/.claude.json`
- Each user has their own credentials — they are not shared automatically
- The Paperclip server (user `paperclip`) requires credentials at `/home/paperclip/.claude/`
- The k3s production pod (user `santoso`) mounts credentials from `/home/santoso/.claude/` — different instance, no conflict
- If the Paperclip server errors `Command not found in PATH: "claude"` → repeat Step 1
- If the Paperclip server errors with a Claude auth/session error → repeat Step 2 (re-copy credentials)
