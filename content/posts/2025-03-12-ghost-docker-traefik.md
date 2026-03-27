---
title: "How I Self-Hosted a Ghost Blog Using Docker, Traefik & AWS"
date: 2025-03-12
draft: false
tags: ["ghost", "docker", "traefik", "aws", "self-hosting"]
description: "Because why not make your blog about blogging also be your DevOps playground? Ghost + Docker + Traefik + AWS for the full infrastructure learning experience."
hero_image: "/images/hero/ghost-forest.jpg"
hero_alt: "Docker containers"
---

## Why I Did This

Like many devs, I hit that moment where I wanted a personal blog — but I didn’t just want to spin up a platform. I wanted to **build the infrastructure** myself, to make it a fun project that was part learning, part feeding the enjoyment of building things. 

I chose [Ghost](https://ghost.org) for the blog engine (modern, fast, beautiful).  
I chose Docker for containerization, [Traefik](https://traefik.io/) for reverse proxy & TLS, and AWS for the hosting playground.  

Why? A mix of elements I was familiar with and are well supported, with some new services I was curious to understand more about. Creating that nice ratio of familiarity to growth that always feels good.

---

## Stack Overview

 Here’s the stack I built:

- 🐳 **Docker** to containerize everything
- ⚡ **Traefik v3** as a smart reverse proxy with automatic HTTPS
- 📰 **Ghost** as the blog platform
- ☁️ **AWS EC2** as the host machine
- 🔐 **Let's Encrypt** for free TLS
- ⚙️ **Shell scripts** for bootstrapping
- 🐙 **Git** for version control

---

## Architecture in a Nutshell


```plaintext
+--------------+       +-------------------+
|  Internet 🌍  | <---> |  Traefik (TLS + Routing)
+--------------+       +-------------------+
                                |
                                v
                   +----------------------+
                   |     Ghost Blog       |
                   +----------------------+
                                |
                                v
                        +---------------+
                        |     AWS EC2    |
                        +---------------+
```

•	Traefik handles HTTPS with Let’s Encrypt
•	Ghost runs on port 2368, Traefik routes traffic to it
•	The entire system is defined with `docker-compose.yml`

---

## The Core Files


**docker-compose.yml**
Defines services: traefik, ghost, and optionally a database. Here’s the gist:

```
services:
  traefik:
    image: traefik:v3.3.4
    restart: always
    container_name: traefik
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.lets-encrypt.acme.email=<your-email>+letsencrypt@gmail.com"
      - "--certificatesresolvers.lets-encrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.lets-encrypt.acme.tlschallenge=true"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./letsencrypt:/letsencrypt"
    networks:
      - web
      - internal

database:
    image: mariadb:11.5
    restart: always
    env_file:
      - .env
    networks:
      - internal
    hostname: db
    volumes:
      - ./log:/var/log/mysql
      - ./lib:/var/lib/mysql
    labels:
      - "traefik.enable=false"

volumes:
  ghost-content:
  mysql-data:
  mysql-log:

networks:
  web:
    external: true
  internal:
    external: false

```

**.env**
Used with MariaDB to store creds

```
MYSQL_ROOT_PASSWORD=<your-root-password-here>
MYSQL_DATABASE=<your-db-name-here>
MYSQL_USER=<your-db-username-here>
MYSQL_PASSWORD=<your-db-user-password-here>

```

**Traefik files**

Defines dynamic routing rules & cert resolver for Let’s Encrypt.

**traefik.toml** 

```
[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http.redirections.entryPoint]
      to = "websecure"
      scheme = "https"

  [entryPoints.websecure]
    address = ":443"

[api]
  dashboard = true

[certificatesResolvers.lets-encrypt.acme]
  email = "<your-email>+letsencrypt@gmail.com"
  storage = "acme.json"
  [certificatesResolvers.lets-encrypt.acme.tlsChallenge]

[providers.docker]
  watch = true
  network = "web"

[providers.file]
  filename = "traefik_dynamic.toml"

[log]
  filePath = "traefik.log"
  level = "ERROR"


```

**traefik_dynamic.toml**

```
[http.middlewares.simpleAuth.basicAuth]
  usersFile = "users.secret"
[http.routers.api]
  rule = "Host(`traefik.trenigma.dev`)"
  entrypoints = ["websecure"]
  middlewares = ["simpleAuth"]
  service = "api@internal"
  [http.routers.api.tls]
    certResolver = "lets-encrypt"

```

**setup.sh**
Quick-start script to spin everything up 🚀

```
#!/bin/bash

# Prompt the user for the domain
read -p "Enter the domain you will be using: " domain

# Prompt the user for the email
read -p "Enter the email for LetsEncrypt notifications: " email

# Replace "trenigma.dev" with the user-provided domain
sed -i '' "s/trenigma.dev/$domain/g" traefik.toml
sed -i '' "s/trenigma.dev/$domain/g" traefik_dynamic.toml
sed -i '' "s/trenigma.dev/$domain/g" README.md

# Replace "your+letsencrypt@email.com" with the user-provided email
sed -i '' "s/<your-email>+letsencrypt@gmail.com/$email/g" traefik.toml

# Copy acme.json.example to acme.json and set permissions
cp acme.json.example acme.json
chmod 0600 acme.json

# Create Docker network
echo "Creating docker network called web"
docker network create web

# Prompt the user for username and password
read -p "Enter the username for basic auth: " username
read -sp "Enter the password for basic auth: " password
echo

# Create a password hash using htpasswd
password_hash=$(echo -n "$password" | openssl passwd -apr1 -stdin)

# Save the username and password hash to users.secret
echo "$username:$password_hash" > users.secret

echo "Setup complete."

echo "You should be able to do a \"docker compose up\" and it'll just start working" 

```
---

## Lazydocker

Special shoutout to Lazydocker as a fantastic tool to make Docker on the box suck less.

**install_lazydocker.sh**

```
#!/bin/bash

# allow specifying different destination directory
DIR="${DIR:-"$HOME/.local/bin"}"

# map different architecture variations to the available binaries
ARCH=$(uname -m)
case $ARCH in
    i386|i686) ARCH=x86 ;;
    armv6*) ARCH=armv6 ;;
    armv7*) ARCH=armv7 ;;
    aarch64*) ARCH=arm64 ;;
esac

# prepare the download URL
GITHUB_LATEST_VERSION=$(curl -L -s -H 'Accept: application/json' https://github.com/jesseduffield/lazydocker/releases/latest | sed -e 's/.*"tag_name":"\([^"]*\)".*/\1/')
GITHUB_FILE="lazydocker_${GITHUB_LATEST_VERSION//v/}_$(uname -s)_${ARCH}.tar.gz"
GITHUB_URL="https://github.com/jesseduffield/lazydocker/releases/download/${GITHUB_LATEST_VERSION}/${GITHUB_FILE}"

# install/update the local binary
curl -L -o lazydocker.tar.gz $GITHUB_URL
tar xzvf lazydocker.tar.gz lazydocker
install -Dm 755 lazydocker -t "$DIR"
rm lazydocker lazydocker.tar.gz

```

---

## Challenges


• **Domain routing with Traefik**: figuring out router rules and how they map to Ghost.

• **Let’s Encrypt certificates**: initial cert failures were due to labeling issues in traefik configs.

• **Mounting config files**: Ghost didn’t love misnamed config.production.json paths — fixed it by using correct mount points in Docker.

• **Database struggles**: Ghost works better with MariaDB than mysql.

---

## What I Learned

• Traefik is **really powerful**, and its Docker provider makes things super dynamic.
	
• Ghost works beautifully in Docker once paths are configured right.

• Lazydocker is the shit for dealing with containers on the box. Please go look it up if you've never heard of it.
	
• You don’t need 15 AWS services — an EC2 box + Docker + smart proxy can go a long way.
	
• Don’t skip `.env` and config validation — saved me hours of debugging.

• Ghost works better with MariaDB than mysql. Also use a `.env` file for creds.

---

## What’s Next?

• Setup [Scaleway](https://www.scaleway.com/en/transactional-email-tem/) for transactional emails

• Hook up CI/CD to GitHub Actions

• Monitoring of some kind, maybe Grafana

• Solid backup strategy for the instance

---

## Reflections

Hosting my own Ghost blog wasn’t just about blogging — it was about owning the stack.
Now every post I write lives on infrastructure I understand, manage, and trust.

If you’re thinking about doing the same — **do it**. It’s an awesome way to level-up your DevOps skills 💪🏽 

---

📎 [Check out the GitHub Repo](https://github.com/treehousepnw/ghost-aws-project)

📬 Got questions or want help? Shoot me an [email](mailto:tree7@protonmail.com). Thanks for reading! ☺️🍻

