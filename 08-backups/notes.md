# 08 — Backups: Automated Backup Script

## Scenario
The web server is running and serving content from `/data`. Build an automated nightly backup system that keeps the last 7 days of backups and logs every run.

## What Was Configured
- Backup directory at `/backups`
- Script `/usr/local/bin/backup.sh` — timestamped tar.gz archives with success/failure logging and 7-day retention
- Scheduled daily at 03:00 via root's crontab

## The Backup Script

```bash
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/backups/data_${TIMESTAMP}.tar.gz"
LOG="/var/log/backup.log"

tar -czf "$BACKUP_FILE" /data 2>/dev/null

if [ $? -eq 0 ]; then
    echo "$TIMESTAMP - SUCCESS: $BACKUP_FILE" >> "$LOG"
else
    echo "$TIMESTAMP - FAILED: could not create $BACKUP_FILE" >> "$LOG"
fi

find /backups -name "*.tar.gz" -mtime +7 -delete
```

## Commands Used

```bash
# 1. Create backup directory
sudo mkdir -p /backups

# 2. Create script
sudo tee /usr/local/bin/backup.sh << 'EOF'
... (script above)
EOF

# 3. Make executable and test
sudo chmod +x /usr/local/bin/backup.sh
sudo /usr/local/bin/backup.sh

# 4. Verify
ls -lh /backups/
# Output: data_20260607_010642.tar.gz

cat /var/log/backup.log
# Output: 20260607_010642 - SUCCESS: /backups/data_20260607_010642.tar.gz

# 5. Schedule daily at 03:00
sudo crontab -e
# Added: 0 3 * * * /usr/local/bin/backup.sh

# 6. Verify
sudo crontab -l
```

## Script Logic Explained

```
TIMESTAMP=$(date +%Y%m%d_%H%M%S)   → unique filename per run
tar -czf "$BACKUP_FILE" /data       → create compressed archive
$?                                  → exit code: 0=success, non-zero=failure
if [ $? -eq 0 ]                     → check if tar succeeded
find ... -mtime +7 -delete          → delete archives older than 7 days
```

## tar Flags Reference

| Flag | Meaning |
|------|---------|
| `-c` | Create archive |
| `-z` | Compress with gzip |
| `-f file` | Output filename |
| `-x` | Extract archive |
| `-t` | List contents |
| `-v` | Verbose output |

## Restore a Backup

```bash
# List contents of a backup
tar -tzf /backups/data_20260607_010642.tar.gz

# Restore to original location
sudo tar -xzf /backups/data_20260607_010642.tar.gz -C /
```

## Gotchas
- **Always test restoring** — a backup you've never restored is not a backup.
- `$?` must be checked immediately after the command — any other command between them resets it.
- `find -mtime +7` means "strictly more than 7 days old" — files exactly 7 days old are kept.
- `2>/dev/null` in the tar command suppresses warnings about changed files during backup — remove it if you want to see errors.
- Cron runs with a minimal environment — use full paths for all commands in cron scripts.

## What I Learned
- A production backup script needs three things: create the archive, log the result, clean up old archives.
- `$?` and `if/then/else` are the foundation of robust bash scripting.
- Timestamped filenames are the simplest and most reliable naming strategy for backups.
- Scheduling with `0 3 * * *` means "at minute 0 of hour 3" = 03:00 daily.
