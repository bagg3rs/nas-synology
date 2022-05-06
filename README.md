# nas-synology
Things I did with my Synology NAS

## Docker

### Run docker without Sudo
Rather than Sudo'ing this will allow docker commands to be run without envoking `sudo`
1. Create a docker user group `sudo synogroup --add docker`
2. Change the owner group of the docker.sock file `sudo chown root:docker /var/run/docker.sock`
3. Add your user to the new docker group `sudo synogroup --member docker $USER`
