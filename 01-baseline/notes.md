# 01 — Baseline: Server Identity & User Management

## Scenario
A new server was just provisioned. Before anything else gets installed, give it a proper identity, create team accounts, and grant admin access via sudo — not by sharing the root password.

## What Was Configured
- Hostname set to `syslab.local`
- User `sysadmin` created with sudo access via the `wheel` group
- User `developer` created with no sudo access

## Commands Used

```bash
# Set hostname persistently
hostnamectl set-hostname syslab.local

# Verify hostname took effect immediately
hostnamectl status
hostname

# Create sysadmin user with home directory
useradd sysadmin

# Set password for sysadmin
passwd sysadmin

# Grant sudo access by adding to wheel group
usermod -aG wheel sysadmin

# Verify group membership
id sysadmin
# Output: uid=5063(sysadmin) gid=5000(sysadmin) groups=5000(sysadmin),10(wheel)

# Create developer user (no sudo)
useradd developer

# Set password for developer
passwd developer

# Verify both users exist
grep -E 'sysadmin|developer' /etc/passwd

# Test sudo access
su - sysadmin
sudo whoami
# Output: root
```

## Verification Output

```
# hostnamectl status
Static hostname: syslab.local
Operating System: Red Hat Enterprise Linux 10.2

# grep -E 'sysadmin|developer' /etc/passwd
sysadmin:x:5063:5000::/home/sysadmin:/bin/bash
developer:x:5064:5064::/home/developer:/bin/bash

# sudo whoami
root
```

## Key Concepts

| Concept | Detail |
|---------|--------|
| `hostnamectl` | Persistent hostname change — writes to `/etc/hostname` |
| `wheel` group | Members can run sudo on RHEL — never share the root password |
| `useradd` | Creates user + home directory + sets default shell |
| `usermod -aG` | Adds user to a group without removing existing groups (`-a` = append) |
| `/etc/passwd` | Stores user accounts (not passwords — those are in `/etc/shadow`) |

## Gotchas
- `usermod -G wheel` without `-a` **removes the user from all other groups** — always use `-aG`.
- Changing the hostname takes effect immediately with `hostnamectl` — no reboot needed.
- `su - username` (with the dash) loads the user's full environment. `su username` (no dash) doesn't — always use the dash.

## What I Learned
- Server baseline is the first thing configured before any service is installed.
- Sudo via `wheel` group is the RHEL standard — it keeps root access auditable and avoids shared passwords.
- Always verify with `sudo whoami` — not just `id` — to confirm sudo is actually functional.
