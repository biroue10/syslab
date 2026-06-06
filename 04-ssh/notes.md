# 04 — SSH: Key-Based Authentication & Hardening

## Scenario
Password-based SSH is vulnerable to brute-force attacks. Configure key-based authentication for `sysadmin` and lock down the SSH daemon to disable password login and direct root access.

## What Was Configured
- RSA-4096 key pair generated (ED25519 blocked by FIPS mode)
- Public key deployed to `sysadmin` via `ssh-copy-id`
- `PermitRootLogin no` and `PasswordAuthentication no` set in sshd config
- Drop-in config file override discovered and fixed
- Key-based login verified after hardening

## Commands Used

```bash
# 1. Generate RSA-4096 key pair (ED25519 not allowed in FIPS mode)
ssh-keygen -t rsa -b 4096

# 2. Copy public key to sysadmin
ssh-copy-id sysadmin@localhost

# 3. Test key-based login
ssh sysadmin@localhost
# Result: logged in without password prompt

# 4. Harden sshd_config
sudo vim /etc/ssh/sshd_config
# Set: PermitRootLogin no
# Set: PasswordAuthentication no

# Fix drop-in override (RHEL 10 specific)
sudo grep -r "PermitRootLogin" /etc/ssh/sshd_config.d/
# Found: /etc/ssh/sshd_config.d/01-permitrootlogin.conf: PermitRootLogin yes
sudo sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config.d/01-permitrootlogin.conf

# 5. Restart SSH daemon
sudo systemctl restart sshd

# 6. Verify effective running config
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication"
# Output:
# permitrootlogin no
# passwordauthentication no

# 7. Confirm key login still works after hardening
ssh sysadmin@localhost
# Result: logged in without password
```

## Key Concepts

| Concept | Detail |
|---------|--------|
| Private key | Stays on your machine — never share it |
| Public key | Placed in `~/.ssh/authorized_keys` on the server |
| `ssh-copy-id` | Safely deploys public key with correct permissions |
| `sshd -T` | Dumps the full effective running config — more reliable than reading the file |
| Drop-in configs | `/etc/ssh/sshd_config.d/*.conf` override the main config on RHEL 10 |

## Gotchas & Lessons Learned

- **FIPS mode** disallows ED25519 — use `rsa -b 4096` on FIPS-enabled systems.
- **Always test key login before disabling `PasswordAuthentication`** — a mistake here locks you out completely.
- **Drop-in files win** — on RHEL 10, `/etc/ssh/sshd_config.d/` files take precedence over `/etc/ssh/sshd_config`. Always check there when a setting doesn't take effect.
- `sshd -T` is the authoritative way to verify what the daemon is actually using — not just `grep` on the config file.
- After any `sshd_config` change, `systemctl restart sshd` is required — changes don't apply on `reload` for all options.

## What I Learned
- Key-based SSH is the production standard — passwords are never used on real servers.
- SSH hardening is two steps: set the config AND verify with `sshd -T` that it's actually active.
- RHEL's drop-in config system (`sshd_config.d/`) means the main config file is not the whole story.
