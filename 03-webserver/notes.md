# 03 — Web Server: nginx + Firewall + SELinux

## Scenario
The `/data` volume needs to serve web content. Install nginx, point it at `/data`, open the firewall, and confirm the server is reachable. A real end-to-end service deployment.

## What Was Configured
- nginx installed and enabled as a persistent service
- `/data/index.html` created as the web root
- nginx configured to serve from `/data`
- Port 80 (HTTP) opened permanently in firewalld
- SELinux context fixed to allow nginx to read `/data`

## Commands Used

```bash
# 1. Install nginx
sudo dnf install nginx -y

# 2. Start and enable nginx
sudo systemctl start nginx
sudo systemctl enable nginx

# 3. Verify nginx is running
sudo systemctl status nginx

# 4. Create the web page
echo "Welcome to syslab" | sudo tee /data/index.html

# 5. Configure nginx web root — edit /etc/nginx/nginx.conf
# Changed: root /usr/share/nginx/html  →  root /data;

# 6. Reload nginx to apply config
sudo systemctl reload nginx

# 7 & 8. Open HTTP in firewall (permanently + reload to activate)
sudo systemctl restart firewalld   # needed because firewalld started with read-only FS
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
# Output: cockpit dhcpv6-client http ssh

# 9. Fix SELinux context — nginx requires httpd_sys_content_t
sudo chcon -Rv --type=httpd_sys_content_t /data/

# Test
curl http://localhost
# Output: Welcome to syslab
```

## Key Concepts

| Concept | Detail |
|---------|--------|
| `systemctl start` + `enable` | Start now AND survive reboot — always both |
| `firewall-cmd --permanent` | Saves rule to disk — does NOT activate immediately |
| `firewall-cmd --reload` | Activates permanent rules — always run after `--permanent` |
| `httpd_sys_content_t` | SELinux context required for nginx to read files |
| `chcon -Rv --type=X /path/` | Recursively change SELinux context |

## Troubleshooting: 403 Forbidden

nginx returned `403 Forbidden` even though file permissions were correct (`-rw-r--r--`).

**Root cause:** The `/data` filesystem was newly created and had no SELinux labels — files showed `unlabeled_t` context instead of `httpd_sys_content_t`. SELinux silently denied nginx access.

**Lesson:** On RHEL, a 403 from nginx almost always means SELinux — check with `ls -laZ` before chasing permissions.

## Gotchas
- `sudo echo "..." > file` does NOT work — the `>` runs as your user, not root. Use `echo "..." | sudo tee file` instead.
- `firewall-cmd --permanent` alone does nothing until `--reload` is run.
- A service that starts during a read-only filesystem event retains stale file handles — restart it after the FS is fixed.
- New filesystems (especially on loop/LVM) have `unlabeled_t` SELinux context — always relabel before serving content.

## What I Learned
- Deploying a service requires four things: install, start, enable, and open the firewall.
- SELinux is silent — it doesn't scream at you, it just blocks things. Always check context with `ls -laZ` when something works on permissions but still fails.
- `curl http://localhost` is the fastest way to verify a web server is working correctly.
