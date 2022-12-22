# Configure AWS Cloud9 & GitHub
- `git config --global user.name "<USERNAME>"`
- `git config --global user.email "<EMAIL_ADDRESS>"`
- `ssh-keygen -t rsa`
- Go to GitHub > Code > SSH > Create Public keygen
- `cat /home/ec2-user/.ssh/id_rsa.pub` > Paste this into GitHub Public Key
- `eval $(ssh-agent -s)`
- `ssh-add /home/ec2-user/.ssh/id_rsa`
- `git clone git@github.com:binbashburns/docker.git`

# Install Tree (ALWAYS)
- `sudo yum install tree`
- `sudo tree -L 1 /var/lib/docker`
- `sudo tree -L 2 /var/lib/docker`
- `sudo tree -L 3 /var/lib/docker`

# Docker Basics
- `docker --version`
- `docker version`
- `docker info`
- `docker -h`
- `docker search python`
- `docker search --filter is-official=true python`
- `docker pull public.ecr.aws/docker/library/python:latest`
- `docker images`
- `docker history <IMAGE_ID>`
- `docker create python` 
- `docker ps- a`
- `docker start <CONTAINER_NAME>`
- `docker rename <OLD_CONTAINER_NAME> <NEW_CONTAINER_NAME>`

## Docker Run
- `docker run python python -c 'print("docker build your skills!")'`
- `docker run python python --version`
- `docker run python:latest python --version`
- `docker run python:3.8 python --version`
- `docker run -it python`: Creates dynamically named interactive python terminal.
- `docker run -it --name py_container_name python`: Creates interactive python container named "py_container_name" using defaults.
- `docker run --name python3 -itd python`: Runs in detached mode, which means the interactive terminal exists, you're just not in it.
- `docker exec -it python3 /bin/sh`: logs into the bash shell of the python container.
- `docker attach <CONTAINER_NAME>`: Drops you into an interactive terminal on a running container. But once you exit, the container dies.
- `docker attach --detach-keys="ctrl-b" python3`: Drops you into an interactive terminal, and allows you to use CTRL+B to escape it, without killing the container.

## Docker Inspect
- `sudo yum install jq -y`: This will make it easier to read the output of Docker Inspect.
- `docker inspect --help`
- `docker ps -a`: See what containers you have to pick from. I'll use `python3` for the following.
- `docker inspect -s python3`: Displays total file sizes if the type is container.
- `docker inspect -s python3 | jq`: Pipe it to JQ to make it easier to read.
- `docker inspect -s python3 | jq .[0] | jq keys`: This is the information you can get at the first level of your Docker Inspect command
- `docker inspect -s python3 | jq .[0].ResolvConfPath`: This extracts the value of the `ResolvConfPath` key in layer 1 (`[0]`), in quotes
- `docker inspect -s python3 | jq -r .[0].ResolvConfPath`: Same as previous, but strips the quotes.
- `sudo cat $(docker inspect -s python3 | jq -r .[0].ResolvConfPath)`: Concatenates the value of `./resolve.conf` 

## Docker ps and filtering
- Docker Docs: https://docs.docker.com/engine/reference/commandline/ps/
- `docker ps -a`: Show all containers
- `docker rm <CONTAINER_NAME>`: Delete a container
- `docker rm $(docker ps -a -q -f status=exited)`: Delete all Docker containers where STATUS=exited
- `docker ps -a -f ancestor=python`: Present all Docker containers whos parent image is `python`
- `docker ps -a --filter before=$(docker ps -a -l --format {{.Names}})`: Presents all Docker images BESIDES the latest container image (`-l` means latest)
- `docker rm $(docker ps -a -q --filter before=$(docker ps -a -l --format {{.Names}}))`: Deletes all Docker images BESIDES the latest container
- `docker rmi -f $(docker images -q)`: Delete all Docker images
- 