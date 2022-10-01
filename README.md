# traefik_dockercompose
Simple Docker Compose File for Traefik v2


I assume you have Docker and Docker Compose installed.

For this compose file to work, you will need a data directory with a acme.json, the traefik.yml and the dynamic_conf.yml file.

```
mkdir -p /opt/containers/traefik/data
touch /opt/containers/traefik/data/acme.json
chmod 600 /opt/containers/traefik/data/acme.json
touch /opt/containers/traefik/data/traefik.yml
touch /opt/containers/traefik/data/dynamic_conf.yml

```
The acme.json needs the 600 Permisions.
All my Docker Containers are in the /opt/containers directory, feel free to change that.

Paste this into the traefik.yml file
```
api:
  dashboard: true

entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"
  mc:
    address: ":25565"



providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: "/dynamic_conf.yml"
    
```

Create a dynamic_conf.yml File in the data directory. And Paste this in it.
```
tls:
  options:
    default:
      minVersion: VersionTLS12

      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_AES_128_GCM_SHA256
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256

      curvePreferences:
        - CurveP521
        - CurveP384

      sniStrict: true


http:
  middlewares:
    secHeaders:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        frameDeny: true
        sslRedirect: true
        #HSTS Configuration
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"
```



Copy the docker-compose.yml and the dynamic.yml into your traefik directory.

Dynamic:
```
http:
  middlewares:
    secHeaders:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        frameDeny: true
        sslRedirect: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"


    redirect:
      redirectScheme:
        scheme: https
```
Docker-compose:
```
version: "3.3"
services:
    traefik:
        image: traefik:v2.4
        restart: unless-stopped
        container_name: traefik
        ports:
            - "80:80"
            - "443:443"


        command:
            ## API Settings ##
            - --api.insecure=true
            - --api.dashboard=true
            - --api.debug=true
            - --log.level=ERROR
            ##Provider Settings##
            - --providers.docker=true
            - --providers.docker.exposedbydefault=false
            - --providers.file.filename=/dynamic.yaml
            - --providers.docker.network=proxy
            ## Entrypoints ##
            - --entrypoints.http.address=:80
            - --entrypoints.https.address=:443
           # - --entrypoints.mc.address=:25565  #uncomment if you have a Minecraft Server behind traefik.

            ##Certificate settings##
            - --certificatesresolvers.mytlschallenge.acme.tlschallenge=true
            - --certificatesresolvers.mytlschallenge.acme.email=yourmail@domain.com  # add your mail adress
            - --certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json

        volumes:
            - ./letsencrypt:/letsencrypt
            - /var/run/docker.sock:/var/run/docker.sock
            - ./dynamic.yaml:/dynamic.yaml
        networks:
            - proxy

        labels:
            - "traefik.enable=true"
            - "traefik.http.routers.api.rule=Host(`traefik.yourdomain.com`)"   # add your domain for the traefik dashboard
            - "traefik.http.routers.traefik.entrypoints=http"
            - "traefik.http.routers.api.service=api@internal"


networks:
    proxy:
        external: true

```
DONT Forget to change the Domains and Email!!!

Create the Docker network "proxy" that we're using:
```
sudo docker network create proxy
```

Start the Container:
```
sudo docker-compose up -d
```
