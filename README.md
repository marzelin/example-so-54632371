# Workaround

Change the `nginx-net` name to be alphabetically earlier that `common-net`, i.e. `anginx-net`. It seems that when publishing ports from container connected to more than one network, docker chooses the network interface whose network's name is alphabetically first.

# Overview
We've got 2 services:
1. `nginx-service` - server that serves html on port 80. This is `nginx:alpine` with one line modified in `/etc/nginx/conf.d/default.conf`: 

   - `   listen       nginx-net_nginx:80;`

1. `tester-service` (unmodified alpine image)

And 2 networks:
1.  `common-net` shared by both services
1. `nginx-net` that is exclusive to `nginx-service`

Port `80` on `nginx-net` is published on host's port `8080`

```
services:
  nginx-service:
    ports:
      - 127.0.0.1:8080:80
    networks:
      nginx-net:
        aliases:
          - nginx-net_nginx
      common-net:
        aliases:
          - common-net_nginx
  tester-service:
    networks:
      - common-net

networks:
  nginx-net:
  common-net:
```

# Requirements
Make `nginx-service` published ports (80) unaccessible to the second network (`common-net`) but it should still work on host

# How to run:
go to project's dir and type:
`docker-compose up -d`

# Test case:
access shell in `tester-service`:

`docker-compose exec tester-service ash`

1. you should be able to communicate with `nginx-service`:

   - `ping -c 2 nginx-service` should work

2. you should not be able to access nginx server on port 80:

   - `wget -O nginx-service` should return access denied

3. you should be able to access the server on host:

   - when on host running `wget -O localhost:8080` should return nginx default site

# The problem

Currently 3. condition isn't met. It looks like docker forwards ports only from `common-net` network interface while I have nginx listening on `nginx-net` interface (so that the webserver can't be accessible to `tester-service`). 

I'd like to be able to tell docker that is should listen to the ip aliased as `nginx-net_nginx` not `common-net_nginx`.

# helpful commands:

`nginx -s reload` - reloads the server to reflect changes in the server configuration (doesn't work for every type of change unfortunatelly so use with caution)

`netstat -tln` shows on what ports a host/container is listening

`ip addr show` or `ifconfig` - shows network interfaces
