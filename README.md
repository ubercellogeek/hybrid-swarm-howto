# Mixed OS Docker Swarm Setup How-To

This document walks through the basic steps to get a hybrid windows + linux docker swarm up and running.

## Host Preparation

For this demonstration, two virtual machines, or physical machines with the following operating systems must be set-up, and up to date. 

### Windows Instance
- Windows 2022 Server or Datacenter
- 8GB RAM
- 50GB Disk
- 4 Core CPU

### Linux Instance
- Ubuntu Server 22.04 (x86_64)
- 8GB RAM
- 30GB Disk
- 4 Core CPU

Each of these systems should have a statically assigned IP.

## Linux Setup

To install the latest version of docker, simply copy the `install_docker.sh` file onto the target system. Run the script as `root`. 

After the script finishes, verify by making sure docker is installed by issuing the following command: `docker info`

## Windows Setup

To install the latest version of docker, simply copy the `install_docker.ps1` file onto the target system. Run the script from an elevated powershell prompt as Administrator.

You may be asked to restart the system, if so, reboot.

Ensure that the latest version of docker has been installed by issuing the following command from an elevated Administrator powershell prompt: `docker info`

## Swarm Initialization

On the linux machine, we first have to create the Swarm. To do this, ensure that the host is connected to the network and that docker is up and running. 

> It is also recommended to customize the network address space that Swarm will use. Please choose a network range that DOES NOT collide with any network ranges on the host machine. 

Execute the following command as `root`:

`docker swarm init --default-addr-pool 10.112.0.0/12`

> You may recieve a message saying that there are multiple ip addresses on the host and that you need to specify one to listen on as the Swarm Manager. If so, please add the following additional argument `--advertise-addr YOUR_LINUX_HOST_IP_HERE`

You should receive a message displaying the token with which to use to join a worker to the node. Copy that command for the next step.

### Join windows node to the swarm as a worker

On the windows host, run the join worker command that as echoed out to the console from the previous Swarm initialization on the linux host.

Make sure that you run the command in an elevated powershell prompt.

Example:

`docker swarm join --token SWMTKN-1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx 192.168.1.100:2377`



## Swarm Configuration

On the linux host, create a new `overlay` networked called `proxy-public`. To do this, execute the following command:

`docker network create -d overlay proxy-public`


## Create a stack

On the linux host, create a simple nginx manager stack by executing the following:

`docker stack deploy -c nginx_proxy_manager.yml nginx-proxy-manager`

This will create an instance of nginx upon which you can forward all external web requests to any service within the swarm.

Log into the nginx UI manager by going to `http://IP_OF_YOUR_LINUX_NODE:81` The default username is `admin@example.com` and the password is `changeme`. 

Next, on the linux host, create a new stack which will deploy a simple IIS container on the windows node. 

`docker stack deploy -c test_windows_stack.yml iis-test`

It will take some time, but eventually the service should spin up a container on the windows host. To see the status of the service, issue the following command on the linux host:

`docker service ls`

## Testing it out

After the service shows 1/1 replicas up and running, its time to test the overlay network mesh routing. Within the nginx UI manager, create a new Proxy Host that points to the internal service. In this case, it would be `iis-test_iis-web`. Pick a domain name that is resolvable from your machine and is pointed at the linux host. Enter that in the "Domain Names" field. Within the Forward Hostname / IP field, set it to `iis-test_iis-web`. Make sure to specify port 80. Hit save.

You should now be able to visit the domain name that is pointed at the linux host over port 80 and see the IIS welcome page. Example: `http://my_custom_domain_name` (note again that `my_custom_domain_name` should resolve to the IP of the linux host).
