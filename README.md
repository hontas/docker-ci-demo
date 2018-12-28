# docker-ci-demo

## Prerequisites:
You will nedd the following
* A Linux computer/virtual machine that has a public ip and ports 80 and 443 opened.
* Docker https://docs.docker.com/install/
* docker-compose https://docs.docker.com/compose/install/

### DNS
Do one of the following:
* Set up a dns with wildcard subdomains that point to the public ip
* **OR**
* Set up the following subdomains: (CNAME to root domain)
    * portainer.(domain)
    * traefik.(domain)
    * ptraefik.(domain)
    * drone.(domain)
    * docker.(domain)
    * gitlab.(domain)
    * demoapp.(domain)
    * presentation.(domain)
    * portal.(domain)

## Clone repo
* Clone this repo to your machine and go to the *docker-ci-demo* folder.

## If you want to run this on anything other than a registered domain (locally or docker-machine)
* - Use `dnsmasq` for DNS partial matches (wildcard), guide here: https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/
* - Setup self signed certificates (traefik wont be able to generate real ones). inspo: https://github.com/wekan/wekan/wiki/Traefik-and-self-signed-SSL-certs
```shell
# create certs
mkdir -p build/traefik/certs
cd build/traefik/certs
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout traefikKey.key -out traefikCert.crt
cd ../..
sudo chmod 755 traefik
sudo chmod 750 traefik/certs
chmod 644 traefik/certs/traefikCert.crt
chmod 600 traefik/certs/traefikKey.key
```
1. Comment the lines under [acme]
2. Then uncomment the following lines in traefik config file
```shell
[[entryPoints.https.tls.certificates]]
  certFile = "/certs/traefikCert.crt"
  keyFile = "/certs/traefikKey.key"
```
3. And finally uncomment this line in traefik Dockerfile
```shell
COPY certs /certs
```

## Network
Create the network that is used by all apps.
```
docker network create core
```

# .env file
Add your domain to the .env settings file:
```
echo SERVER_DOMAIN=fredriklowenhamn.com > .env
(replace fredriklowenhamn.com with your domain)
```

# Start containers
```
docker-compose up -d traefik presentation
docker-compose up -d
```

## Configure GitLab
Wait for gitlab to start. https://gitlab.fredriklowenhamn.com  

* Change password and login (as root)

* Create a group called *secure*. 
All members of the group *secure* will be able to access oauth2proxy

Test that docker registry works by logging in from host
```
docker login docker.fredriklowenhamn.com
```
If this fails, go to https://docker.fredriklowenhamn.com and try again afterwards. This forces traefik to get ssl certificate from letsencrypt.

### Add application
* Select trusted
* Select all scopes
* Add callbacks
* https://portal.fredriklowenhamn.com/oauth2/callback
* https://drone.fredriklowenhamn.com/authorize

Edit .env file  
```
echo COOKIE_SECRET=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | fold -w 32 | head -n 1)  >> .env  
echo CLIENT_ID=(Application Id) >> .env  
echo CLIENT_SECRET=(Application Secret) >> .env 
```

Refresh affected contatiners by running:
```
docker-compose up -d
```

# Demo app
* Go to gitlab and create a new project called *demoapp*  
* Add a readme to the project so that it can be cloned.  

## Enable repo in drone
* Under secrets add docker_username and docker_secret  

## Clone & add files
- Clone the repo you just created 
- Add the files from presentation/app in this repo.
- Edit *.drone.yml* and change *fredriklowenhamn.com* to your domain
- Push

Check the build in drone.

## Start demoapp
```
curl https://raw.githubusercontent.com/lowet84/docker-ci-demo/master/presentation/app/docker-compose.yml | DOMAIN=fredriklowenhamn.com REPO=root/demoapp docker-compose -f - up -d
```

# Watchtower
* Remeber to log in to docker!  
Watchtower watches for changes to the docker image on the registry and pulls and upgrades if a newer image exists.
Set the label: com.centurylinklabs.watchtower.enable=true on all containers that should be updated automatically.

# Continuous Delivery
* Edit and push to check that the continuous delivery is working.  

<br><br><br><br>
## Template for proxy
```
(service name)_proxy:
    container_name: (service name)_proxy
    image: lowet84/oauth2proxy
    labels:
      - traefik.enable=true
      - traefik.port=4180
      - traefik.frontend.rule=Host:(service name).${SERVER_DOMAIN}
    environment:
      - COOKIE_SECRET
      - SERVER_DOMAIN
      - CLIENT_ID
      - CLIENT_SECRET
      - SERVICE=(service name):(service port)
      - GROUP=(security group)
    restart: always
    networks:
      - core
```
