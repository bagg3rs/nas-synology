# nas-synology
Things I did with my Synology NAS

## Docker

### Run docker without Sudo
Rather than Sudo'ing this will allow docker commands to be run without envoking `sudo`
1. Create a docker user group `sudo synogroup --add docker`
2. Change the owner group of the docker.sock file `sudo chown root:docker /var/run/docker.sock`
3. Add your user to the new docker group `sudo synogroup --member docker $USER`

### Create network for Traefik

**Important:** Use `eth0` as the parent interface (not `ovs_eth0`):

```bash
docker network create --driver=macvlan \
  --gateway=192.168.0.254 --subnet=192.168.0.0/24 --ip-range=192.168.0.0/26 \
  -o parent=eth0 -o macvlan_mode=bridge vlan_home
```

### Automatic Network Creation on Startup

The MacVLAN network gets deleted after Synology reboots, causing Docker containers to fail. Create an automatic startup script:

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

# Start down-things stack
cd /volume1/docker/down-things
/usr/local/bin/docker-compose up -d

echo "Startup script completed"
EOF

chmod +x /volume1/docker/startup.sh
```

2. **Set up Task Scheduler:**
   - DSM > Control Panel > Task Scheduler
   - Create > Triggered Task > User-defined script
   - Name: "Docker Network Startup"
   - User: root
   - Schedule: Boot-up
   - Run command: `/volume1/docker/startup.sh`

### Docker Compose External Networks

When using pre-created MacVLAN networks, configure your `docker-compose.yml` to use external networks:

```yaml
networks:
  vlan_home:
    external: true  # Use existing network instead of creating new one
```

### Running OpenVPN

1. Install VPN Server from package manager
2. After an upgrade you may need to run docker-compose from ssh session.

If you ever get the `ERROR: Cannot open TUN/TAP dev /dev/net/tun: No such device` especially after a system upgrade then run or schedule the following

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

### Resource Management

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
- **Media servers** (jellyfin, plex): 2-4GB RAM, 1.0-2.0 CPU  
- **VPN/Proxy** (traefik, vpn): 256-512MB RAM, 0.25-0.5 CPU
- **Utilities** (portainer, monitoring): 128-256MB RAM, 0.1-0.25 CPU

## Backup Pi's 🥧 to NFS share
I want to backup database and configuration files to my NAS.

### Enable NFS on the Synology NAS

1. From the Synology web interface, Go to Control Panel > File Services > NFS

2. Check Enable NFS Service. Set Maximum NFS Protocol to NFSv4.1 and click Apply

### Create Backup and share allow IP of Pi's

1. On Synology web interface, go to Control Panel > Shared Folder
2. Select the folder you want to share and click Edit
Click NFS Permissions > Create
3. Enter a hostname or IP for the Raspberry Pi that will be accessing the folder.
4. Leave Privilege on Read / Write and leave other settings at default, and click Save.

`Note the Mount path at the bottom of the folder edit page, then click Save again to save the folder settings.`

### Mount NFS Share on Pi's

1. From the terminal – install nfs if not already installed:
`sudo apt update && sudo apt install nfs-common`
2. Create the mount directory – change the path as appropriate:
`sudo mkdir /mnt/backup`

3. Test access – `replace IP address/ hostname with your details`
`sudo mount -t nfs -o proto=tcp,port=2049 192.168.XX.XX:/volume1/backup /mnt/backup`
4. Create persistent mount (e.g. will be around after reboot)
`sudo nano /etc/fstab`
Add this line to the bottom, again replacing ip addresses / hostname and file paths as needed:
`192.168.XX.XX:/volume1/backup /mnt/backup nfs defaults 0 0`
Exit, save the file.
5. Test fstab by running `mount -a` and then `ls /mnt/backup`

## Troubleshooting

### Common Issues

**"Download stack partially down"**
- Usually caused by MacVLAN network missing after reboot
- Solution: Use the automatic startup script above

**"Pool overlaps with other one on this address space"**
- Docker trying to create conflicting networks
- Solution: Use `external: true` in docker-compose.yml networks section

**VPN containers failing to start**
- Check TUN device exists: `ls -la /dev/net/tun`
- Check TUN module loaded: `lsmod | grep tun`
- Recreate TUN device if needed (see OpenVPN section above)
