---
title: "Minio for local S3 storage using podman"
date: 2021-11-03T12:49:00-04:00
draft: false
authors:
- Jason Montleon
tags:
- minio
- s3
- podman
---

# Minio for local S3 storage using podman

## Introduction
When using Single Node OpenShift and other options for local clusters it may be helpful to also have a local s3 instance.

## The very basics
An instance of minio without HTTPS can be created very easily.

### Networking considerations
Because the default ports for HTTP and HTTPS are privileged you must make a decision how to run these commands.
- Run the `podman run` commands as root
- Create `/etc/sysctl.conf.d/99-unprivileged.conf` with the line `net.ipv4.ip_unprivileged_port_start=80` and run `sudo sysctl --system` 
- Modify the first half of the port redirects and proxy configs from `80` and `443` to ports higher than 1024 (for example 8080 and 8443)

You can open the ports via firewall-cmd if you are running firewalld.
```
sudo firewall-cmd --add-port 80/tcp
sudo firewall-cmd --add-port 9001/tcp
```

To make these permanent run them again:
```
sudo firewall-cmd --permanent --add-port 80/tcp
sudo firewall-cmd --permanent --add-port 9001/tcp
```

### Create a data directory
- `mkdir -p /srv/podman/minio/data`

### Launch Minio
```
podman run \
  --detach \
  --name=minio-server \
  -e MINIO_ROOT_USER=changeme \
  -e MINIO_ROOT_PASSWORD=changeme \
  -v /srv/podman/minio/data:/data:z \
  --network=private \
  --restart unless-stopped \
  -p 80:9000 \
  -p 9001:9001 \
  docker.io/minio/minio:latest gateway nas /data --console-address ":9001"
```

### Create a bucket 
- Visit the UI at `http://localhost` or the `http://$remote-hostname` and login with the credentials from your podman command.
- Click `Buckets` in the left menu
- Click `Create Bucket` in the top right
- Give the bucket a name like `velero`
- Click save

You are now free to configure the bucket in MTC. Most likely you'll need to use the hostname of your system to access it.

## More advanced configuration with SSL
If you'd like to add SSL you can run minio and an nginx proxy in a pod together to accomplish this.

### Configure an nginx reverse proxy
We need to create some more directories for nginx configs:
```
mkdir -p /srv/podman/proxy/etc/nginx/conf.d/
mkdir -p /srv/podman/proxy/etc/ssl/certs
```

Create /srv/podman/proxy/etc/nginx/nginx.conf
```
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

http {
  include       /etc/nginx/mime.types;

  map $status $loggable {
    ~^[23]  0;
    default 1;
  }

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

  access_log  /var/log/nginx/access.log main if=$loggable;

  sendfile        on;
  keepalive_timeout  65;
  autoindex on;
  ssl_certificate /etc/ssl/certs/server.crt;
  ssl_certificate_key /etc/ssl/certs/server.key;

  include /etc/nginx/conf.d/*.conf;
}

events {
  worker_connections  1024;
}
```
    
Next you'll need to create the reverse proxy configuration. You will need to update the three server_name lines with the hostname of your system. Wildcards and multiple names are valid if you need to provide them. See http://nginx.org/en/docs/http/server_names.html for more information.
  
/srv/podman/proxy/etc/nginx/conf.d/minio.conf:
```
server {
  listen 80;
  server_name hostname;
  
  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen      443 ssl;
  server_name hostname;
  ignore_invalid_headers off;
  client_max_body_size 0;
  client_body_buffer_size 10m; 
  proxy_buffering off;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_set_header Connection "";
    chunked_transfer_encoding off;
	proxy_pass http://localhost:9000;
  }
}

server {
  listen 9002;
  server_name hostname;

  location / {
    return 301 https://$host:9443$request_uri;
  }
}

server {
  listen      9443 ssl;
  server_name hostname;
  ignore_invalid_headers off;
  client_max_body_size 0;
  proxy_buffering off;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Host $http_host;
    proxy_set_header Connection "";
    chunked_transfer_encoding off;
    proxy_pass http://localhost:9001;
  }
}
```
  
Create or procure your SSL cert and place your `server.crt` and `server.key` in /srv/podman/proxy/etc/ssl/certs

This [Red Hat Enable Sysadmin](https://www.redhat.com/sysadmin/webserver-use-https) post contains a basic example of how to generate certs if you require one.

### Networking considerations

Use firewall-cmd to open the SSL ports if you are running firewalld
```
sudo firewall-cmd --add-port 443/tcp
sudo firewall-cmd --add-port 9443/tcp
```

To make these changes permanent also run:
```
sudo firewall-cmd --permanent --add-port 443/tcp
sudo firewall-cmd --permanent --add-port 9443/tcp
```

### Launch Minio
We once again run minio, but this time we do so by creating a pod in the process. The port redirects may look odd here. Ports need to be exposed when the pod is created regardless of which container they are for so we're exposing all of the ports required for nginx. This includes ports to redirect HTTP requests to HTTPS.

```
podman run \
  --detach \
  --pod=new:minio \
  --name=minio-server \
  -e MINIO_ROOT_USER=changeme \
  -e MINIO_ROOT_PASSWORD=changeme \
  -v /srv/podman/minio/data:/data:z \
  --network=private \
  --restart unless-stopped \
  -p 80:80 \
  -p 443:443 \
  -p 9001:9002 \
  -p 9443:9443 \
  docker.io/minio/minio:latest gateway nas /data --console-address ":9001"
```

Now we start nginx in the pod:
```
podman run \
  --detach \
  --name=proxy \
  --pod=minio \
  -v /srv/podman/proxy/etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro,z \
  -v /srv/podman/proxy/etc/nginx/conf.d:/etc/nginx/conf.d:ro,z \
  -v /srv/podman/proxy/etc/ssl/certs:/etc/ssl/certs:ro,z \
  --restart unless-stopped \
  docker.io/nginx:latest
```

You should now be able to connect to the Minio service and console using SSL.

When dealing with a containers in a pod you manage them with `podman pod` commands. For example, `podman pod rm -f minio`.
