# syslab — Linux Server Build from Scratch

> A fully documented, hands-on Linux server configured step by step for the RHCSA EX200 exam and real-world sysadmin work.

---

## What This Is

This repo documents the complete build of a production-style Linux server — from bare OS to a hardened, monitored, automated system. Every component is configured manually, verified, and explained. No scripts that do the work for you; every command was typed and understood.

**Stack:** RHEL 9 · nginx · firewalld · SELinux · LVM · systemd · SSH

---

## Architecture

```
┌─────────────────────────────────────────┐
│              syslab server              │
│                                         │
│  Users & sudo    →  01-baseline/        │
│  Storage / LVM   →  02-storage/         │
│  Web server      →  03-webserver/       │
│  SSH hardening   →  04-ssh/             │
│  SELinux         →  05-selinux/         │
│  Scheduling      →  06-scheduling/      │
│  Monitoring      →  07-monitoring/      │
│  Backups         →  08-backups/         │
└─────────────────────────────────────────┘
```

---

## Build Progress

| # | Component | Status |
|---|-----------|--------|
| 01 | Baseline — hostname, users, sudo | 🔄 In progress |
| 02 | Storage — LVM, filesystems, fstab | ⏳ Pending |
| 03 | Web server — nginx, firewall | ⏳ Pending |
| 04 | SSH — key auth, hardening | ⏳ Pending |
| 05 | SELinux — contexts, booleans | ⏳ Pending |
| 06 | Scheduling — cron, systemd timers | ⏳ Pending |
| 07 | Monitoring — journald, log rotation | ⏳ Pending |
| 08 | Backups — automated scripts | ⏳ Pending |

---

## How to Read This Repo

Each folder contains a `notes.md` with:
- **What** was configured and **why**
- Every command used, with real output
- Gotchas, exam traps, and lessons learned

---

*Built as part of RHCSA EX200 exam preparation.*
