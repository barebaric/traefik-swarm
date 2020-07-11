# Old stuff

This document contains installation instructions that have been made obsolete by improving the
main docker-compose file. However, I want to keep them as a reference, as they are still correct
and functioning.

## Install Certbot and create wildcard certificate, to be used mostly for the registry

```
apt-get install software-properties-common python-software-properties
wget https://dl.eff.org/certbot-auto
mv certbot-auto /usr/local/bin/certbot-auto
chown root /usr/local/bin/certbot-auto
chmod 0755 /usr/local/bin/certbot-auto

export BASE_DOMAIN=barebaric.com
export EMAIL=knipknap@gmail.com
apt install virtualenv
vi /usr/local/bin/certbot-auto # python-virtualenv durch virtualenv ersetzen and delete flag --no-site-packages
certbot-auto certonly --manual --preferred-challenges=dns --email $EMAIL --server https://acme-v02.api.letsencrypt.org/directory --agree-tos -d *.$BASE_DOMAIN
```

Here create DNS record according to instructions.

```
# Verify certificates
certbot-auto certificates 

# Prepare combined certificates
cd /etc/letsencrypt/live/$BASE_DOMAIN && \
cp privkey.pem domain.key && \
cat cert.pem chain.pem > domain.crt && \
chmod 777 domain.*
ln -s privkey.pem /etc/letsencrypt/live/barebaric.com/key.pem
ln -s cert.pem /etc/letsencrypt/live/barebaric.com/certificate.pem

# Setup letsencrypt certificates renewing 
cat <<EOF > /etc/cron.d/letsencrypt
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
30 2 * * 1 root /usr/bin/certbot renew >> /var/log/letsencrypt-renew.log && cd /etc/letsencrypt/live/$BASE_DOMAIN && cp privkey.pem domain.key && cat cert.pem chain.pem > domain.crt && chmod 777 domain.*
EOF
```

## Set up docker registry

```
export EMAIL=knipknap@gmail.com
export REGISTRY_DOMAIN=registry.barebaric.com

# Create certificate. For some reason, I couldn't get it to work with a wildcard certificate.
certbot certonly --standalone --preferred-challenges http --non-interactive  --staple-ocsp --agree-tos -m $EMAIL -d $REGISTRY_DOMAIN
cd /etc/letsencrypt/live/$REGISTRY_DOMAIN && \
cp privkey.pem domain.key && \
cat cert.pem chain.pem > domain.crt && \
chmod 777 domain.*
ln -s privkey.pem /etc/letsencrypt/live/registry.barebaric.com/key.pem
ln -s cert.pem /etc/letsencrypt/live/registry.barebaric.com/certificate.pem
cat <<EOF >> /etc/cron.d/letsencrypt
0 2 * * 1 root /usr/bin/certbot renew >> /var/log/letsencrypt-renew.log && cd /etc/letsencrypt/live/$REGISTRY_DOMAIN && cp privkey.pem domain.key && cat cert.pem chain.pem > domain.crt && chmod 777 domain.*
EOF
```

### Create docker-compose file for docker registry

```
mkdir /var/lib/docker-registry
cd /var/lib/docker-registry
vim docker-compose.yml
```

```
registry:
  restart: always
  image: registry:latest
  ports:
    - 5000:5000
  environment:
    REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
    REGISTRY_HTTP_TLS_KEY: /certs/domain.key
    REGISTRY_AUTH: htpasswd
    REGISTRY_AUTH_HTPASSWD_PATH: /etc/docker/registry/passfile
    REGISTRY_AUTH_HTPASSWD_REALM: Registry Realm
  volumes:
    - /etc/letsencrypt/live/registry.barebaric.com:/certs
    - /var/lib/docker-registry/passfile:/etc/docker/registry/passfile
    - /data/docker-registry:/var/lib/registry
```

## Terminal with ssl (probably deprecated, can be done in main docker-compose.yml)

```
docker run -t --name=gateone -v /etc/letsencrypt/live/barebaric.com:/etc/gateone/ssl/:ro -p 1443:8000 dezota/gateone
```

## Create a tag that Traefik can use to constrain certificate storage to only one node.

Wird nur gebraucht, wenn Traefik nicht sowieso auf ein einzelnes Node constrained ist.

```
# Zertifikatzugriff konfigurieren. Hier wird lediglich das aktuelle node als das Zertifikats-Storage-Node getaggt
# Traefik benutzt die Kombination aus Email und bekannter Domain um von Letsencrypt weitere Certs dynamisch abzurufen
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
```
