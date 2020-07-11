# Setting up a Swarm/Traefik server on Ubuntu 20.2

I worked on optimizing these setup instructions for a long time.
I believe this is now a quick and simple set of instructions to get your swarm up and running, including management tools.

## Prerequisites

- A VM or server running **Ubuntu 20.2** that is reachable using SSH.
- At least ~30G worth of storage, though that won't get you far, i.e. you will have to prune your containers very frequently. I recommend at minimum 40G, and personally would go with at least 100G.

## What's in the box?

What you'll have when you are done:

| Default Endpoint | Product | Description
|----------|---------|-------------
| - | [Docker Swarm](https://docs.docker.com/engine/swarm/) | The core swarm
| registry.yourdomain.com | [Your own private docker registry](https://docs.docker.com/registry/) | Needed to deploy things to your swarm
| traefik.yourdomain.com | [Traefik](https://github.com/containous/traefik/) | The automatic reverse proxy, including its admin UI. [Lets Encrypt](https://letsencrypt.org/) will automatically create SSL certificates for you
| keycloak.yourdomain.com | [Keycloak](https://github.com/keycloak/keycloak) | SSO server
| mail.yourdomain.com | [Postfix](https://github.com/knipknap/docker-simple-mail-forwarder) | A SMTP server to forward emails to you
| gateone.yourdomain.com | [GateOne](https://github.com/liftoff/GateOne) | An HTTPS based SSH server
| swarmpit.yourdomain.com | [Swarmpit](https://github.com/swarmpit/swarmpit) | Manage your swarm and monitor resources (using influxdb)
| droppy.yourdomain.com | [Droppy](https://github.com/silverwind/droppy) | File storage server with a web interface
| yourdomain.com | [nginx](https://www.nginx.com/) | Webserver to serve public files uploaded using Droppy

If you don't want or need any of these services, just remove them from docker-compose.yml.
With all these services combined, I believe you are well set-up for deploying your app stack.

# Getting started

I actually recommend that you fork this repository, as it allows you to

- Manage all input parameters (environment variables mentioned below) in Github Secrets
- Use Github's Actions to deploy whenever you make any change. This repository includes an auto-deployment workflow, see [deploy.yml](.github/workflows/deploy.yml).

If you don't want to do that, you can also just download the docker-compose.yml file and deal with setting variables yourself.

## Partition your storage as follows

Partitions:

- At least 8G root partition
- At least 5G /var/log
- Rest /var/lib

## Install Docker

```
sudo -s
apt update
apt install mailutils apt-transport-https ca-certificates curl software-properties-common apache2-utils

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt update
apt install docker-ce -y
systemctl start docker

docker swarm init
```

## (Optional, not needed if you have a good SSH connection): Make browser-based SSH client work temporarily, without SSL

```
docker run -t --name=gateone -p 8000:8000 dezota/gateone
```

## Install docker-compose

```
sudo -s
curl -L https://github.com/docker/compose/releases/download/1.26.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### Create docker deployment user

```
adduser github
adduser github docker
su - github
ssh-keygen

mv .ssh/id_rsa.pub .ssh/authorized_keys
cat .ssh/id_rsa
rm .ssh/id_rsa
cat .ssh/authorized_keys

exit
```

Store the hostname, private key (id_rsa) and public key (id_rsa.pub) in Github secrets (or set through environment variables in docker-compose file).

```
DOCKER_SSH_HOST
DOCKER_SSH_PRIVATE_KEY
DOCKER_SSH_PUBLIC_KEY
```


## Install local-persist volume driver, and create some volumes

```
curl -fsSL https://raw.githubusercontent.com/MatchbookLab/local-persist/master/scripts/install.sh | sudo bash
docker volume create -d local-persist -o mountpoint=/var/lib/docker-registry --name=docker-local-persist
docker volume create -d local-persist -o mountpoint=/var/lib/droppy --name=droppy
docker volume create -d local-persist -o mountpoint=/var/lib/droppy/public/ --name=droppy-public
```

(see https://github.com/MatchbookLab/local-persist)


## Prepare to install Traefik

```
docker network create --driver=overlay traefik-public
export USER=admin
export PASSWORD=abcd
openssl passwd -apr1 "$PASSWORD"
```

Install domain, username, hashed password, and email in Github Secrets (or set through environment variables in docker-compose file).

```
TRAEFIK_DASHBOARD_DOMAIN
TRAEFIK_DASHBOARD_USERNAME
TRAEFIK_DASHBOARD_HASHED_PASSWORD
LETSENCRYPT_EMAIL
```


## Notes regarding Docker registry

Basic authentication is handled by Traefic, not by the registry, so there is not need to configure anything here.
Just hash the password:

```
htpasswd -Bbn myuser mypassword
```

and then store the hostname, username and hashed password in Github secrets (or set through environment variables in docker-compose file).

**WARNING**: Hostname has to include port number!

Example:

```
DOCKER_SSH_HOST=mydomain.com:5000
DOCKER_SSH_PRIVATE_KEY=`cat id_rsa`
DOCKER_SSH_PUBLIC_KEY=`cat id_rsa.pub`
```


## Set Keycloak admin user and password

Just put in Github secrets (or set through environment variables in docker-compose file). Example:

```
KEYCLOAK_ADMIN_USERNAME=user
KEYCLOAK_ADMIN_PASSWORD=password
```


# Configure simple-mail-forwarder

To enable a mail forwarder, define the following variable in Github Secrets (or set through environment variables in docker-compose file):

```bash
SMF_CONFIG=user@mydomain.com:test@gmail.com;user@mydomain2.com:test@gmail.com
```

## Bring the whole stack online

Start deploy on Github.
Or, copy the docker-compose.yml from this repository root, and install using "docker deploy" (after setting all the variables mentioned above).
Done.


## Test the registry user

```
curl https://myuser:mypassword@$REGISTRY_DOMAIN:5000/v2/_catalog
```

## Other stuff to do

```
apt-get update
apt-get upgrade -y
journalctl --vacuum-size=100M
```
