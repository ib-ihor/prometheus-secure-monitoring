# Secure Remote Server Monitoring (Prometheus + Node Exporter + Nginx)

This repository contains a production-ready configuration for safely monitoring a remote web server with a public IP address using **Prometheus** and **Node Exporter** inside Docker, secured behind an **Nginx Reverse Proxy**.

## 🛑 The Problem
By default, Node Exporter exposes system metrics on public port `9100` without any authentication. Leaving this port open to the entire internet allows attackers to gather sensitive infrastructure data (processes, disks, network states, etc.).

## 🚀 The Solution
This architecture completely hides port `9100` from the public internet by binding it strictly to `127.0.0.1`. All incoming monitoring traffic is routed through **Nginx**, which enforces:
1. **SSL/TLS Encryption** via Let's Encrypt (prevents password sniffing over public networks).
2. **Basic Authentication (bcrypt)** to block unauthorized access.

---

## 🛠️ Deployment Guide

### 1. Deploy Node Exporter
Clone this repository to your remote server and run:
```bash
docker compose up -d
```
*Note: Node Exporter is now running but only accessible from within the server itself.*

### 2. Configure Nginx & Basic Auth
Generate a secure password hash using Docker (replace `monitor_user` with your username):
```bash
docker run --rm -ti alpine sh -c "apk add --no-cache apache2-utils && htpasswd -nBC 12 monitor_user"
```
Save the output string into `/etc/nginx/.htpasswd` on your host machine.

Copy the `nginx/node_exporter.conf` file to `/etc/nginx/sites-available/`, enable it, and restart Nginx:
```bash
sudo ln -s /etc/nginx/sites-available/node_exporter.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

### 3. Secure with SSL (Certbot)
Obtain a free Let's Encrypt SSL certificate to enable HTTPS:
```bash
sudo certbot --nginx -d my.domen.com
```

### 4. Connect Prometheus
Add the following configuration to your central `prometheus.yml` instance:
```yaml
scrape_configs:
  - job_name: 'remote-web-server'
    scheme: https
    basic_auth:
      username: 'monitor_user'
      password: 'your_secure_password_in_cleartext'
    static_configs:
      - targets: ['my.domen.com:443']
```

---

## 🏆 Key Achievements
- **Zero Attack Surface**: Port 9100 is closed to external scanners.
- **MITM Protection**: All metrics and credentials transit via encrypted HTTPS.
- **Production-Ready**: Scalable pattern for multi-server environments.
