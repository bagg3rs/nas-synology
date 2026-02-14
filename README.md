# nas-synology
Docker Compose stack for Synology NAS with Traefik reverse proxy, VPN, download clients, and various utilities.

## 📋 Prerequisites

- Synology NAS (DSM 7.0+)
- Docker package installed from Package Center
- SSH access enabled (for network setup)
- (Optional) Domain name with Cloudflare DNS

## 🚀 Quick Start

### 1. Enable Docker Without Sudo

Rather than using `sudo` for every command:

1. Create a docker user group:
   ```bash
   sudo synogroup --add docker
   ```

2. Change the owner group of the docker.sock file:
   ```bash
   sudo chown root:docker /var/run/docker.sock
   ```

3. Add your user to the new docker group:
   ```bash
   sudo synogroup --member docker $USER
   ```

4. Log out and back in for changes to take effect

### 2. Create MacVLAN Network

**Important:** Use `eth0` as the parent interface (not `ovs_eth0`):

```bash
docker network create --driver=macvlan \
  --gateway=192.168.0.254 --subnet=192.168.0.0/24 --ip-range=192.168.0.0/26 \
  -o parent=eth0 -o macvlan_mode=bridge vlan_home
```

**Note:** Adjust `gateway`, `subnet`, and `ip-range` to match your network configuration.

### 3. Set Up Automatic Network Creation

The MacVLAN network gets deleted after Synology reboots. Create an automatic startup script:

1. **Create startup script:**
   ```bash
   cat > /volume1/docker/startup.sh << 'EOF'
   #!/bin/bash
   # MacVLAN network startup script for Synology NAS
   
   # Wait for network to be ready
   sleep 30
   
   # Create MacVLAN network if it doesn't exist
   if ! /usr/local/bin/docker network ls | grep -q vlan_home; then
       echo "Creating MacVLAN network..."
       /usr/local/bin/docker network create --driver=macvlan \
           --gateway=192.168.0.254 \
           --subnet=192.168.0.0/24 \
           --ip-range=192.168.0.0/26 \
           -o parent=eth0 \
           -o macvlan_mode=bridge \
           vlan_home
       echo "MacVLAN network created"
   else
       echo "MacVLAN network already exists"
   fi
   
   # Start docker-compose stack
   cd /volume1/docker/nas-synology
   /usr/local/bin/docker-compose up -d
   
   echo "Startup script completed"
   EOF
   
   chmod +x /volume1/docker/startup.sh
   ```

2. **Set up Task Scheduler:**
   - DSM > Control Panel > Task Scheduler
   - Create > Triggered Task > User-defined script
   - **Name:** "Docker Network Startup"
   - **User:** root
   - **Schedule:** Boot-up
   - **Run command:** `/volume1/docker/startup.sh`

### 4. Clone Repository

```bash
cd /volume1/docker
git clone https://github.com/bagg3rs/nas-synology.git
cd nas-synology
```

### 5. Configure Environment Variables

Copy the example environment file:

```bash
cp .env.example .env
```

Edit `.env` and configure:

**Required:**
- `PUID` / `PGID` - Your user/group ID (find with `id <username>`)
- `TZ` - Your timezone (e.g., `Europe/London`)
- `IPv4_PREFIX` - Your network prefix (e.g., `192.168.0`)

**If using Traefik with Let's Encrypt:**
- `CF_DNS_API_TOKEN` - Cloudflare DNS API token
- `CF_ZONE_API_TOKEN` - Cloudflare Zone API token
- `EMAIL` - Your email for Let's Encrypt
- `TRAEFIKUSER` - Dashboard credentials (generate with `htpasswd -nb admin password`)

**Domain names** (if using Traefik):
- `TRAEFIK`, `DELUGE`, `PROWLARR`, etc. - Configure your domains

### 6. Configure VPN (Optional)

If using the VPN container:

1. Place your OpenVPN config files in `/volume1/docker/config/vpn/`
2. Adjust the network range in `docker-compose.yml` if needed:
   ```yaml
   command: '-f "" -r "YOUR_NETWORK/24"'
   ```

### 7. Start the Stack

```bash
docker-compose up -d
```

## 📦 Services Included

- **Traefik** - Reverse proxy with automatic HTTPS
- **VPN** - OpenVPN client for routing specific containers
- **LibreSpeed** - Network speed test
- **Heimdall** - Application dashboard
- **Uptime Kuma** - Uptime monitoring
- **Portainer** - Docker management UI
- **Code Server** - VS Code in browser

## 🌐 Accessing Services

With Traefik domains configured (HTTPS):
- Traefik Dashboard: `https://traefik.yourdomain.com`
- Portainer: `https://portainer.yourdomain.com`
- Uptime Kuma: `https://uptime.yourdomain.com`
- And others as configured...

## 🔧 Configuration

