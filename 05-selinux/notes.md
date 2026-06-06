# 05 — SELinux: Contexts, Booleans & Troubleshooting

## Scenario
SELinux is the number one reason things silently fail on RHEL. Learn to diagnose and fix SELinux issues using the correct permanent approach — not quick fixes that break on the next relabel.

## What Was Configured
- Verified SELinux is in Enforcing mode
- Demonstrated the difference between `chcon` (temporary) and `semanage fcontext` (permanent)
- Created a permanent context rule for `/data` → `httpd_sys_content_t`
- Enabled `httpd_can_network_connect` boolean persistently
- Verified no recent SELinux denials

## Commands Used

```bash
# 1. Check SELinux mode
getenforce
# Output: Enforcing

# 2. Compare contexts
ls -lZ /data/index.html
ls -lZ /var/www/html/
# /var/www/html has httpd_sys_content_t from permanent policy
# /data had httpd_sys_content_t only because of previous chcon (temporary)

# 3. Create new file and check context
echo "SELinux test" | sudo tee /data/test.html
ls -Z /data/test.html
# Inherited httpd_sys_content_t from parent (chcon on directory)

# Prove chcon is temporary — restorecon resets it
sudo restorecon -v /data/test.html
# Relabeled to default_t — chcon wiped out

# 4. Create permanent policy rule
sudo semanage fcontext -a -t httpd_sys_content_t "/data(/.*)?"

# 5. Apply the rule
sudo restorecon -Rv /data/
# Relabeled test.html from default_t to httpd_sys_content_t

# 6. Verify
ls -Z /data/
# Both files: httpd_sys_content_t

# 7. Enable boolean persistently (-P = permanent)
sudo setsebool -P httpd_can_network_connect on

# 8. Verify boolean
getsebool httpd_can_network_connect
# Output: httpd_can_network_connect --> on

# 9. Check audit log for denials
sudo ausearch -m avc -ts recent
# Output: <no matches>
```

## chcon vs semanage — Critical Difference

| Method | Persistent? | Survives restorecon? | Use case |
|--------|------------|---------------------|----------|
| `chcon` | No | No | Quick testing only |
| `semanage fcontext` + `restorecon` | Yes | Yes | Production, exam |

**Always use `semanage` on the exam.** `chcon` fixes will be wiped on the next relabel.

## SELinux Troubleshooting Workflow

```
Something fails silently
       ↓
sudo ausearch -m avc -ts recent   ← find the denial
       ↓
Is it a context issue?    → semanage fcontext + restorecon
Is it a boolean?          → setsebool -P <boolean> on
Is it a port label?       → semanage port -a -t <type> -p tcp <port>
```

## Key Commands Reference

| Command | Purpose |
|---------|---------|
| `getenforce` | Show current SELinux mode |
| `sestatus` | Full SELinux status |
| `ls -Z` | Show SELinux context of files |
| `chcon -t type file` | Temporarily change context |
| `semanage fcontext -a -t type "/path(/.*)?"` | Add permanent context rule |
| `restorecon -Rv /path/` | Apply policy rules to files |
| `getsebool -a` | List all booleans |
| `setsebool -P boolean on` | Set boolean persistently |
| `ausearch -m avc -ts recent` | Show recent SELinux denials |

## Gotchas
- **Never disable SELinux** on the exam — set to permissive (`setenforce 0`) to troubleshoot, then fix the root cause.
- `chcon` changes are lost when `restorecon` is run — never use it as a permanent fix.
- The `-P` flag on `setsebool` is required for persistence — without it the boolean resets on reboot.
- SELinux denials are silent — the service appears to work but returns errors (like 403). Always check `ausearch` when something unexpectedly fails on RHEL.

## What I Learned
- SELinux enforces mandatory access control on top of regular Unix permissions — both must be correct for access to work.
- The correct fix order: check context with `ls -Z`, add rule with `semanage fcontext`, apply with `restorecon`.
- Booleans allow fine-grained policy tuning without writing custom SELinux policy.
- A clean `ausearch` output means no recent denials — useful baseline to verify after any config change.
