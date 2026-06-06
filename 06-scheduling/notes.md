# 06 — Scheduling: Cron & Systemd Timers

## Scenario
Every production server runs automated tasks — health checks, log rotation, backups, cleanups. Schedule a health check with cron and a /tmp cleanup with a systemd timer.

## What Was Configured
- Health check script running every minute via cron, appending to `/var/log/health.log`
- Systemd `cleanup.service` + `cleanup.timer` running daily at 02:00 to delete old `/tmp` files

## Commands Used

```bash
# 1. Create health check script
sudo tee /usr/local/bin/health-check.sh << 'EOF'
#!/bin/bash
echo "=== $(date) ===" >> /var/log/health.log
uptime >> /var/log/health.log
df -h >> /var/log/health.log
echo "" >> /var/log/health.log
EOF

# 2. Make executable
sudo chmod +x /usr/local/bin/health-check.sh

# Test manually before scheduling
sudo /usr/local/bin/health-check.sh
cat /var/log/health.log

# 3. Schedule with cron (root)
sudo crontab -e
# Added: * * * * * /usr/local/bin/health-check.sh

sudo crontab -l

# 4. Verify after 2 minutes
cat /var/log/health.log
# Confirmed entries at every minute

# 5. Create systemd service unit
sudo tee /etc/systemd/system/cleanup.service << 'EOF'
[Unit]
Description=Clean up old files in /tmp

[Service]
Type=oneshot
ExecStart=find /tmp -type f -mtime +1 -delete
EOF

# 6. Create systemd timer unit
sudo tee /etc/systemd/system/cleanup.timer << 'EOF'
[Unit]
Description=Daily cleanup of /tmp

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF

# 7. Enable and start timer
sudo systemctl daemon-reload
sudo systemctl enable --now cleanup.timer

# 8. Verify timer
systemctl list-timers cleanup.timer
# Output: next run Sun 2026-06-07 02:00:00 CEST
```

## Cron Syntax

```
* * * * * command
│ │ │ │ │
│ │ │ │ └── day of week (0=Sun, 6=Sat)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)

Examples:
* * * * *        → every minute
0 2 * * *        → daily at 02:00
0 2 * * 0        → every Sunday at 02:00
*/5 * * * *      → every 5 minutes
```

## Systemd Timer Unit Structure

```ini
[Unit]
Description=...

[Timer]
OnCalendar=*-*-* 02:00:00   # daily at 2am
Persistent=true               # run missed jobs on next boot

[Install]
WantedBy=timers.target
```

## Cron vs Systemd Timers

| Feature | Cron | Systemd Timer |
|---------|------|---------------|
| Logging | Syslog only | journald (full integration) |
| Monitoring | `crontab -l` | `systemctl list-timers` |
| Missed jobs | No | Yes (with `Persistent=true`) |
| Dependencies | No | Yes (systemd units) |
| RHEL default | Legacy | Modern standard |

## Key Commands

| Command | Purpose |
|---------|---------|
| `crontab -e` | Edit current user's crontab |
| `crontab -l` | List current user's crontab |
| `sudo crontab -e` | Edit root's crontab |
| `systemctl list-timers` | Show all active timers + next run |
| `systemctl enable --now unit` | Enable at boot AND start immediately |
| `journalctl -u cleanup.service` | View service run logs |

## Gotchas
- **Always test scripts manually** before scheduling — a broken script scheduled to run every minute fills logs fast.
- `systemctl enable` without `--now` = timer won't activate until next reboot.
- `Persistent=true` ensures missed runs (e.g. server was off at 02:00) execute on next boot.
- `sudo crontab -e` edits root's crontab — `crontab -e` (no sudo) edits your own user's crontab. They are different.
- After creating/modifying systemd units, always run `systemctl daemon-reload` before enabling.

## What I Learned
- Cron is simple and reliable for basic scheduling; systemd timers are more powerful and better integrated with RHEL.
- A systemd timer always needs a matching `.service` unit — the timer is just the schedule, the service does the work.
- `systemctl list-timers` is far more useful than `crontab -l` for understanding when jobs will run.