### External Networks

When using pre-created MacVLAN networks, ensure your `docker-compose.yml` uses external networks:

```yaml
networks:
  vlan_home:
    external: true  # Use existing network instead of creating new one
```

### OpenVPN TUN Device Setup

If you get `ERROR: Cannot open TUN/TAP dev /dev/net/tun: No such device` (especially after system upgrades):

```bash
#!/bin/sh

# Create the necessary file structure for /dev/net/tun
if ( [ ! -c /dev/net/tun ] ); then
  if ( [ ! -d /dev/net ] ); then
    mkdir -m 755 /dev/net
  fi
  mknod /dev/net/tun c 10 200
  chmod 0755 /dev/net/tun
fi

# Load the tun module if not already loaded
if ( !(lsmod | grep -q "^tun\s") ); then
  insmod /lib/modules/tun.ko
fi
```

Add this to Task Scheduler as a boot-up script.

### Resource Limits

Add resource limits to prevent containers from consuming unlimited resources:

```yaml
services:
  service_name:
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '0.5'
        reservations:
          memory: 256M
    logging:
      driver: json-file
      options:
        max-size: 10m
        max-file: 3
```

**Recommended limits:**
- **Media servers**: 2-4GB RAM, 1.0-2.0 CPU  
- **VPN/Proxy** (Traefik, VPN): 256-512MB RAM, 0.25-0.5 CPU
- **Utilities** (Portainer, monitoring): 128-256MB RAM, 0.1-0.25 CPU

## 💾 Backup Raspberry Pi to NFS Share

### Enable NFS on Synology

1. DSM > Control Panel > File Services > NFS
2. Check **Enable NFS Service**
3. Set **Maximum NFS Protocol** to NFSv4.1
4. Click **Apply**

### Create Share and Allow Access

1. Control Panel > Shared Folder
2. Select folder > **Edit**
3. Click **NFS Permissions** > **Create**
4. Enter Pi hostname or IP
5. Set **Privilege** to **Read / Write**
6. Note the **Mount path** at bottom
7. Click **Save**

### Mount NFS Share on Raspberry Pi

1. Install NFS client (if not installed):
   ```bash
   sudo apt update && sudo apt install nfs-common
   ```

2. Create mount directory:
   ```bash
   sudo mkdir /mnt/backup
   ```

3. Test access (replace with your details):
   ```bash
   sudo mount -t nfs -o proto=tcp,port=2049 192.168.XX.XX:/volume1/backup /mnt/backup
   ```

4. Create persistent mount (survives reboot):
   ```bash
   sudo nano /etc/fstab
   ```
   
   Add this line (adjust IP and paths):
   ```
   192.168.XX.XX:/volume1/backup /mnt/backup nfs defaults 0 0
   ```

5. Test fstab:
   ```bash
   sudo mount -a
   ls /mnt/backup
   ```

## 🐛 Troubleshooting

### Common Issues

**"Download stack partially down"**
- Usually caused by MacVLAN network missing after reboot
- Solution: Use the automatic startup script (see step 3 above)

**"Pool overlaps with other one on this address space"**
- Docker trying to create conflicting networks
- Solution: Use `external: true` in docker-compose.yml networks section

**VPN containers failing to start**
- Check TUN device exists: `ls -la /dev/net/tun`
- Check TUN module loaded: `lsmod | grep tun`
- Recreate TUN device (see OpenVPN section above)

**"Permission denied" on volumes**
- Check PUID/PGID in `.env` matches your user
- Verify folder permissions: `ls -la /volume1/docker/config`

**Traefik not getting certificates**
- Verify Cloudflare API tokens are correct
- Check DNS records exist for your domains
- Try Let's Encrypt staging first (uncomment staging line)
- Check logs: `docker-compose logs -f traefik`

**Docker commands require sudo**
- Ensure you followed "Enable Docker Without Sudo" steps
- Log out and back in after adding user to docker group

## 🔒 Security Notes

1. **Never commit `.env`** - Contains sensitive tokens and passwords
2. **Change default passwords** - Set strong passwords for all services
3. **Use Traefik authentication** - Protect services with basic auth or SSO
4. **Keep tokens secure** - Use `.env` file, never hardcode in compose file
5. **Regular updates** - Keep containers updated with `docker-compose pull`
6. **Firewall rules** - Only expose necessary ports to internet

## 📁 Directory Structure

```
/volume1/docker/
├── nas-synology/
│   ├── docker-compose.yml    # Main compose file
│   ├── .env                   # Your environment variables (DO NOT COMMIT)
│   └── .env.example           # Template for .env
└── config/                    # Service configuration volumes
    ├── vpn/                   # OpenVPN configs
    ├── heimdall/
    ├── uptime-kuma/
    ├── portainer/
    └── code-server/
```

## 📝 License

MIT
