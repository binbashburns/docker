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

## Flask Stuff
- `cd dev1`: Move into `dev1`
- `docker run -it python /bin/sh`: Open interactive Python shell
- We want access to this `/app` directory from within the Python container. To do that we'll use the `-v` flag
- `docker run -it -v $PWD/app:/app python /bin/sh`: Starts interactive Python container, mounts the `/app` directory to `/app` in the container
- `cd app`: From the interactive Docker container, move into the mounted folder. 
- Now when you modify the code in the `app` folder in the Docker mounted directory, it modifies the files in the actual directory on the host.
- `pip3 install -r requirements.txt`: This installs everything defined in the `requirements.txt` file
- `python app.py`: Starts flask server. You can now navigate to it using the public IP
- This method of starting the Flask server is not ideal. We don't want to have to start it manually. 
- Let's make a new file in the `/app` directory and call it `run.sh`
- `chmod +x run.sh`
- `docker run -it -v $PWD/app:/app -w /app python /bin/sh run.sh`: Run this from `/docker/mtc-docker/infrastructure/dev1`
- `docker inspect $(docker ps -q) | jq -r .[0].NetworkSettings.IPAddress`: Gathers internal IP address of the Docker container running flask
- Open a new terminal, then run
- `curl http://172.17.0.2:5000`
- You should see something output like this:
```
{
  "data": {
    "name": "0f9156ced9e9", 
    "temp": 11
  }
}
```
- You could've also chained these two together with: 
- `curl http://$(docker inspect $(docker ps -q) | jq -r .[0].NetworkSettings.IPAddress):5000`

## Docker Ports and Detaching
- `docker run -d -v $PWD/app:/app -w /app python /bin/sh run.sh`: Runs locally, and the container is only accessible from the host
- `docker run -d --publish 5000 -v $PWD/app:/app -w /app python /bin/sh run.sh`: Exposes port 5000 **ON THE CONTAINER** to be accessible from the host IP (either the IPv4 address or `localhost`)
- `docker ps -a`: See which port is open on the localhost. (Example: `0.0.0.0:49153->5000/tcp`)
- Open a browser and navigate to `http://<PUBLIC_IP>:49153`, for example
- `docker run -dp 5000:5000 -v $PWD/app:/app -w /app python /bin/sh run.sh`: Exposes port 5000 **ON THE CONTAINER** and maps it to port 5000 **ON THE HOST**
- You can now navigate to `http://<PUBLIC_IP>:5000

## Overlay2 Filesystem Deep Dive
- `docker pull busybox`: Busy Box is a great candidate for small tasks (pinging something, troubleshooting)
- `docker inspect busybox`: Upon observation, filesystem layers are located in `/var/lib/overlay2/` (see `GraphDriver` > `Data` > *)
- `sudo su`
- `cd /var/lib/docker/overlay2`
- `tree --inodes -L 3`: View 3 layers into `overlay2` directory to understand file structure.
- `docker run -v /data -d busybox ping google.com`: Run a busybox container, add the /data directory to the container, `-d` for detached, and pinging Google.com

## Dockerfile
- Dockerfile docs: https://docs.docker.com/engine/reference/builder/
- Instead of typing `docker run -dp 5000:5000 -v $PWD/app:/app -w /app python /bin/sh run.sh`, use a Dockerfile
-  In the Dockerfile, use this text:
```
 FROM python
 
 COPY ./app/ /app
 
 WORKDIR /app
 
 RUN pip install -r requirements.txt
 
 EXPOSE 5000
 
 ENTRYPOINT [ "python", app.py ]
```
- Now go to the `.../mtc-docker/infrastructure/dev1` directory and run 
- `docker build -t dev1 .`: Builds the Docker container using the specifications defined in the `Dockerfile`