# docker image: aprunit [Layer 2]

This layer 2 docker image is built upon [hwdm/apbase image](https://hub.docker.com/r/hwdm/apbase/) `latest` tag

## Description
This is the base image with runit process manager and go-dnsmasq dns resolver.

## Additions
Packages: `runit go-dnsmasq libressl ca-certificates bind-tools curl`

Files and Folders:
```
/:
    entrypoint

/etc/:
    runit
    services.d
    preinit.d
    postclr.d

/usr/bin/:
    rservice

```

## Usage
Convention: Applications dirname should start from 10, like 10-apache, 11-nginx etc.
Example: nginx
Add Service Enabling Script: /etc/preinit.d/10-nginxon
Add Daemon Management Scripts: /etc/services.d/nginx/{run, finish}

## Tags

* `latest` tracks the newest tag from [upstream](https://hub.docker.com/r/hwdm/apbase/)
* `212r3` indicates the glibc version in the image, i.e, `212r3=runit 2.1.2-r3`

# License
[Apache 2.0](https://www.tldrlegal.com/l/apache2)
