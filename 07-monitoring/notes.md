# 07 — Monitoring: Journald & Log Management

## Scenario
Something broke on the server at 3am. Figure out what happened using only the logs. Reading logs is how sysadmins diagnose everything from failed services to security incidents.

## What Was Practiced
- Querying journald with filters (service, priority, boot, real-time)
- Comparing journald vs traditional rsyslog text logs
- Checking disk usage of journal
- Reading `/var/log/messages` and `/var/log/secure`

## Commands Used

```bash
# 1. Last 20 lines of journal
journalctl -n 20

# 2. Follow journal in real time
journalctl -f
# (Ctrl+C to stop)

# 3. Filter by service
journalctl -u nginx

# 4. Filter by priority (errors and above)
journalctl -p err -n 20

# 5. This boot only
journalctl -b

# 6. Journal disk usage
journalctl --disk-usage
# Output: 268.7M

# 7. Traditional messages log
sudo tail -n 10 /var/log/messages

# 8. Authentication log
sudo tail -n 15 /var/log/secure
```

## Most Useful journalctl Flags

| Flag | Purpose |
|------|---------|
| `-n N` | Last N lines |
| `-f` | Follow in real time |
| `-u service` | Filter by systemd unit |
| `-p err` | Priority: err and above |
| `-b` | Current boot only |
| `-b -1` | Previous boot |
| `--since "1 hour ago"` | Time-based filter |
| `--disk-usage` | Journal disk usage |
| `-xe` | Recent entries with explanations (use when service fails) |

## Priority Levels

| Level | Number | Meaning |
|-------|--------|---------|
| emerg | 0 | System unusable |
| alert | 1 | Immediate action required |
| crit | 2 | Critical condition |
| err | 3 | Error |
| warning | 4 | Warning |
| notice | 5 | Normal but significant |
| info | 6 | Informational |
| debug | 7 | Debug messages |

## journald vs rsyslog

| Feature | journald | rsyslog |
|---------|----------|---------|
| Format | Binary | Plain text |
| Location | `/run/log/journal` or `/var/log/journal` | `/var/log/messages`, `/var/log/secure` etc. |
| Resilience | Survives read-only FS | Stops writing if FS is read-only |
| Query tool | `journalctl` | `grep`, `tail`, `less` |
| Persistence | Configurable | Always text on disk |

## Real Incident from This Session
`/var/log/messages` stopped updating on May 29 when the root filesystem went read-only. journald kept capturing everything. This proves journald is more resilient — always check `journalctl` first when traditional logs seem stale.

## Gotchas
- `journalctl -xe` is the single most useful command when a service fails to start — use it immediately after a failed `systemctl start`.
- Journal logs are not persistent by default on some systems — check `/var/log/journal` exists; if not, create it and restart systemd-journald.
- `-p err` shows priority 3 and below (more severe) — not "err and lower severity."
- `journalctl -b -1` shows the previous boot — extremely useful after a crash or unexpected reboot.

## What I Learned
- journald is the authoritative log source on RHEL — rsyslog text files are secondary.
- Filtering with `-u`, `-p`, and `-b` makes finding relevant logs fast even on busy systems.
- A sysadmin reads logs before touching anything — understanding what happened is always step one.
