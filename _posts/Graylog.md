---
layout: post
title: "Building a Centralized Logging Pipeline with Graylog on Debian"
description: >
  A complete guide to installing a Graylog server natively on Debian using the apt package manager, secured behind an Nginx reverse proxy using Certbot and Cloudflare DNS.
---

Centralizing homelab logs transforms chaotic syslogs and web traffic access requests into a searchable, visual dashboard.
{:.lead}

While containerized deployments are popular, running Graylog directly on a Debian virtual machine ensures tight integration with system services and straightforward resource allocation. This guide covers provisioning Graylog, MongoDB, and OpenSearch via Debian's package manager, followed by routing web interface traffic securely through an Nginx reverse proxy using an SSL certificate provisioned by Certbot and Cloudflare DNS.

* toc
{:toc .large-only}

## System Preparation and Prerequisites

Start with a fresh Debian installation. Graylog's backend heavily relies on memory for Elasticsearch/OpenSearch indexing, so allocate at least 4GB of RAM.

Install the base dependencies required for the installation process:

~~~bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install apt-transport-https openjdk-17-jre-headless uuid-runtime pwgen dirmngr gnupg wget -y
~~~

## 1. Install MongoDB

Graylog uses MongoDB to store configuration data and metadata. Add the MongoDB repository and install the service:

~~~bash
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb http://repo.mongodb.org/apt/debian bullseye/mongodb-org/6.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
~~~

Enable and start the MongoDB daemon:

~~~bash
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
~~~

## 2. Install OpenSearch

OpenSearch handles the heavy lifting of storing and indexing the actual log messages. 

Add the OpenSearch repository and install the package:

~~~bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo apt-key add -
echo "deb https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
sudo apt-get update
sudo apt-get install opensearch -y
~~~

Before starting OpenSearch, edit the configuration file at `/etc/opensearch/opensearch.yml` to set the cluster name and disable the security plugin (as this is a single-node homelab setup). Add the following lines:

~~~yaml
cluster.name: graylog
node.name: ${HOSTNAME}
path.data: /var/lib/opensearch
path.logs: /var/log/opensearch
discovery.type: single-node
network.host: 127.0.0.1
action.auto_create_index: false
plugins.security.disabled: true
~~~

Enable the service:

~~~bash
sudo systemctl daemon-reload
sudo systemctl enable opensearch.service
sudo systemctl start opensearch.service
~~~

## 3. Install and Configure Graylog

With the database components running, download the Graylog repository package and install the server:

~~~bash
wget https://packages.graylog2.org/repo/packages/graylog-5.2-repository_latest.deb
sudo dpkg -i graylog-5.2-repository_latest.deb
sudo apt-get update
sudo apt-get install graylog-server -y
~~~

Next, configure the primary settings in `/etc/graylog/server/server.conf`. You must generate a `password_secret` (at least 64 characters) and a `root_password_sha2` (the hash of your admin password).

Generate the secret:
~~~bash
pwgen -N 1 -s 96
~~~

Generate the SHA-256 hash (replace `yourpassword`):
~~~bash
echo -n "yourpassword" | sha256sum
~~~

Paste both values into `/etc/graylog/server/server.conf`. You also need to bind the HTTP interface so Nginx can reach it:

~~~conf
password_secret = <your-pwgen-string>
root_password_sha2 = <your-sha256-hash>
http_bind_address = 127.0.0.1:9000
http_external_uri = https://logs.yourdomain.com/
~~~

Start Graylog:

~~~bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl start graylog-server.service
~~~

## 4. Securing Nginx with Certbot and Cloudflare DNS

To secure the connection to the web interface, we will use Nginx to handle SSL termination and proxy the traffic to port 9000. By using the Cloudflare DNS plugin for Certbot, we can obtain a Let's Encrypt certificate by automatically deploying a DNS TXT record, rather than exposing port 80.

Install Nginx, Certbot, and the Cloudflare plugin:

~~~bash
sudo apt-get install nginx certbot python3-certbot-dns-cloudflare -y
~~~

Next, generate an API token in your Cloudflare dashboard. For security, create a custom token with strictly **Zone: DNS: Edit** permissions rather than using your Global API Key. 

Save this token securely on your Debian server:

~~~bash
sudo mkdir -p /etc/letsencrypt
sudo nano /etc/letsencrypt/cloudflare.ini
~~~

Add your token to the file:

~~~ini
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
~~~

Secure the file permissions so only the root user can read it:

~~~bash
sudo chmod 600 /etc/letsencrypt/cloudflare.ini
~~~

Now, request the SSL certificate:

~~~bash
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini -d logs.yourdomain.com
~~~

Once the certificate generation is successful, create a new Nginx server block at `/etc/nginx/sites-available/graylog`:

~~~nginx
server {
    listen 80;
    server_name logs.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name logs.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/logs.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/logs.yourdomain.com/privkey.pem;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Graylog-Server-URL https://$server_name/;
        proxy_pass http://127.0.0.1:9000;
    }
}
~~~

Enable the site and restart Nginx to apply the configuration:

~~~bash
sudo ln -s /etc/nginx/sites-available/graylog /etc/nginx/sites-enabled/
sudo systemctl restart nginx
~~~

You can now securely log in to your dashboard at `https://logs.yourdomain.com`.
