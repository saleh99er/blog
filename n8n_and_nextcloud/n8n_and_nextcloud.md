# Hosting n8n and nextcloud
**Written by Saleh Hassen on April 21, 2024**

## Intro
As frugally as possible I would like to host two services:
- N8n, a workflow automation platform (similar to Zapier) that allows me to integrate various services and APIs together with little to no code. The community edition allows for self-hosting.
- Nextcloud, an open source content collaboration platform including calendar, tasks, file management, photos and more.

I'll walk through how I hosted these services using Caddy and Docker compose.

## Cloud Provider

I wanted a cloud provider that provided at least 2 GiB of RAM and at least 50 GB of disk space. Both services could perform well with these specs, without a lot of computational demand. I would also prefer hosting the server near me within the United States.

After comparing cloud providers, I found the Hetzner cloud CPX21 instance was the best exchange of monthly cost for computing resources. The CPX21 includes 3 vCPUs, 4 GB of RAM, 80 GB of SSD disk space, 20 TB of traffic, all for $7.57 per month with an additional $0.64 per month for an IPv4 address. Costing a total of $8.21 per month. Hetzner also was able to provide cloud hosting near me.

In comparison with AWS, a t4g.small instance (2 vCPUs 2 GiB RAM), would cost me $8.25 per month for 50 GB of Elastic Block Storage and $12.49 per month for 24/7 runtime. With a savings plan I could reduce the amortized cost in exchange for long term commitment but this would at least cost me $7.44 per month. So worst case $20.74 per month and best case $15.69 per month.

I also considered Digitalocean. For 50 GB of disk space and 2 GB of RAM I would pay $12 per month. 

## Create my server instance

I created an account on Hetzner cloud and created a CPX21 instance within the United States. I chose a Debian 12 Operating system.

## Domain Name

I acquired a domain name from namecheap.com and pointed the new domain's domain servers to Hetzner domain name servers. 

Then on Hetzner's DNS console I assigned A and AAAA records to my new cloud instance.

## General linux server setup

I connected with my server with SSH and largely followed [this video tutorial intended for nextcloud AIO setup](https://youtu.be/Nh2-LjIymmQ?si=HuGcFBoOAuuiAV-v&t=288) to set up some best security practices.

### Create a limited sudo user & update
Create a user with limited sudo access with commands `adduser <username>` and `usermod -aG sudo <username>`. Store your password in a safe place. Login with the new user. We can test the new user's access and perform package updates by request the list of packages and performing the available upgrades with ``sudo apt update && sudo apt upgrade``.

### Adjust system date & time

I found the timezone was not in sync with my local time using `timedatectl`, to update this use `sudo timedatectl set-timezone <your timezone>` from an option in `timedatectl list-timezones` . Reboot afterwards, `sudo reboot`

### Disable root SSH login

Then I disabled root SSH login. Please make sure your other user is setup before this step. If you want some redundancy in case you're unable to sign in to your current user in the future, setup another limited sudo user with different credentials. Revise the sshd config with `sudo nano /etc/ssh/sshd_config` , you’ll want to have only one line present in the file referencing PermitRootLogin and for it to have the line `PermitRootLogin no` . Then restart the service, `sudo systemctl restart sshd`. 

### Firewall setup

Now we’ll setup a firewall and open the ports needed for nextcloud.  

```bash
sudo apt install ufw
sudo ufw enable
sudo ufw allow 80/tcp # HTTP
sudo ufw allow 443/tcp # HTTPS
sudo ufw allow 8080/tcp # Nextcloud AIO Interface
sudo ufw allow 8443/tcp # SSL
sudo ufw allow 3478 # Nextcloud talk
sudo ufw allow ssh # to maintain access to the server through SSH
```

### Install Docker, Git, and psmisc

Next we'll install Docker with `curl -fsSL get.docker.com | sudo sh`. So we can call upon docker without sudo priveleges, we'll create a docker group and add user to permissions with `sudo groupadd docker` and `sudo usermod -aG docker $USER`.

We'll also install Git to clone the needed repositories for N8n and Nextcloud. `sudo apt install git`.

Lastly, in case we need fuser for freeing ports I also installed `sudo apt install psmisc`. You can kill a process using a port with `fuser -k <port number>/<transport protocol>`. If the port is being used by a docker container you can instead kill the container with `docker container kill <container name>`.

## Nextcloud and n8n setup

From the home directory I created a new directory called server. I started with [the docker compose file](https://github.com/nextcloud/all-in-one/blob/main/compose.yaml) from the Nextcloud AIO repository as a baseline template for Nextcloud. I made some revisions to align the started instances toward my needs. 
```bash
cd ~
mkdir server
cd server
mkdir nextcloud/ncdata
mkdir n8n/data
```
- I commented out the mapping of port 80 and 8443 since I was running the mastercontainer behind a reverse proxy
- I uncommented the environment section's APACHE_PORT and APACHE_IP_BINDING to set the needed environment variable to run Nextcloud behind a reverse proxy
- I defined the NEXTCLOUD_DATADIR environment variable, specifying I would like to host directory for Nextcloud's datadir at ~/server/nextcloud/ncdata
- I added two services for Caddy and n8n
- I added a .env file specifying parameters for my n8n instance including credentials and data directory location.
- I also added a Caddyfile that will be read by the caddy service mapping web requests to the two subdomains of my choice to the appropriate docker container. 

My version of the files are located in [n8n_and_nextcloud](n8n_and_nextcloud) directory
- (.env)[.env]
- (docker-compose.yaml)[docker-compose.yaml]
- (Caddyfile)[Caddyfile]

You can use a CLI editor or an IDE with SSH support to create and add the above files.

Within youer server directory, start the services with `docker-compose up -d`.

Complete the setup for Nextcloud AIO on port 8443. Check that you can access both n8n and Nextcloud. I believe this current template is extensible, allowing you to host more services within the same server assuming the instance can meet the resource demands.

Once the setup is done we can see the running docker containers with `docker ps`

![Docker ps command output with the following containers: nextcloud/all-in-one, n8n, caddy, nextcloud/aio-apache, nextcloud/aio-notify, nextcloud/aio-nextcloud, nextcloud/aio-docker-socket-proxy, nextcloud/aio-imaginary, nextcloud/aio-redis, nextcloud/aio-postgresql, nextcloud/aio-talk, nextcloud/aio-collabora](running_containers_after_setup.png)

## Sources

https://docs.hetzner.com/dns-console/dns/general/authoritative-name-servers/

https://youtu.be/Nh2-LjIymmQ?si=HuGcFBoOAuuiAV-v&t=288

https://github.com/nextcloud/all-in-one

https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md

https://github.com/nextcloud/all-in-one/blob/main/compose.yaml

https://www.youtube.com/watch?t=847&v=E9wfEtGr_tc

## Next steps

I like using Nextcloud and n8n so far. As of right now I don't have a backup strategy for my data hosted on Hetzner cloud but will create some automatic backups before hosting any critical data. 

I had some trouble getting IPv6 support to work, I commented the network configuration for docker-compose.yaml for now, I may try again in the future.