## Flarum Guide

_Almost to say that flarum docker mode installation is very simple, although you occurred some tiny error, please check with the docker log output, you will find the root cause._

### Prepare

- Git
- Nginx
- Docker
- Docker Compose

```bash
# Currently I used Oracle Free Cloud Instance to run this application.
# VM detail
uname -a 
Linux 3.10.0-1062.4.3.el7.x86_64 #1 SMP Wed Nov 13 23:58:53 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

# memory usage
free -h
              total        used        free      shared  buff/cache   available
Mem:           987M        571M        154M         80M        261M        316M
Swap:          8.0G        119M        7.9G

# cpu usage
top -c
top - 07:32:03 up 1 day,  3:55,  1 user,  load average: 0.19, 0.23, 0.23
Tasks: 146 total,   1 running, 145 sleeping,   0 stopped,   0 zombie
%Cpu0  :  2.2 us,  1.3 sy,  0.0 ni, 83.1 id,  0.0 wa,  0.0 hi,  0.0 si, 13.4 st
%Cpu1  :  1.6 us,  1.9 sy,  0.0 ni, 84.3 id,  0.0 wa,  0.0 hi,  0.0 si, 12.2 st
KiB Mem :  1011456 total,   149516 free,   591088 used,   270852 buff/cache
KiB Swap:  8388604 total,  8266456 free,   122148 used.   317808 avail Mem

# docker stats
CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
7bdaa29f9c16        mariadb             2.67%               91.18MiB / 987.8MiB   9.23%               1.89MB / 5.9MB      83.6MB / 364MB      31
9eaf55c64347        flarum              4.82%               11.57MiB / 987.8MiB   1.17%               6.24MB / 3.89MB     32.6MB / 999kB      
```

### Installation

- Pull docker image

```bash
# Current use version 0.1.0-beta.10-stable
# Pull from hub.docker.com :
docker pull mondedie/docker-flarum:0.1.0-beta.10-stable

# or build it manually :
docker build -t mondedie/docker-flarum https://github.com/mondediefr/flarum.git#master
```

- Edit docker-compose.yml

```yml
version: "3"

services:
  flarum:
    image: mondedie/docker-flarum:0.1.0-beta.10-stable
    container_name: flarum
    env_file:
      - /home/opc/apps/flarum/.flarum.env
    ports:
      - "127.0.0.1:8888:8888"
    volumes:
      - ./flarum/assets:/flarum/app/public/assets
      - ./flarum/extensions:/flarum/app/extensions
      - ./nginx/conf.d:/etc/nginx/conf.d
    depends_on:
      - mariadb

  mariadb:
    image: mariadb:10.4
    container_name: mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=xxxxx
      - MYSQL_DATABASE=flarum
      - MYSQL_USER=flarum
      - MYSQL_PASSWORD=xxxxx
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - ./mysql:/var/lib/mysql

```

- Edit .flarum.env config

> If you want to know all docker parameters, please check with this link: https://github.com/mondediefr/docker-flarum#environment-variables

```bash
# vim .flarum.env
DEBUG=true
FORUM_URL=https://bbs.homelss.group
# Database configuration
DB_HOST=mariadb
DB_NAME=flarum
DB_USER=flarum
DB_PASS=xxxx
DB_PREF=flarum_
DB_PORT=3306
# User admin flarum (environment variable for first installation)
# /!\ admin password must contain at least 8 characters /!\
#FLARUM_ADMIN_USER=xxxx
#FLARUM_ADMIN_PASS=xxxx
#FLARUM_ADMIN_MAIL=xxxx@gmail.com
#FLARUM_TITLE=Test flarum

```
- Run it

```bash
docker-compose up -d
```

- Nginx configuration

```bash
# almost same with mastodon nginx config :)
# sudo vim /etc/nginx/conf.d/flarum.conf
map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}

server {
  listen 80;
  listen [::]:80;
  server_name bbs.homeless.group;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { root /usr/share/nginx/html; allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/bbs.homeless.group/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/bbs.homeless.group/privkey.pem;

  client_max_body_size 50m;

  gzip on;
  gzip_disable "msie6";
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_buffers 16 8k;
  gzip_http_version 1.1;
  gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

  add_header Referrer-Policy "no-referrer";
  add_header Strict-Transport-Security "max-age=31536000";

  location / {
    try_files $uri @proxy;
  }

  location @proxy {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Proxy "";
    proxy_pass_header Server;

    proxy_pass http://127.0.0.1:8888;
    proxy_buffering off;
    proxy_redirect off;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;

    tcp_nodelay on;
  }

  error_page 500 501 502 503 504 /500.html;
}

```

### References

[docker-flarum](https://github.com/mondediefr/docker-flarum)