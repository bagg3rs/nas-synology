# nas-synology
Things I did with my Synology NAS

## Docker

### Run docker without Sudo
Rather than Sudo'ing this will allow docker commands to be run without envoking `sudo`
1. Create a docker user group `sudo synogroup --add docker`
2. Change the owner group of the docker.sock file `sudo chown root:docker /var/run/docker.sock`
3. Add your user to the new docker group `sudo synogroup --member docker $USER`

### Create network for Traefik

```
docker network create --driver=macvlan \

 --gateway=192.168.0.254 --subnet=192.168.0.0/24 --ip-range=192.168.0.0/26 \
 
 -o parent=ovs_eth0 -o macvlan_mode=bridge vlan_home
```
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
`sudo apt update`
`sudo apt install nfs-common`
2. Create the mount directory – change the path as appropriate:
`sudo mkdir /mnt/backup`

3. Test access – `replace IP address/ hostname with your details`
`sudo mount -t nfs4 -o proto=tcp,port=2049 192.168.XX.XX:/volume1/backup /mnt/backup`
4. Create persistent mount (e.g. will be around after reboot)
`sudo nano /etc/fstab`
Add this line to the bottom, again replacing ip addresses / hostname and file paths as needed:
`192.168.XX.XX:/volume1/backup /mnt/share nfs defaults 0 0`
Exit, save the file, and reboot the pi (sudo reboot now).
Once the Pi is back up, ensure the mount worked with 
`ls /mnt/backup`
