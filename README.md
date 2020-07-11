# Setting up my Swarm/Traefik server

Instructions for Ubuntu 20.2.
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

Store the hostname, private key (id_rsa) and public key (id_rsa.pub) in Github secrets (or through environment variables in docker-compose file).

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

Install hashed password, user and password in Github Secrets (or through environment variables in docker-compose file).


## Notes regarding Docker registry

```
htpasswd -Bbn myuser mypassword
```

Basic authentication is handled by Traefic, not by the registry, so there is not need to configure anything here.
Just store the hostname, username and hashed password in Github secrets (or through environment variables in docker-compose file).

**WARNING**: Hostname has to include port number!


## Set Keycloak admin user and password

Just put in Github secrets:

```
KEYCLOAK_ADMIN_USERNAME
KEYCLOAK_ADMIN_PASSWORD
```

# Configure simple-mail-forwarder

To enable a mail forwarder, define the following variable in Github Secrets (or through environment variables in docker-compose file):

```bash
SMF_CONFIG=sam@barebaric.com:knipknap@gmail.com;sam@spiff.xyz:knipknap@gmail.com
```

## Bring Traefik online

Start deploy on Github.
Or, cpy the docker-compose.yml from this repository root, and install using "docker deploy" (after replacing included $ variables).
Done.


## Test the registry user

```
curl https://myuser:mypassword@$REGISTRY_DOMAIN:5000/v2/_catalog
```

## Other stuff

```
apt-get update
apt-get upgrade -y
journalctl --vacuum-size=100M
```
