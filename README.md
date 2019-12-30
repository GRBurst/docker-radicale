# Radicale Docker Compose

Docker compose file for a radicale server instance. It is based on this [docker image](https://hub.docker.com/r/tomsquest/docker-radicale).

The compose file is ment to run behind a reverse proxy that is available through the `my-reverse-proxy` network.

Remeber to use *more secure* environment variables than the defaults provided here, in this case:

- DOCKER_STORAGE: Persistent path of your host for the docker image

Run it with:
```bash
docker-compose pull
docker-compose up -d --remove-orphans --build
```

Note: You have to provide a `users` file in the build folder with credentials for radicale. Radicale uses `basic auth` (htpasswd) with bcrypt.


## Example Setup

### nginx reverse proxy

In many cases, you run several docker images behind a reverse proxy. Here is how I use it:

Create a `docker network` to let services from different compose files communicate

```bash
docker network create my-reverse-proxy
build: build
```


Nginx `docker-compose` file:

```Dockerfile
version: '3.7'

services:
  nginx:
    container_name: nginx
    build: build
    restart: on-failure
    networks:
      - my-reverse-proxy
      - default
    ports:
      - 80:80
      - 443:443
    volumes:
      - ${TLS_CERTS_DIR:-./.test_certs}:/tls_certs/:ro
      - auth-nginx:/auth/

volumes:
  auth-nginx:

networks:
  my-reverse-proxy:
    name: my-reverse-proxy
```


... and the radicale `docker-compose` file:

```Dockerfile
version: '3.7'

services:
  radicale:
    build: ./build
    container_name: radicale
    networks:
      - my-reverse-proxy
    init: true
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - SETUID
      - SETGID
      - CHOWN
      - KILL
    healthcheck:
      test: curl -f http://127.0.0.1:5232 || exit 1
      interval: 30s
      retries: 3
    restart: unless-stopped
    volumes:
        - ${DOCKER_STORAGE}/radicale:/data
        - config-radicale:/config

volumes:
  config-radicale:

networks:
  my-reverse-proxy:
    external: true
```

In the nginx configuration, you need to set appropriate `proxy_pass` to your radicale instance.

Notes:
1. You can use the hostname `radicale` in the `proxy_pass` since they are sharing a network
2. In the example, `https_server.include` is a file providing my default https settings for nginx

It will look similar like the following:

```nginx
server {
    server_name radicale.example.com;
    include conf.d/common/https_server.include;
    client_max_body_size       10m;
    client_body_buffer_size    128k;

     location / {
         proxy_pass http://radicale:80/;
         include conf.d/common/proxy.include;
     }
}
```
