# Jellyfin Media Stack with Nginx Proxy Manager

A complete, production-ready Docker Compose setup for a self-hosted media server featuring Jellyfin, automated content management with the *arr apps, VPN-protected torrenting, and domain routing via Nginx Proxy Manager.

## 📋 Table of Contents

- [Overview](#overview)
- [Stack Components](#stack-components)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
- [Installation](#installation)
- [Nginx Proxy Manager Setup](#nginx-proxy-manager-setup)
- [Domain Configuration](#domain-configuration)
- [Service Configuration](#service-configuration)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Useful Commands](#useful-commands)

## 🎯 Overview

This stack provides a complete media automation and streaming solution:
- **Jellyfin** for streaming your media library
- **Sonarr** for TV show management
- **Radarr** for movie management
- **Prowlarr** for indexer management
- **qBittorrent** for downloading content (VPN-protected via Gluetun)
- **Jellyseerr** for media requests
- **Nginx Proxy Manager** for reverse proxy with SSL certificates

All services are containerized and can be accessed through your own custom domain with automatic HTTPS certificates.

## 🧩 Stack Components

### Core Services

| Service | Port | Purpose |
|---------|------|---------|
| **Jellyfin** | 8096 | Media streaming server |
| **Sonarr** | 8989 | TV show management |
| **Radarr** | 7878 | Movie management |
| **Prowlarr** | 9696 | Indexer management |
| **qBittorrent** | 8080 | Torrent client (VPN-protected) |
| **Jellyseerr** | 5055 | Media request platform |
| **Nginx Proxy Manager** | 81 | Admin UI for proxy configuration |

### Supporting Infrastructure

- **Gluetun**: VPN gateway that routes qBittorrent traffic through ProtonVPN
- **Docker Networks**: `media-net` for inter-service communication

## 🏗️ Architecture

```
Internet
    │
    ├─> Nginx Proxy Manager (Ports 80, 443)
    │       │
    │       ├─> jellyfin.yourdomain.com → Jellyfin (8096)
    │       └─> requests.yourdomain.com → Jellyseerr (5055)
    │
    └─> VPN Gateway (Gluetun)
            └─> qBittorrent (all torrent traffic)
```

**Note:** Other services (Sonarr, Radarr, Prowlarr, qBittorrent) can be accessed directly via their ports on your local network and do not need public domain routing.

### VPN Protection
- qBittorrent runs within Gluetun's network namespace
- All torrent traffic is routed through ProtonVPN
- Only the WebUI is exposed on port 8080
- If VPN fails, qBittorrent loses all connectivity (killswitch)

## ✅ Prerequisites

### System Requirements
- Linux server (Ubuntu 20.04+ recommended)
- Docker Engine 20.10+
- Docker Compose v2.0+
- At least 4GB RAM (8GB+ recommended)
- Sufficient storage for media files

### Network Requirements
- Static IP address for your server (recommended)
- Router access for port forwarding (if accessing from outside your network)
- Domain name pointed to your public IP

### Required Accounts
- ProtonVPN account with port forwarding support
- Domain registrar account (for DNS configuration)

## 📁 Directory Structure

Create the following directory structure on your host:

```bash
sudo mkdir -p /mnt/docker/{jellyfin/{config,cache},sonarr,radarr,prowlarr,qbittorrent,jellyseerr/config,nginxproxymanager/{data,letsencrypt},media/{movies,tv,downloads/{complete,incomplete}}}
```

Set proper ownership (adjust UID/GID as needed):

```bash
sudo chown -R 1000:1000 /mnt/docker
sudo chmod -R 775 /mnt/docker
```

### Directory Layout

```
/mnt/docker/
├── jellyfin/
│   ├── config/          # Jellyfin configuration
│   └── cache/           # Jellyfin cache
├── sonarr/              # Sonarr configuration
├── radarr/              # Radarr configuration
├── prowlarr/            # Prowlarr configuration
├── qbittorrent/         # qBittorrent configuration
├── jellyseerr/
│   └── config/          # Jellyseerr configuration
├── nginxproxymanager/
│   ├── data/            # NPM configuration
│   └── letsencrypt/     # SSL certificates
└── media/
    ├── movies/          # Movie library
    ├── tv/              # TV show library
    └── downloads/
        ├── complete/    # Completed downloads
        └── incomplete/  # In-progress downloads
```

## 🚀 Installation

### Step 1: Clone this Repository

```bash
git clone https://github.com/julian-heidt/jellyfin-media-stack.git
cd jellyfin-media-stack
```

### Step 2: Configure Environment Variables

Copy the example environment file and edit it with your settings:

```bash
cp stack.env.example stack.env
nano stack.env
```

Key variables to configure:
- `TZ`: Your timezone (e.g., `America/New_York`)
- `PUID` and `PGID`: Your user and group IDs (run `id` command to find these)
- `OPENVPN_USER`: Your ProtonVPN username
- `OPENVPN_PASSWORD`: Your ProtonVPN password
- `SERVER_HOSTINGS`: ProtonVPN server (e.g., `node-xx-xx.protonvpn.net`) - Choose a server that supports port forwarding
- `JELLYFIN_PublishedServerUrl`: Your Jellyfin public URL (e.g., `jellyfin.yourdomain.com`)

**Important**: The `stack.env` file contains sensitive credentials and is excluded from Git. Never commit this file to version control.

### Step 3: Create Docker Network

```bash
docker network create media-net
```

### Step 4: Deploy Nginx Proxy Manager

First, deploy the Nginx Proxy Manager:

```bash
docker compose -f nginx-proxy-manager.yml up -d
```

Wait for it to start (about 30 seconds), then access the admin interface at `http://your-server-ip:81`

**Default credentials:**
- Email: `admin@example.com`
- Password: `changeme`

**Important**: Change these credentials immediately after first login!

### Step 5: Deploy Media Stack

```bash
docker compose -f media-stack-gluetun.yml up -d
```

### Step 6: Verify All Services Are Running

```bash
docker ps
```

You should see all containers running. Check logs if any container is restarting:

```bash
docker logs <container_name>
```

## 🌐 Nginx Proxy Manager Setup

Nginx Proxy Manager allows you to route your custom domain to Jellyfin and other services with automatic SSL certificates.

### Initial Setup

1. Access NPM admin UI at `http://your-server-ip:81`
2. Login with default credentials (or your new credentials)
3. Navigate to **Settings** → **Default Site** and configure as needed

### Configure Router Port Forwarding

Forward these ports from your router to your Nginx Proxy Manager (NPM):
- Port **80** (HTTP) → NPM IP:80
- Port **443** (HTTPS) → NPM IP:443

**Note:** The NPM IP is typically the same as your server's local IP address where the Nginx Proxy Manager container is running.

### DNS Configuration

At your domain registrar (e.g., Cloudflare, Namecheap, GoDaddy):

1. Create **A records** pointing to your public IP for Jellyfin and Jellyseerr:
   ```
   jellyfin.yourdomain.com → Your_Public_IP
   requests.yourdomain.com → Your_Public_IP
   ```

2. Wait for DNS propagation (can take 5 minutes to 48 hours)

**Note:** We're only setting up public domains for Jellyfin and Jellyseerr. Other services (Sonarr, Radarr, Prowlarr, qBittorrent) should remain on your local network for security reasons and can be accessed via their local ports.

### Add Proxy Host for Jellyfin

1. In Nginx Proxy Manager, click **Proxy Hosts** → **Add Proxy Host**

2. **Details Tab:**
   - **Domain Names:** `jellyfin.yourdomain.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `jellyfin` (container name) or `your-server-ip`
   - **Forward Port:** `8096`
   - **Cache Assets:** ✓ (enabled)
   - **Block Common Exploits:** ✓ (enabled)
   - **Websockets Support:** ✓ (enabled - important for Jellyfin!)

3. **SSL Tab:**
   - **SSL Certificate:** Select "Request a new SSL Certificate"
   - **Force SSL:** ✓ (enabled)
   - **HTTP/2 Support:** ✓ (enabled)
   - **HSTS Enabled:** ✓ (enabled)
   - **Email Address:** Your email for Let's Encrypt notifications
   - **I Agree to the Let's Encrypt Terms of Service:** ✓

4. **Click Save**

NPM will automatically request an SSL certificate from Let's Encrypt and configure HTTPS.

### Add Proxy Host for Jellyseerr

Repeat the same process for Jellyseerr (Media Requests):

1. In Nginx Proxy Manager, click **Proxy Hosts** → **Add Proxy Host**

2. **Details Tab:**
   - **Domain Names:** `requests.yourdomain.com`
   - **Scheme:** `http`
   - **Forward Hostname/IP:** `jellyseerr` (container name)
   - **Forward Port:** `5055`
   - **Cache Assets:** ✓ (enabled)
   - **Block Common Exploits:** ✓ (enabled)
   - **Websockets Support:** ✓ (enabled)

3. **SSL Tab:**
   - **SSL Certificate:** Select "Request a new SSL Certificate"
   - **Force SSL:** ✓ (enabled)
   - **HTTP/2 Support:** ✓ (enabled)
   - **HSTS Enabled:** ✓ (enabled)
   - **Email Address:** Your email for Let's Encrypt notifications
   - **I Agree to the Let's Encrypt Terms of Service:** ✓

4. **Click Save**

**Accessing Other Services:**
Other services like Sonarr, Radarr, Prowlarr, and qBittorrent should be accessed locally via their ports for security:
- Sonarr: `http://your-server-ip:8989`
- Radarr: `http://your-server-ip:7878`
- Prowlarr: `http://your-server-ip:9696`
- qBittorrent: `http://your-server-ip:8080`

### Advanced: Custom Nginx Configuration

For Jellyfin, you may want to add custom Nginx configuration for better performance:

1. In the Proxy Host settings, go to the **Advanced** tab
2. Add this configuration:

```nginx
# Disable buffering when the nginx proxy gets very resource heavy upon streaming
proxy_buffering off;

# Client request size limit
client_max_body_size 20M;
```

## ⚙️ Service Configuration

### Jellyfin Setup

1. Access Jellyfin: `https://jellyfin.yourdomain.com`
2. Complete the initial setup wizard:
   - Set admin username and password
   - Add media libraries pointing to `/media/movies` and `/media/tv`
   - Configure hardware acceleration (VA-API is already configured)

3. Update the published server URL:
   - Go to **Dashboard** → **Networking**
   - Set **Published Server URL** to: `https://jellyfin.yourdomain.com`
   - Save changes

### qBittorrent Setup

1. Access qBittorrent at `http://your-server-ip:8080`
2. Default credentials:
   - Username: `admin`
   - Password: `adminadmin`
3. Change password immediately!
4. Configure download paths:
   - Navigate to **Settings** → **Downloads**
   - Default Save Path: `/data/downloads/complete`
   - Temp Path: `/data/downloads/incomplete`

### Prowlarr Setup

1. Access Prowlarr at `http://your-server-ip:9696`
2. Add indexers (trackers)
3. Connect to Sonarr and Radarr:
   - Go to **Settings** → **Apps**
   - Add Sonarr (URL: `http://sonarr:8989`)
   - Add Radarr (URL: `http://radarr:7878`)
   - Get API keys from each service's Settings → General

### Sonarr Setup

1. Access Sonarr at `http://your-server-ip:8989`
2. Add root folder: `/data/tv`
3. Connect download client (qBittorrent):
   - Settings → Download Clients → Add → qBittorrent
   - Host: `gluetun` (since qBittorrent uses its network)
   - Port: `8080`
4. Configure media management settings

### Radarr Setup

1. Access Radarr at `http://your-server-ip:7878`
2. Add root folder: `/data/movies`
3. Connect download client (qBittorrent):
   - Settings → Download Clients → Add → qBittorrent
   - Host: `gluetun`
   - Port: `8080`
4. Configure media management settings

### Jellyseerr Setup

1. Access Jellyseerr at `http://your-server-ip:5055`
2. Connect to Jellyfin:
   - URL: `http://jellyfin:8096`
   - Use your Jellyfin admin credentials
3. Connect to Sonarr and Radarr with their API keys
4. Configure request settings and permissions

## 🔒 Security Considerations

### VPN Security
- **Never expose your real IP**: Always verify qBittorrent traffic goes through VPN
- **Test the killswitch**: Stop the VPN container and verify qBittorrent loses connectivity
- **Use port forwarding**: Enable in ProtonVPN settings for better connectivity

### Password Security
- Change all default passwords immediately
- Use strong, unique passwords for each service
- Consider using a password manager

### Network Security
- Don't expose service ports directly to the internet (except 80, 443 via NPM)
- Use NPM Access Lists to restrict admin interfaces
- Enable 2FA where available
- Keep all containers updated regularly

### API Key Protection
- Treat API keys like passwords
- Don't commit them to Git repositories
- Regenerate if compromised

### Environment File Security
- The `stack.env` file contains sensitive credentials (VPN username/password, etc.)
- This file is in `.gitignore` to prevent accidental commits
- **Important**: If `stack.env` was previously tracked by Git, you should remove it from tracking:
  ```bash
  git rm --cached stack.env
  git commit -m "Remove stack.env from tracking"
  ```
- Alternatively, use `git-crypt` or similar tools to encrypt sensitive files in your repository
- Always use `stack.env.example` as a template and never commit actual credentials

### SSL/TLS
- Always enable "Force SSL" in NPM
- Enable HSTS for additional security
- Let's Encrypt certificates auto-renew every 60 days

### Firewall Configuration

Configure UFW (if using Ubuntu):

```bash
sudo ufw allow 22/tcp      # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS
sudo ufw enable
```

## 🔧 Troubleshooting

### Jellyfin Can't Find Media
- Verify directory permissions: `ls -la /mnt/docker/media`
- Check that PUID/PGID match your user: `id`
- Verify the media files are in the correct directories
- Scan libraries in Jellyfin dashboard

### qBittorrent Won't Start
- Check Gluetun logs: `docker logs gluetun`
- Verify VPN credentials are correct
- Ensure `/dev/net/tun` exists: `ls -la /dev/net/tun`
- Try a different ProtonVPN server

### SSL Certificate Issues
- Verify ports 80 and 443 are open and forwarded
- Check DNS propagation: `nslookup jellyfin.yourdomain.com`
- Review NPM logs: `docker logs npm`
- Ensure email address is valid in Let's Encrypt settings

### Services Can't Communicate
- Verify all containers are on `media-net` network: `docker network inspect media-net`
- Recreate the network if needed
- Check container logs for connection errors

### VPN Connection Fails
- Verify ProtonVPN credentials
- Check if VPN server supports OpenVPN
- Review Gluetun logs: `docker logs gluetun`
- Try a different server or VPN provider

### Performance Issues
- Check CPU/RAM usage: `docker stats`
- Verify hardware acceleration is working in Jellyfin
- Consider adding more RAM or CPU cores
- Check disk I/O performance

### Access Denied / Permission Errors
- Verify file ownership: `sudo chown -R 1000:1000 /mnt/docker`
- Check PUID/PGID settings in docker-compose files
- Ensure UMASK is set correctly (002)

## 📝 Useful Commands

### Container Management

```bash
# View all running containers
docker ps

# View all containers (including stopped)
docker ps -a

# View logs for a specific container
docker logs -f <container_name>

# Restart a specific container
docker restart <container_name>

# Stop all services
docker compose -f media-stack-gluetun.yml down

# Start all services
docker compose -f media-stack-gluetun.yml up -d

# Pull latest images
docker compose -f media-stack-gluetun.yml pull

# Update and restart services
docker compose -f media-stack-gluetun.yml pull && docker compose -f media-stack-gluetun.yml up -d
```

### Network Management

```bash
# List all Docker networks
docker network ls

# Inspect media-net network
docker network inspect media-net

# Remove and recreate network (stop services first)
docker network rm media-net
docker network create media-net
```

### Volume Management

```bash
# List all volumes
docker volume ls

# Inspect a specific volume
docker volume inspect <volume_name>

# Clean up unused volumes
docker volume prune
```

### System Maintenance

```bash
# Check disk usage by Docker
docker system df

# Clean up unused images, containers, and networks
docker system prune -a

# View resource usage
docker stats

# Check VPN status
docker exec gluetun wget -qO- https://api.ipify.org
```

### Backup Configuration

```bash
# Backup all configuration directories
sudo tar -czf jellyfin-backup-$(date +%Y%m%d).tar.gz /mnt/docker/

# Exclude media files from backup (config only)
sudo tar -czf config-backup-$(date +%Y%m%d).tar.gz \
  --exclude=/mnt/docker/media \
  --exclude=/mnt/docker/jellyfin/cache \
  /mnt/docker/
```

### Health Checks

```bash
# Test if services are reachable
curl -I http://localhost:8096  # Jellyfin
curl -I http://localhost:8989  # Sonarr
curl -I http://localhost:7878  # Radarr
curl -I http://localhost:9696  # Prowlarr
curl -I http://localhost:8080  # qBittorrent
curl -I http://localhost:5055  # Jellyseerr

# Check VPN IP (should not be your real IP)
docker exec qbittorrent wget -qO- https://api.ipify.org
```

## 📚 Additional Resources

- [Jellyfin Documentation](https://jellyfin.org/docs/)
- [Sonarr Wiki](https://wiki.servarr.com/sonarr)
- [Radarr Wiki](https://wiki.servarr.com/radarr)
- [Prowlarr Wiki](https://wiki.servarr.com/prowlarr)
- [Nginx Proxy Manager Docs](https://nginxproxymanager.com/guide/)
- [Gluetun Wiki](https://github.com/qdm12/gluetun-wiki)
- [Docker Documentation](https://docs.docker.com/)

## 🤝 Contributing

Feel free to submit issues or pull requests to improve this setup!

## 📄 License

This configuration is provided as-is for personal use. Ensure you comply with your local laws regarding media content and VPN usage.

---

**Note**: This setup is intended for personal use and legal content only. Always respect copyright laws and use this responsibly.
