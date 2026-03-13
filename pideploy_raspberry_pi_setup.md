
# Raspberry Pi CI/CD Platform with PiDeploy

## Overview
Complete step-by-step guide to rebuild the Raspberry Pi CI/CD deployment platform.

Includes:
- Docker runtime
- Caddy reverse proxy
- Tailscale networking
- PiDeploy deployment controller
- GitHub Actions CI/CD pipeline

Follow the commands sequentially to recreate the system from scratch.

---

## High Level Architecture

```mermaid
flowchart TD
Developer[Developer Push Code] --> GitHubRepo[GitHub Repository]
GitHubRepo --> CI[GitHub Actions CI Pipeline]
CI --> BuildJar[Build Spring Boot JAR]
BuildJar --> BuildImage[Build Docker Image]
BuildImage --> PushImage[Push Image to GHCR]
PushImage --> DeployStep[Deployment Step]
DeployStep --> Tailscale[Join Tailscale Network]
Tailscale --> SSH[SSH into Raspberry Pi]
SSH --> PiDeploy[PiDeploy Deployment Controller]
PiDeploy --> DockerContainers[Docker Containers]
DockerContainers --> Caddy[Caddy Reverse Proxy]
Caddy --> Dev1[Environment dev1]
Caddy --> Dev2[Environment dev2]
```

---

## Directory Layout

```
/srv
 ├── infrastructure
 │    ├── caddy
 │    └── postgres
 │
 ├── environments
 │    ├── dev1
 │    └── dev2
 │
 └── pideploy
      ├── pideploy
      └── state.json
```

---

# Raspberry Pi Setup

## Update system

```bash
sudo apt update
sudo apt upgrade -y
```

## Install utilities

```bash
sudo apt install git curl jq -y
```

---

# Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
sudo reboot
```

Verify docker

```bash
docker version
```

---

# Install Docker Compose

```bash
sudo apt install docker-compose-plugin
```

---

# Create directories

```bash
sudo mkdir -p /srv/infrastructure
sudo mkdir -p /srv/environments
sudo mkdir -p /srv/pideploy
```

---

# Create docker network

```bash
docker network create platform
```

---

# Setup Caddy

Create folder

```bash
mkdir -p /srv/infrastructure/caddy
```

Create file

```
/srv/infrastructure/caddy/Caddyfile
```

Content

```
:80 {

    redir /dev1 /dev1/

    handle_path /dev1/* {
        reverse_proxy dev1-app:8080
    }

    redir /dev2 /dev2/

    handle_path /dev2/* {
        reverse_proxy dev2-app:8080
    }

}
```

Run Caddy

```bash
docker run -d --name caddy --restart always --network platform -p 80:80 -v /srv/infrastructure/caddy/Caddyfile:/etc/caddy/Caddyfile caddy
```

---

# Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status
```

Enable funnel

```bash
sudo tailscale funnel 80
```

---

# Enable SSH

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

# PiDeploy Script

File

```
/srv/pideploy/pideploy
```

```bash
#!/bin/bash

STATE_FILE="/srv/pideploy/state.json"
IMAGE="ghcr.io/kaustav1999paul/fithub-test"

ENV=$2
COMMIT=$3

if [ "$1" == "deploy" ]; then

echo "Deploying commit $COMMIT to $ENV..."

docker pull $IMAGE:$COMMIT

docker rm -f $ENV-app 2>/dev/null

docker run -d \
--name $ENV-app \
--network platform \
--restart always \
--label com.centurylinklabs.watchtower.enable=true \
$IMAGE:$COMMIT

jq ".\"$ENV\" = {\"commit\": \"$COMMIT\", \"auto\": false}" \
$STATE_FILE > tmp.json && mv tmp.json $STATE_FILE

echo "Deployment complete"

elif [ "$1" == "autodeploy" ]; then

COMMIT=$2

for ENV in $(jq -r 'keys[]' $STATE_FILE); do

AUTO=$(jq -r ".\"$ENV\".auto" $STATE_FILE)

if [ "$AUTO" == "true" ]; then

echo "Auto deploying $ENV"

docker pull $IMAGE:$COMMIT

docker rm -f $ENV-app 2>/dev/null

docker run -d \
--name $ENV-app \
--network platform \
--restart always \
$IMAGE:$COMMIT

jq ".\"$ENV\".commit=\"$COMMIT\"" \
$STATE_FILE > tmp.json && mv tmp.json $STATE_FILE

fi

done

elif [ "$1" == "get" ]; then

ENV=$2

jq ".\"$ENV\"" $STATE_FILE

elif [ "$2" == "autodeploy" ]; then

ENV=$1

jq ".\"$ENV\".auto=true" \
$STATE_FILE > tmp.json && mv tmp.json $STATE_FILE

echo "Latest commit will be deployed in $ENV"

else

echo "Usage:"
echo "pideploy deploy <env> <commit>"
echo "pideploy get <env>"
echo "pideploy <env> autodeploy"

fi
```

Make executable

```bash
chmod +x /srv/pideploy/pideploy
```

---

# Create global command

```bash
sudo ln -s /srv/pideploy/pideploy /usr/local/bin/pideploy
```

---

# Create state file

```bash
echo '{}' > /srv/pideploy/state.json
```

---

# GitHub Actions Workflow

`.github/workflows/deploy.yml`

```yaml
name: Build & Deploy Backend

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

jobs:

  build-and-deploy:
    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 25

      - name: Build JAR
        run: |
          chmod +x gradlew
          ./gradlew clean bootJar

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/kaustav1999paul/fithub-test:latest
            ghcr.io/kaustav1999paul/fithub-test:${{ github.sha }}
          platforms: linux/amd64,linux/arm64

      - name: Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: pi.tailXXXX.ts.net
          username: ${{ secrets.PI_USER }}
          key: ${{ secrets.PI_SSH_KEY }}
          script: |
            sleep 5
            pideploy autodeploy ${{ github.sha }}
```

---

# Deployment Commands

Manual deploy

```
pideploy deploy dev1 <commit-hash>
```

Enable auto deploy

```
pideploy dev1 autodeploy
```

Check deployment

```
pideploy get dev1
```

---

# Example URLs

```
https://pi.tailXXXX.ts.net/dev1/hello
https://pi.tailXXXX.ts.net/dev2/hello
```
