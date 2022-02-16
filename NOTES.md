# Docker Notes

## Initializing the Docker Environment
- Install Docker
- Enable the Docker Daemon
- Configure user permissions
- Pull a container from Docker hub
```
# Install Docker
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum -y install docker-ce

# Enable the Docker Daemon
sudo systemctl enable --now docker

# Configure user permissions
sudo usermod -aG docker <username>

# Pull a container from Docker hub
docker run hello-world
```

