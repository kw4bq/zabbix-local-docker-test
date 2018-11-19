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
5. Let's create a user-defined bridged Docker network for our containers to use to communicate. Containers added to this network will automatically OPEN ALL ports to hosts INSIDE network. Any ports needing to be opened for hosts OUTSIDE the network will need to be exposed. We'll expose port 80 for the web interface in a later step so we can access it from our host machine. Now, let's create our bridged network, `docker network create zabbix-net`. (Alternately, we can specify the subnet and ip-range if needed, `docker network create --driver=bridge --subnet=172.30.0.0/16 --ip-range=172.30.0.0/24 --gateway=172.28.0.1 zabbix-net`.)
6. We can inspect this network to confirm the new network settings. `docker network inspect zabbix-net`. 
7. NOTE: Depending on which network is created, you may need to adjust the IP addressed in the Run section directly below.

## Run
Now we have the docker images on our machine, we can run the images with necessary configs and attach them to our birdged network.

1. Let's first start mysql. `docker run --name mysql-server-0 --network zabbix-net -e MYSQL_ROOT_PASSWORD="zabbix4mysql8" -d mysql:8` and we can check the logs `docker logs mysql-server-0`. Note: yes this is obviously insecure defining the root password on the cli.
2. Let's start the zabbix server. `docker run --name zserver-0 --network zabbix-net -e DB_SERVER_HOST="172.30.0.2" -e MYSQL_USER="root" -e MYSQL_PASSWORD="zabbix4mysql8" -d zabbix/zabbix-server-mysql:ubuntu-trunk` and let's check the logs again `docker logs zserver-0`.
3. Let's start the zabbix web interface. Note here, that we will expose port 80 so we can access the interface from the host machine. `docker run --name zweb-0 --network zabbix-net -p 80:80 -e ZBX_SERVER_HOST="172.30.0.3" -e DB_SERVER_HOST="172.30.0.2" -e MYSQL_USER="root" -e MYSQL_PASSWORD="zabbix4mysql8" -e PHP_TZ="America/New_York" -d zabbix/zabbix-web-nginx-mysql:ubuntu-trunk` and check the logs `docker logs zweb-0`.
4. Next let's verify our containers are attached to the network and confirm the ip addresses match the parameters above, `docker network inspect zabbix-net`.
5. The next step is to login to the zabbix web interface as setup a few parameters. visit `http://localhost:80` in your browser and you can sign in with the default credentials `Admin` and `zabbix`.
6. On the Zabbix Dashboard, let's confirm our server is running `Zabbix server is running	Yes	172.30.0.3:10051`.

## Auto Discovery
1. Let's setup our Discovery network and ensure new agents are added as Hosts. This will help us verify the docker network and zabbix server/agents are working as expected. Further customization may be desired.
2. In the web interface `Configuration > Discovery > Local network`. Let's set our network range as `172.30.0.0/24`, Check Interval `10s` and ensure the Check is `Zabbix agent` port `10050` and key `system.uname` should be fine. Additionally, we want the check uniqueness to be `IP address`. Now "Update".
3. Let's setup an Action to add the host, tag it, and give it a template. `Configuration > Actions > Create Action`. Note: you'll see an exisiting "Disabled" Auto discovery action, but we can ignore that one.
4. Name the new Action "Auto Discovery" then add a Condition that `Host IP equals 172.30.0.0/24`. 
5. Under `Operations` let's add the following Operations `Add Host`, `Add to host groups: Discovered hosts`, and `Link to templates: Template App Zabbix Agent`. This isn't obviosuly what we want exactly, but it will at least let us confirm our agents and docker network/containers are working as expected.
6. Be sure to click `Enabled` on the Actions tab, and click Update.
7. On the Dashboard, we might want to add a widget for `Discovery Status > Local Network`.

## Adding Zabbix Agents
1. Now we will add some sample agents, and ensure they are found by our server. These docker containers, by default, are run in `unpriviledged` mode, so if we want to give the agent containers access to the host system's resources, we can start run the agents with the `--priviledged` flag. However, I have not tested this specifically, yet.
2. Let's start an agent. `docker run --name zagent-0 --network zabbix-net -e ZBX_HOSTNAME="zagent-0" -e ZBX_SERVER_HOST="172.30.0.3" -d zabbix/zabbix-agent:ubuntu-trunk` and check the logs `docker logs zagent-0`.
3. We can check the network again to ensure the container was attached properly, `docker network inspect zabbix-net`.
4. Depending on the refresh rate of your Monitoring Dashboard, you will soon see under `Discovery Status > Local Network` a single agent `Up(1)`.
5. Next, we can add another agent, using the command above, but changing the `--name`, i.e. `docker run --name zagent-1 --network zabbix-net -e ZBX_HOSTNAME="zagent-4" -e ZBX_SERVER_HOST="172.30.0.3" -d zabbix/zabbix-agent:ubuntu-trunk`.
6. We'll want to verify it's been added to our docker network, and then ultimately added as a Zabbix Agent via our Auto Discovery rule.

## Futher work
1. We may need to change the `Interface` the agent for our `Zabbix server` to be the first agent we ran in step 2. We can find this setting under `Configuration > Hosts > Zabbix Server`. Just change the interface to the IP of one of the agents you started, i.e. `172.30.0.5:10050` and set it as the `default`. This will eliminate some Problems in the Dashboard.
2. As running agents on the local machine doesn't seem to be of much use we will want to consider deploying agents on other machines. Since the `bridged` network only applies to the local machine, we will want to consider Docker's `overlay` network. I haven't set up an overlay network, but there is documentation at https://docs.docker.com.
3. Setting up a `docker-compose.yaml` script to automate this deployment.



