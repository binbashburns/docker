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
- Another example:
```
FROM Ubuntu

RUN apt-get update && apt-get install python

RUN pip install flask flask-mysql

COPY . /opt/source-code

ENTRYPOINT FLASK_APP=/opt/source-code/app.py flask run
```
## Docker Compose
- Let's assume all applications and containers are already built and loaded to the Docker repository
- We'll start with the data layer first: `docker run -d --name=redis redis`
- Next, we deploy the PostgreSQL database by running: `docker run -d --name=db postgres`
- Then, we'll deploy a frontend app for a voting interface by running an instance of a voting app image: `docker run -d --name=vote -p 5000:80 voting-app`
- Next, we'll deploy a "results" web application that shows the results to the user, which publishes port 80 on the container to port 5001 on the host running the Docker engine: `docker run -d --name=result -p 5001:80 result-app`
- Finally, we deploy the worker by running an instance of the worker image: `docker run -d --name=worker worker`
- When we try to access this from the browser, it shows an Internal Server Error. The reason is because we've built the applications, but we've not networked them together.
- This is where we use LINKS. 
### Links
- `--link` is a command line option which can be used to link two containers together.
- Using the voting app web service as an example, it is dependent on the Redis service.
- When the webserver starts as you can see in this piece of code on the webserver it looks for a Redis server running on the host.
```
def get_redis():
  if not hasattr(g, 'redis'):
    g.redis = Redis(host="redis", db=0, socket_timeout=5)
  return g.redis
```
- But the voting app container cannot resolve a host by the name Redis to make the voting app aware of the Redis service. We'll add a link option while running the voting app container to link it to the Redis container: `docker run -d --name=vote -p 5000:80 --link redis:redis voting-app`
- Similarly, the results application will be linked to the database:  `docker run -d --name=result -p 5001:80 db:db result-app`
- And lastly, the worker application needs to link up with redis and postgres: `docker run -d --name=worker --link db:db --link redis:redis worker`
- Note: Using links this way is deprecated and support may be removed in the future. A better way is to use Docker Compose:
```
redis:
  image: redis

db:
  image: postgres:9.4

vote:
  image: voting-app
  ports:
    - 5000:80
  links:
    - redis

result:
  image: result-app
  ports:
    - 5001:80
  links:
    - db

worker:
  image: worker
  links:
    - redis
    - db
```
- Now it's as simple as running a `docker-compose up` command.
- Let's add a network, outside of the default bridged network:
```
version: 2
services:
  
  redis:
    image: redis
    networks:
      - backend
  
  db:
    image: postgres:9.4
    networks:
      - backend
      
  vote:
    image: voting-app
    networks:
      - front-end
      - back-end
  
  result:
    image: result
    networks:
      - front-end
      - back-end
      
  networks:
    front-end:
    back-end:
```

## Docker Engine
- `docker -H=remote-docker-engine:2375`
- `docker -H=10.123.2.1:2375 run nginx`

## Docker Volumes/Storage
- `docker run -v data_volume:/var/lib/mysql mysql`
- `docker run -v data_volume2:/var/lib/mysql mysql`
- `docker run -v /data/mysql:/var/lib/mysql mysql`
- `docker volume create data_volume`
- `docker run \ --mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql`
- Run a mysql container again, but this time map a volume to the container so that the data stored by the container is stored at /opt/data on the host.
- Use the same name : mysql-db and same password: db_pass123 as before. Mysql stores data at /var/lib/mysql inside the container.
- `docker run -v /opt/data:/var/lib/mysql -d --name mysql-db -e MYSQL_ROOT_PASSWORD=db_pass123 mysql`
- Disaster strikes.. again! And the database crashed again. But this time we have the data stored at `/opt/data` directory. Re-deploy a new mysql instance using the same options as before.
- Just run the same command as before. Here it is for your convenience: `docker run -v /opt/data:/var/lib/mysql -d --name mysql-db -e MYSQL_ROOT_PASSWORD=db_pass123 mysql`

## Docker Networking
- When you install Docker it creates three networks automatically: `bridge`, `none`, and `host`.
- `bridge` is the default network.
- Bridge network: `docker run ubuntu`
- None: `docker run Ubuntu --network=none`
- Host: `docker run Ubuntu -- network=host`
- You can also make user-defined networks: 
```
docker network create \
  --driver bridge \
  --subnet 182.18.0.0/16 \
  --gateway= 182.18.0.1
  custom-isolated-network
```
### Inspect the Network
- `docker inspect blissful_hopper`
- `docker network ls`
- `docker network inspect bridge`
