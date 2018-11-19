# zabbix-local-docker-test

Setup a local Zabbix server, zabbix web interface, and zabbix agents using Docker

There isn't going to be much code in this repository as it's just instructions for setting up a working Zabbix server, agents, and web interface using Docker on your local machine. I may add a `docker-compose.yaml` file to simplify this setup.

Zabbix is an enterprise-grade, free, open-source monitoring tool. We are using Zabbix 4.2.0 LTS with `ubuntu-trunk` as well as `mysql:8`.

## Prereqs
Docker

## Installation
You'll need the following images from offical Docker hub repos. Tags can be changed as needed below.

1. First, let's get mysql, the default database Zabbix will use. `docker pull mysql:8`
2. Let's get the Zabbix server. `docker pull zabbix/zabbix-server-mysql:ubuntu-trunk`
3. Let's get the Zabbix web interface. `docker pull zabbix/zabbix-web-nginx-mysql:ubuntu-trunk`
4. Let's get the Zabbix agent. `docker pull zabbix/zabbix-agent:ubuntu-trunk`
5. Let's create a user-defined bridged Docker network for our containers to use to communicate. Containers added to this network will automatically open ALL ports to the network. Any ports needing to be opened for hosts outside the network will need to be exposed. We'll expose port 80 for the web interface in a later step so we can access it. `docker network create zabbix-net`. (Alternately, we can specify the subnet and ip-range if needed, `docker network create --driver=bridge --subnet=172.30.0.0/16 --ip-range=172.30.0.0/24 --gateway=172.28.0.1 zabbix-net`.)
6. We can inspect this network to confirm the new network settings. `docker network inspect zabbix-net`.

## Run
Now we have the docker images on our machine, we can run the images with necessary configs and attach them to our birdged network.

1. Let's first start mysql. `docker run --name mysql-server-0 --network zabbix-net -e MYSQL_ROOT_PASSWORD="zabbix4mysql8" -d mysql:8` and we can check the logs `docker logs mysql-server-0`. Note: yes this is obviously insecure defining the root password on the cli.
2. Let's start the zabbix server. `docker run --name zserver-0 --network zabbix-net -e DB_SERVER_HOST="172.18.0.2" -e MYSQL_USER="root" -e MYSQL_PASSWORD="zabbix4mysql8" -d zabbix/zabbix-server-mysql:ubuntu-trunk` and let's check the logs again `docker logs zserver-0`.
3. Let's start the zabbix web interface. Note here, that we will expose port 80 so we can access the interface from the host machine. `docker run --name zweb-0 --network zabbix-net -p 80:80 -e ZBX_SERVER_HOST="172.18.0.3" -e DB_SERVER_HOST="172.18.0.2" -e MYSQL_USER="root" -e MYSQL_PASSWORD="zabbix4mysql8" -e PHP_TZ="America/New_York" -d zabbix/zabbix-web-nginx-mysql:ubuntu-trunk` and check the logs `docker logs zweb-0`.
4. Next let's verify our containers and confirm the ip addresses match the parameters above, `docker network inspect zabbix-net`.
5. The next step is to login to the zabbix web interface as setup a few parameters. visit `http://localhost:80` in your browser and you can sign in with the default credentials `Admin` and `zabbix`.





