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

## Working with Prebuilt Docker Images
- Explore Docker Hub
- Get and View httpd
- Run the website in httpd
- Get and View Nginx
- Run the website in Nginx
```
# Explore Docker Hub
https://hub.docker.com/

# Get and View httpd
docker pull httpd:latest
docker run --name httpd -p 8080:80 -d httpd:latest
http://[public-ip]:8080

# Run the website in httpd
git clone https://github.com/linuxacademy/content-widget-factory-inc.git
docker stop httpd
docker rm httpd
docker run --name httpd -p 8080:80 -v $(pwd)/web:/usr/local/apache2/htdocs:ro -d httpd:latest

# Get and View Nginx
docker pull nginx:latest
docker run --name nginx -p 8081:80 -d nginx:latest
http://[public-ip]:8081

# Run the website in Nginx
git clone https://github.com/linuxacademy/content-widget-factory-inc.git
docker stop nginx
docker rm nginx
docker run --name nginx -v $(pwd)/web:/usr/share/nginx/html:ro -p 8081:80 -d nginx


```

## Handcrafting a Container Image
- Get and Run the Base Image
- Add the Website to the Template
- Create an Image from the Template
- Clean up the Template for a Second Version
- Run Multiple Containers from the Image
```
# Get and Run the Base Image
docker pull httpd:latest
docker run --name webtemplate -d httpd:latest

# Add the Website to the Template
docker exec -it webtemplate bash
apt update && apt install git -y
git clone https://github.com/linuxacademy/content-widget-factory-inc.git /tmp/widget-factory-inc
rm htdocs/index.html
cp -r /tmp/widget-factory-inc/web/* htdocs/
exit

# Create an Image from the Template
docker ps
docker commit 574d4fc7e46d example/widgetfactory:v1

# Clean up the Template for a Second Version
docker exec -it webtemplate bash
rm -rf /tmp/widget-factory-inc/
apt remove git -y && apt autoremove -y && apt clean
exit
docker ps
docker commit 574d4fc7e46d example/widgetfactory:v2

# Run Multiple Containers from the Image
docker run -d --name web1 -p 8081:80 example/widgetfactory:v2
docker run -d --name web2 -p 8082:80 example/widgetfactory:v2
docker run -d --name web3 -p 8083:80 example/widgetfactory:v2
```

## Storing Container Data In Docker Volumes
- Discover Anonymous Docker Volumes
- Create a Docker Volume
- Use the Website Volume with Containers
- Clean up Unused Volumes
- Back up and Restore the Docker Volume
```
# Discover Anonymous Docker Volumes
docker volume ls
docker run -d --name db1 postgres:12.1
docker run -d --name db2 postgres:12.1
docker inspect db1 -f '{{ json .Mounts }}' | python -m json.tool
docker inspect db2 -f '{{ json .Mounts }}' | python -m json.tool
docker run -d --rm --name dbTmp postgres:12.1
docker volume ls
docker inspect dbTmp
docker stop db2 dbTmp
docker ps

# Create a Docker Volume
docker volume create website
docker volume ls

# Use the Website Volume with Containers
sudo cp -r /home/cloud_user/widget-factory-inc/web/* /var/lib/docker/volumes/website/_data
sudo ls -l /var/lib/docker/volumes/website/_data/
ll widget-factory-inc/web/
docker run -d --name web1 -p 80:80 -v website:/usr/local/apache2/htdocs:ro httpd:2.4
curl 54.91.249.112:80
docker run -d --name webTmp --rm -v website:/usr/local/apache2/htdocs:ro httpd:2.4
docker exec -it webTmp bash
ls -l htdocs
exit
docker stop webTmp
docker ps
docker volume ls

# Clean up Unused Volumes
docker rm db2
docker volume prune
docker volume ls

# Back up the Docker Volume
sudo su -
docker volume inspect website
tar czf /tmp/website_$(date +%Y-%m-d-%H%M).tgz -C /var/lib/docker/volumes/website/_data .
ls -l /tmp/website_*.tgz
tar tf /tmp/website_2022-02-d-0810.tgz
exit
docker run -it --rm -v website:/website -v /tmp:/backup bash tar czf /backup/website_$(date +%Y-%m-d-%H%M).tgz -C /website .
ls -l /tmp/website_*
tar tf /tmp/website_2022-02-d-0815.tgz

# Restore the Docker Volume
sudo su -
cd /var/lib/docker/volumes
cd website/_data/
rm * -rf
tar xf /tmp/website_2022-02-d-0815.tgz .

```