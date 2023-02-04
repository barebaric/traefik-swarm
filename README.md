# Setting up a Swarm/Traefik server on Ubuntu 20.04 LTS

I worked on optimizing these setup instructions for a long time.
I believe this is now a quick and simple set of instructions to get your swarm up and running, including management tools.

## Prerequisites

- A VM or server running **Ubuntu 20.04 LTS** that is reachable using SSH.
- At least ~30G worth of storage, though that won't get you far, i.e. you will have to prune your containers very frequently. I recommend at minimum 40G, and personally would go with at least 100G.

## What's in the box?

What you'll have when you are done:

| Default Endpoint | Product | Description
|----------|---------|-------------
| - | [Docker Swarm](https://docs.docker.com/engine/swarm/) | The core swarm
| - | [Docker GC](https://hub.docker.com/r/clockworksoul/docker-gc-cron) | Automatic container garbage collection (runs at midnight by default)
| https://&#8203;registry.yourdomain.com | [Docker Registry UI](https://github.com/Joxit/docker-registry-ui) | UI for your private docker registry.
| https://&#8203;registry.yourdomain.com:5000 | [Docker Registry](https://docs.docker.com/registry/) | Your own private docker registry. Needed to deploy your own containers to your swarm
| https://&#8203;traefik.yourdomain.com | [Traefik](https://github.com/containous/traefik/) | The automatic reverse proxy, including its admin UI. [Lets Encrypt](https://letsencrypt.org/) will automatically create SSL certificates for you
| https://&#8203;swarmpit.yourdomain.com | [Swarmpit](https://github.com/swarmpit/swarmpit) | Manage your swarm and monitor resources (using influxdb)
| https://&#8203;keycloak.yourdomain.com | [Keycloak](https://github.com/keycloak/keycloak) | SSO server
| https://&#8203;mail.yourdomain.com:25 | [Postfix](https://github.com/knipknap/docker-simple-mail-forwarder) | An SMTP server to forward emails to you
| https://&#8203;mail.yourdomain.com:487 | [Postfix](https://github.com/knipknap/docker-simple-mail-forwarder) (TLS) | An SMTPS server to forward emails to you.<br/>$\textcolor{red}{\text{This service doesn't work!}}$<br/>While I managed to make this accept TLS connections, it doesn't fully work; I suspect a problem related to TLS options such as renegotiation. If you figure this out, please let me know.
| https://&#8203;gateone.yourdomain.com | [GateOne](https://github.com/liftoff/GateOne) | An HTTPS based SSH client
| https://&#8203;droppy.yourdomain.com | [Droppy](https://github.com/silverwind/droppy) | File storage server with a web interface
| https://&#8203;web.yourdomain.com | [nginx](https://www.nginx.com/) | Webserver to serve public files uploaded using Droppy

If you don't want or need any of these services, just remove them from docker-compose.yml.
With all these services combined, I believe you are well set-up for deploying your app stack.

# Getting started

I actually recommend that you fork this repository, as it allows you to

- Manage all input parameters (environment variables mentioned below) in Github Secrets
- Use Github's Actions to deploy whenever you make any change. This repository includes an auto-deployment workflow, see [deploy.yml](.github/workflows/deploy.yml).

If you don't want to do that, you can also just download the docker-compose.yml file and deal with setting variables yourself.

## Partition your storage as follows

Partitions:

- At least 15G root partition
- At least 5G /var/log
- Rest /var/lib

## Install Docker

```bash
sudo -s
apt update
apt-get upgrade -y
apt install mailutils apt-transport-https ca-certificates curl software-properties-common apache2-utils

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt update
apt install docker-ce -y

# Optionally limit log size
journalctl --vacuum-size=100M    

systemctl start docker
```

And on the master node:

```bash
docker swarm init
```

## Join worker nodes (optional)

On the worker nodes, install Docker as above, but then execute the following commands once per worker.

On master:

```bash
docker swarm join-token worker
```

Copy the result to the worker node, for example:

```bash
docker swarm join --token SWMTKN-1-sdrgddrg0988sr9sdgrddafvsefsgsg098drgrag-wfdr098drgrd8g 172.173.174.175:2377
```


## (Optional, not needed if you have a good SSH connection): Make browser-based SSH client work temporarily, without SSL

```bash
docker run -t --name=gateone -p 8000:8000 dezota/gateone
```

## Install docker-compose

```bash
sudo -s
curl -L https://github.com/docker/compose/releases/download/1.26.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## Create a deployment user (only needed if you deploy from Github, not if you do it locally)

```bash
adduser -g docker github
su - github
ssh-keygen

mv .ssh/id_rsa.pub .ssh/authorized_keys
cat .ssh/id_rsa
rm .ssh/id_rsa
cat .ssh/authorized_keys

exit
```

Store the hostname, private key (id_rsa) and public key (id_rsa.pub) in Github secrets (or set through environment variables in docker-compose file).
Example:

```bash
DOCKER_SSH_HOST=mydomain.com
DOCKER_SSH_PRIVATE_KEY=`cat id_rsa`
DOCKER_SSH_PUBLIC_KEY=`cat id_rsa.pub`
```


## Install local-persist volume driver, and create some volumes

```bash
curl -fsSL https://raw.githubusercontent.com/MatchbookLab/local-persist/master/scripts/install.sh | sudo bash
docker volume create -d local-persist -o mountpoint=/var/lib/docker-registry --name=docker-local-persist
docker volume create -d local-persist -o mountpoint=/var/lib/droppy --name=droppy
docker volume create -d local-persist -o mountpoint=/var/lib/droppy/public/ --name=droppy-public
```

(see https://github.com/MatchbookLab/local-persist)


## Prepare to install Traefik

Install domain, username, hashed password, and email in Github Secrets (or set through environment variables in docker-compose file).
For example:

```bash
TRAEFIK_DASHBOARD_DOMAIN=mydomain.com
TRAEFIK_DASHBOARD_USERNAME=admin
TRAEFIK_DASHBOARD_HASHED_PASSWORD=`openssl passwd -apr1 "$PASSWORD"`
LETSENCRYPT_EMAIL=me@example.com
```


## Notes regarding Docker registry

Basic authentication is handled by Traefic, not by the registry, so there is not need to configure anything here.
Just store the hostname, username and hashed password in Github secrets (or set through environment variables in docker-compose file).

**WARNING**: Hostname has to include port number!

Example:

```bash
REGISTRY_DOMAIN=mydomain.com:5000
REGISTRY_USERNAME=myuser
REGISTRY_HASHED_PASSWORD=`htpasswd -Bbn myuser mypassword`
```

## Set Keycloak admin user and password

Just put in Github secrets (or set through environment variables in docker-compose file). Example:

```bash
KEYCLOAK_ADMIN_USERNAME=user
KEYCLOAK_ADMIN_PASSWORD=password
```

## Configure simple-mail-forwarder

To enable a mail forwarder, define the following variable in Github Secrets (or set through environment variables in docker-compose file):

```bash
SMF_CONFIG=user@mydomain.com:test@gmail.com;user@mydomain2.com:test@gmail.com:mypassword
```

> **Warning**
> Make sure to include a password after the last `:` in the SMF_CONFIG variable. Otherwise spammers will find and use your mail relay.
> This password will be used for authentication on your mail server before it relays anything to anyone.
> The way SMF works is that the local email addresses double as user names for authentication, and the password is shared between all
> these users.

Example to test:

```
$ telnet mail.example.com 25
EHLO mail.example.com
AUTH LOGIN
334 VXNlcm5hbWU6          # This is the server requesting the username
dXNlckBteWRvbWFpbi5jb20=  # This is the base64 encoded username, e.g. `echo -n 'user@mydomain.com' | base64`
334 UGFzc3dvcmQ6          # This is the server requesting the password
bXlwYXNzd29yZA==          # This is the base64 encoded password, e.g. `echo -n 'mypassword' | base64`
```

> **Note**
> If you want to use your mail server not just to forward mails to yourself, but as a general mail relay, you will have to **configure
> DKIM** (including setting the appropriate DNS records), and you will need a **dedicated IP address with a PTR record**. You cannot
> set this PTR record in your own DNS records and will need to ask your IP address provider to do that for you.
> It is almost always easier and safer to ask your hosting provider if they already provide an SMTP service for their customers.

## Bring the whole stack online

Start deploy on Github.
Or, copy the docker-compose.yml from this repository root, and install using "docker deploy" (after setting all the variables mentioned above).
Done.

## Test the registry user

```bash
curl https://myuser:mypassword@$REGISTRY_DOMAIN:5000/v2/_catalog
```
