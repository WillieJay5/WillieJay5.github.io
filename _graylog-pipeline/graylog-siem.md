---
layout: project
image: /assets/img/graylog/graylog-logo.jfif
title: "Building a Centralized Logging Pipeline with Graylog on Debian"
description: >
  A complete guide to installing a Graylog server natively on Debian using the apt package manager, secured behind an Nginx reverse proxy using Certbot and Cloudflare DNS.
date: 2026-09-30
caption: Graylog SIEM on Debian, behind an Nginx reverse proxy.
links:
  - title: Project overview
    url: /projects/graylog-pipeline/
redirect_from:
  - /2026-09-30-graylog/
---

I sat down and asked myself: What is my homelab's first line of defense?
{:.lead}

In security engineering, there's a core axiom: you can't defend what you can't see. When building out a homelab or testing environments in a sandbox, setting up centralized logging is the bedrock of visibility. Whether you're auditing failed SSH logins, tracking anomalous internal traffic, or performing post-incident forensics, having a central log collector and SIEM platform is essential for meaningful defense-in-depth.

While containerized deployments are popular for quick service spin-ups, running Graylog directly on a dedicated Debian virtual machine gives you direct hypervisor resource guarantees, predictable systemd service control, and zero container-abstraction overhead for memory-heavy indexing. This guide covers provisioning Graylog, MongoDB (management plane), and OpenSearch (data plane) via Debian's package manager, followed by routing web interface traffic securely through an Nginx reverse proxy using an SSL certificate provisioned by Certbot and Cloudflare DNS—keeping your lab's WAN attack surface at zero by avoiding open inbound HTTP/HTTPS ports.

* toc
{:toc .large-only}

## System Preparation and Prerequisites

Start with a fresh, bare-metal or VM installation of Debian in your hypervisor. OpenSearch's indexing engine heavily relies on memory for JVM heap allocation, so allocate at least 4GB of RAM (8GB+ recommended if you plan to ingest heavy syslogs or firewall state tables).

Install the base dependencies required for the installation process:

~~~bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install apt-transport-https openjdk-17-jre-headless uuid-runtime pwgen dirmngr gnupg wget -y
~~~

## 1. Install MongoDB

Graylog uses MongoDB as its management metadata store—holding user credentials, stream configuration rules, dashboards, and alert definitions. Add the MongoDB repository and install the service:

~~~bash
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb http://repo.mongodb.org/apt/debian bullseye/mongodb-org/6.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
~~~

Enable and start the MongoDB daemon under systemd:

~~~bash
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
~~~

## 2. Install OpenSearch

OpenSearch handles the heavy lifting of storing, indexing raw log messages, and executing analytical search queries.

Add the OpenSearch repository and install the package:

~~~bash
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo apt-key add -
echo "deb https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
sudo apt-get update
sudo apt-get install opensearch -y
~~~

Before starting OpenSearch, edit the configuration file at /etc/opensearch/opensearch.yml to define cluster settings. To follow the principle of least exposure in a single-node sandbox, bind OpenSearch strictly to 127.0.0.1 and disable the security plugin to prevent service overhead conflicts with Graylog:

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

With the database components operating locally, download the Graylog repository package and install the server:

~~~bash
wget https://packages.graylog2.org/repo/packages/graylog-5.2-repository_latest.deb
sudo dpkg -i graylog-5.2-repository_latest.deb
sudo apt-get update
sudo apt-get install graylog-server -y
~~~

Next, configure the primary settings in /etc/graylog/server/server.conf. To enforce credential confidentiality, you must generate a password_secret (a high-entropy salt string of at least 64 characters used for session encryption) and a root_password_sha2 (the SHA-256 hash of your admin password).

Generate a secure secret using pwgen:
~~~bash
pwgen -N 1 -s 96
~~~

Generate the SHA-256 hash (replace yourpassword):
~~~bash
echo -n "yourpassword" | sha256sum
~~~

Paste both values into /etc/graylog/server/server.conf. Note that we bind http_bind_address strictly to 127.0.0.1:9000 so the API isn't directly exposed on your broader local network segment:

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

Exposing admin consoles directly over unencrypted HTTP is a vulnerability. We will use Nginx to handle TLS termination and proxy traffic to port 9000. By using the Cloudflare DNS plugin for Certbot, we can obtain a Let's Encrypt certificate via DNS-01 validation rather than opening public inbound ports (80/443) on your router, keeping your lab's WAN attack surface at zero.

Install Nginx, Certbot, and the Cloudflare plugin:

~~~bash
sudo apt-get install nginx certbot python3-certbot-dns-cloudflare -y
~~~

Now would be a good time to review how public DNS and search records work. I use Cloudflare, but you can use whatever you wish. 

Generate an API token in your Cloudflare dashboard. Following the principle of least privilege, create a custom token with strictly Zone: DNS: Edit permissions rather than using your Global API Key—limiting token permissions restricts the blast radius if your lab config were ever compromised.

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

Once the certificate generation is successful, create a new Nginx server block at /etc/nginx/sites-available/graylog:

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
![Graylog Dashboard](/assets/img/graylog/graylog-login.png)
