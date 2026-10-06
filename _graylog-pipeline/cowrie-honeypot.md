---
layout: project
title: "Deploying a Cowrie Honeypot"
description: >
  Stand up an SSH honeypot on Debian 12 that logs login attempts. A good launching point for threat detection.
image: /assets/img/cowrie/cowrie-header.png
date: 2026-10-01
caption: An SSH honeypot that captures real brute-force attempts.
links:
  - title: Project overview
    url: /projects/graylog-pipeline/
redirect_from:
  - /2026-10-01-cowrie-install/
---

With standing up an SIEM tool, I started thinking about how to use it. 
{:.lead}

Obviously I would be ingesting logs, but the question is "what kind"? The purpose of standing up Graylog was to have a centralized source that I can use to demonstrate log ingestion and alerting, but I needed a source of those alerts.

I also wanted to learn more about platform detection and have a kind of "test dummy" that I could toy around with. This lead me to discovering Cowrie.

Cowrie is a premade honeypot that logs login attempts as well as commands executed within its environment. This is exactly what I was looking for.

* toc
{:toc .large-only}

## Environment Overview

*   **Cowrie:** An SSH/Telnet honeypot running in a VM. It presents a fake shell on port 22 and logs every username and password tried, as well as commands issues within the cowrie environment.

## Part 1: Provision the Cowrie VM on Proxmox

Cowrie runs as a lightweight Python application, so the VM requirements are minimal. 

*  Download a Debian 12 ISO and deploy it how you see fit.
*  I would reccomend a VM with 1 CPU core, 2048 MB RAM, and a 10–15 GB disk.

**NOTE**: If your VM's storage supports snapshots, take one after Debian is installed so you can roll back easily.
I would also **highly** recommend that you do not put this on a public facing network or segment. This is a tool made to catch hackers and learn their techniques. If you're just getting into security, I wouldn't dangle bait right in front of their noses.
{:.message}

## Part 2: Install Cowrie

Current versions of Cowrie install as a Python package inside a virtual environment. 

Update the system and install the required dependencies:

~~~bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-venv python3-pip python3-dev libssl-dev libffi-dev build-essential
~~~

Create a dedicated non-root user. Cowrie must never run as root.

~~~bash
sudo adduser --disabled-password cowrie
sudo su - cowrie
~~~

As the `cowrie` user, clone the repository and set up the virtual environment:

~~~bash
git clone https://github.com/cowrie/cowrie
cd cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
(cowrie-env) $ python -m pip install --upgrade pip
(cowrie-env) $ python -m pip install -e '.[dev]'
~~~

Config templates ship under `src/cowrie/data/etc/`. You must copy them to the live `etc/` location:

~~~bash
cp src/cowrie/data/etc/cowrie.cfg.dist etc/cowrie.cfg
cp src/cowrie/data/etc/userdb.example etc/userdb.txt
~~~

Start the honeypot and test the local connection:

~~~bash
bin/cowrie start
~~~

## Part 3: Configure Cowrie

The installation by itself works perfectly fine, but lets put a little bit of realism into our deployment.

### Realistic Authentication
By default, Cowrie accepts almost any password, logging everything as `cowrie.login.success`. This breaks a brute-force demo since the detection keys on failed logins. Edit `etc/userdb.txt` so only one credential succeeds.

~~~text
root:x:!*
root:x:somepassword
~~~

The `!*` denies all passwords, while the second line explicitly allows one credential.

Cowrie also only listens to port 2222 by default. Lets forward port 22 to 2222 so we can simulate some logins.

~~~bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
~~~

Restart Cowrie to apply the changes:

~~~bash
source ~/cowrie/cowrie-env/bin/activate
cowrie restart
~~~

And that's it! Now, lets simulate some logins.

Opening a terminal on a local device that can reach it, lets try to ssh into the cowrie server.

~~~bash
ssh root@calypso.williejaynetwork.com
~~~

Lets give an invalid response and look at the logs.

~~~json
{
  "eventid": "cowrie.login.failed",
  "gl2_receive_timestamp": "2026-10-01 18:28:40.886",
  "gl2_remote_ip": "192.168.100.54",
  "session": "40eee19361d3",
  "gl2_remote_port": 46624,
  "source": "calypso",
  "gl2_source_input": "6abc61547f87c1d4de9348c8",
  "uuid": "660f42e8-bc41-11f1-90cc-bc24115201dc",
  "dst_ip": "192.168.100.54",
  "src_ip": "192.168.50.5",
  "gl2_processing_timestamp": "2026-10-01 18:28:40.887",
  "protocol": "ssh",
  "password": "123456",
  "gl2_source_node": "a620ebdd-8e94-42c4-97dd-93a753c0bab4",
  "gl2_processing_duration_ms": 1,
  "timestamp": "2026-10-01T18:28:40.887Z",
  "gl2_accounted_message_size": 337,
  "level": 1,
  "gl2_processing_error": "Replaced invalid timestamp value in message <ed135470-bdc5-11f1-a05b-bc2411a7ef2c> with current time - Value <2026-10-01T13:28:40.891761Z> caused exception: Invalid format: \"2026-10-01T13:28:40.891761Z\" is malformed at \"T13:28:40.891761Z\".",
  "streams": [
    "6abc5c1f7f87c1d4de932bc2"
  ],
  "gl2_message_id": "01M3WBKJSQ0000JC8XDYMZ7XTH",
  "message": "login attempt [root/123456] failed",
  "src_port": 49851,
  "dst_port": 2222,
  "sensor": "calypso",
  "_id": "ed135470-bdc5-11f1-a05b-bc2411a7ef2c",
  "time": 1790879320.8917608,
  "username": "root"
}
~~~

We can see from the logs that the user input the incorrect password and the attempt failed.

Now that we have Cowrie stood up, the next step would be to get it communicating with Graylog.
