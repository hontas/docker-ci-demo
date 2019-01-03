# docker-ci-demo using docker-machine
> For registered domains - see [master branch](https://github.com/hontas/docker-ci-demo/blob/master/README.md)

## Prerequisites:
You will need the following
* Docker https://docs.docker.com/install/
* docker-compose https://docs.docker.com/compose/install/

### Things referenced throughout the README
*domain*: your own made up domain name  
*vmName*: your created docker-machine vm name

### Clone repo
* Clone this repo and cd into the *docker-ci-demo* folder.

### Create a docker-machine
* Making sure it has enough memory to run GitLab
* [Check version of latest stable boot2docker](https://github.com/boot2docker/boot2docker/releases)
* Replace `vmName` below with desired name

```shell
# If Boot2Docker @ ^18.09.1
docker-machine create vmName --driver virtualbox --virtualbox-memory "4096"
# else (due to bugs)
docker-machine create vmName --virtualbox-memory "4096" --virtualbox-boot2docker-url "https://github.com/boot2docker/boot2docker/releases/download/v18.06.1-ce/boot2docker.iso"

docker-machine env vmName
# run last line of output (to operate in vmName)
# eg: eval $(docker-machine env vmName)
```

### DNS
1. Make up a domain name, eg `my-domain.test`
```shell
echo SERVER_DOMAIN=domain > .env
```
2. Do one of the following:
  * Set up the following subdomains: (CNAME to root domain)
    * portainer.(domain)
    * traefik.(domain)
    * drone.(domain)
    * docker.(domain)
    * gitlab.(domain)
    * demoapp.(domain)
    * presentation.(domain)
    * portal.(domain)

  * Use `dnsmasq` for local DNS wildcard matching, [guide here:](https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/)
  * Edit /etc/hosts on vmName
```shell
docker-machine ssh vmName
sudo vi /etc/hosts
# add 'docker.(domain) gitlab.(domain)' to end of line starting with 127.0.0.1
# hint: press [i] to enter insert mode, [escape] to exit insert mode and :x[enter] to save
exit
```

### SSL
Setup self signed certificates (as traefik wont be able to generate real ones). [Inspo blog post:](https://github.com/wekan/wekan/wiki/Traefik-and-self-signed-SSL-certs)

```shell
# create certs
mkdir -p build/traefik/certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout build/traefik/certs/traefikKey.key -x509 -days 365 -out build/traefik/certs/traefikCert.crt
# Be sure to use the name myregistrydomain.com as a CN - to make docker registry work
sudo chmod 755 build/traefik
sudo chmod 750 build/traefik/certs
chmod 644 build/traefik/certs/traefikCert.crt
chmod 600 build/traefik/certs/traefikKey.key
# Instruct every Docker daemon to trust that certificate
docker-machine scp build/traefik/certs/traefikCert.crt vmName:/home/docker/ca.crt
docker-machine ssh vmName sudo mkdir -p /etc/docker/certs.d/myregistrydomain.com:9000
docker-machine ssh vmName sudo mv ca.crt /etc/docker/certs.d/myregistrydomain.com:9000
```

### Network
Create the network that is used by all apps.
```shell
docker network create core
```

### Start containers
```shell
docker-compose up -d traefik
docker-compose up -d
```

### Add DNS entry inside containers
Get vmName's ip address: `docker-machien ip vmName`  
1. Add wildcard route for *domain* to **traefik**
```shell
curl -X PUT -H 'Content-Type: application/json' \
  -d '{"domains": ["*.(domain)"]}' \
  http://(vmNameIP):5080/container/name/traefik
```
2. Get dns-container's ip and set it in .env'file ()needs to be done again if vm is restarted
```shell
echo DNS_IP=$(docker inspect -f '{{.NetworkSettings.Networks.core.IPAddress}}' dns) >> .env
```

## Configure GitLab
Wait for gitlab to start. https://gitlab.(domain)  

1. Change password and login (as root)
2. Create a group called *secure*.
   All members of the group *secure* will be able to access oauth2proxy
3. Allow requests to the local network from hooks https://gitlab.(domain)/admin/application_settings/network#js-outbound-settings [Read more on GitLab docs](https://docs.gitlab.com/ee/security/webhooks.html)  Otherwise hooks wont work with drone + DNS

Test that docker registry works by logging in from host
```shell
docker login docker.(domain)
```

### Add application
* Select trusted
* Select all scopes
* Add callbacks
* https://portal.(domain)/oauth2/callback
* https://drone.(domain)/login

Edit **.env** file  
```shell
echo CLIENT_ID=(Application Id) >> .env
echo CLIENT_SECRET=(Application Secret) >> .env
echo COOKIE_SECRET=$(LC_ALL=C tr -dc 'A-Za-z0-9' < /dev/urandom | fold -w 32 | head -n 1)  >> .env
```

Refresh affected contatiners
```shell
docker-compose up -d
```

# Demo app
* Go to gitlab and create a new project called *demoapp*  
* Add a readme to the project so that it can be cloned.  

## Enable repo in drone
* Under secrets add docker_username and docker_secret  

## Disable SSL verification for webhook in GitLab
Wont work because self-signed...
* https://gitlab.(domain)/root/demoapp/settings/integrations

## Clone & add files
- Clone the repo you just created 
- Add the files from presentation/app in this repo.
- Edit *.drone.yml* and change *domain* to your domain
- Push

Check the build in drone.

## Start demoapp
```
curl https://raw.githubusercontent.com/lowet84/docker-ci-demo/master/presentation/app/docker-compose.yml | DOMAIN=your-domain REPO=root/demoapp docker-compose -f - up -d
```

# Watchtower
* Remeber to log in to docker!  
Watchtower watches for changes to the docker image on the registry and pulls and upgrades if a newer image exists.
Set the label: com.centurylinklabs.watchtower.enable=true on all containers that should be updated automatically.

# Continuous Delivery
* Edit and push to check that the continuous delivery is working.  

# ToDo
- [ ] Make drone agent resolve gitlab.(domain) - build not working now

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