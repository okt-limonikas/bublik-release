---
title: Network-Restricted Deployment
---

<!--toc:start-->

- [1: Prepare Testing Machine](#1-prepare-testing-machine)
  - [1.1 Install System Dependencies](#11-install-system-dependencies)
  - [1.2 Install Docker](#12-install-docker)
  - [1.3 Setup Bublik Repository](#13-setup-bublik-repository)
- [2: Prepare Target Machine Dependencies](#2-prepare-target-machine-dependencies)
  - [2.1 Transfer Task Binary](#21-transfer-task-binary)
  - [2.2 Install Docker Engine](#22-install-docker-engine)
  - [2.3 Docker Group Setup](#23-docker-group-setup)
  - [2.4 Verify Target Machine Setup](#24-verify-target-machine-setup)
- [3: Deploy](#3-deploy)
  - [3.1 Backup Existing Database (If Applicable)](#31-backup-existing-database-if-applicable)
  - [3.2 Prepare Application Version](#32-prepare-application-version)
  - [3.3 Configure Environment](#33-configure-environment)
  - [3.4 Transfer Docker Images](#34-transfer-docker-images)
  - [3.5 Start the Application](#35-start-the-application)
  - [3.6 Restore DB (If Applicable)](#36-restore-db-if-applicable)
- [Post-Deployment](#post-deployment)
  - [Access Your Application](#access-your-application)
- [Update](#update)
  - [Pre-Update Checklist](#pre-update-checklist)
  - [Update Process](#update-process)
  - [Update Path Example](#update-path-example)
  <!--toc:end-->

This guide walks you through deploying Bublik in an network-restricted environment using two machines:

- **Testing Machine**: Has internet access, used for downloading and preparing components
- **Target Machine**: Air-gapped Ubuntu 24.04 LTS system where Bublik will be deployed

Both machines should be Ubuntu 24.04 LTS
We support `arm64` and `amd64`
We assume `amd64` in below instructions

---

## 1: Prepare Testing Machine

### 1.1 Install System Dependencies

:::danger
Always review scripts before executing them from the internet
:::

```bash
# Install Task (task runner)
sudo sh -c "$(curl -L https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin
```

### 1.2 Install Docker

```bash
# Install Docker and Docker Compose
sudo curl -fsSL https://get.docker.com | sh

# Apply group changes (allows running docker without sudo)
newgrp docker
```

**Verify installation:**

```bash
docker --version
docker compose version
```

### 1.3 Setup Bublik Repository

```bash
git clone --recurse-submodules https://github.com/ts-factory/bublik-docker.git

cd bublik-docker

# Initialize the project
# Note: You may need to log out and back in after Docker installation
task setup
```

---

## 2: Prepare Target Machine Dependencies

All commands in this section are run from the **testing machine** to prepare the target machine.

### 2.1 Transfer Task Binary

```bash
curl -sL https://github.com/go-task/task/releases/latest/download/task_linux_amd64.tar.gz -o task.tar.gz

tar -xzf task.tar.gz task

scp task <target_machine>:/tmp/

ssh <target_machine> "sudo -S mv /tmp/task /usr/local/bin/ && sudo chmod +x /usr/local/bin/task"

ssh <target_machine> "task --version"

rm task task.tar.gz
```

### 2.2 Install Docker Engine

:::danger
It's recommend to follow [official instructions for installation](https://docs.docker.com/engine/install/ubuntu/) if you can
:::

```bash
mkdir -p docker-offline && cd docker-offline

apt download docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

scp *.deb <target_machine>:/tmp/

ssh -t <target_machine> "cd /tmp && sudo dpkg -i *.deb || sudo apt-get install -f -y"

cd .. && rm -rf docker-offline
```

### 2.3 Docker Group Setup

```bash
ssh <target_machine>

sudo usermod -aG docker $USER

newgrp docker
```

Log out and log back in so that your group membership is re-evaluated.

> If you're running Linux in a virtual machine, it may be necessary to restart the virtual machine for changes to take effect.

### 2.4 Verify Target Machine Setup

```bash
ssh <target_machine>

docker --version
docker compose version
```

---

## 3: Deploy

### 3.1 Backup Existing Database (If Applicable)

If you're migrating to docker deployment:

```bash
# Run these commands on your database server
pg_dump -h <db_host> -U <username> -d bublik -f backup_$(date +%Y%m%d_%H%M%S).sql

# Transfer backup to target machine
scp backup_*.sql <target_machine>:~/
```

### 3.2 Prepare Application Version

**On testing machine:**

```bash
# Navigate to repository
cd ~/bublik-docker

# Checkout desired version (e.g., v1.7.0)
git checkout <version_tag>

# Transfer repository to target machine (excluding git history and env files)
rsync -avz --progress --exclude '.git/' --exclude '.env' ~/bublik-docker/ <target_machine>:~/bublik-docker
```

**On target machine:**

```bash
# Navigate to transferred repository
cd ~/bublik-docker

# Run setup (you may need to log out and back in for Docker group changes)
# Required only if it's your first deploy
task setup
```

### 3.3 Configure Environment

1. Edit the `.env` file on the target machine
2. Follow the configuration guide: [Bublik Configuration Documentation](https://ts-factory.github.io/bublik-release/docker/setup#4-configure-environment)

:::warning
You only need **step 4** from instructions if you are migration to docker from bare deploy
:::

### 3.4 Transfer Docker Images

**On testing machine:**

```bash
# Set the correct image tag (remove 'v' prefix, e.g., '1.7.0' for tag 'v1.7.0')
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<tag_without_v>/' .env

# Verify the tag is correct
cat .env | grep IMAGE_TAG

# Pull all required images (if everythin correct no errors should occur)
task pull

# Verify images are downloaded
docker image ls

# Save and transfer images to target machine
# This may take several minutes depending on network speed
docker save \
  ghcr.io/ts-factory/bublik-nginx:<tag_without_v> \
  ghcr.io/ts-factory/bublik-log-server:<tag_without_v> \
  ghcr.io/ts-factory/bublik-runner:<tag_without_v> \
  redis:latest \
  postgres:latest \
  rabbitmq:3-management \
| gzip | ssh <target_machine> "gunzip | docker load"
```

**On target machine:**

```bash
# Verify images were transferred successfully
docker image ls

# Set the correct image tag
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<tag_without_v>/' .env

# Verify the tag is correct
cat .env | grep IMAGE_TAG
```

### 3.5 Start the Application

**On target machine:**

```bash
# Start all services
task up
```

**Startup Information:**

- Initial startup may take up to 30 seconds
- Check logs with `docker compose logs -f` if needed

### 3.6 Restore DB (If Applicable)

**On target machine:**

```bash
# Navigate to transferred repository
cd ~/bublik-docker

# Restore DB
task backup:restore -- <path_to_backup.sql>

# Restart
task up
```

---

## Post-Deployment

### Access Your Application

- Navigate to `http://<target_machine_ip>` in your browser
- Default ports are defined in your `.env` file

---

## Update

:::warning
Sequential updates are mandatory <br />
**Never skip minor versions** during updates to prevent system instability and data corruption
:::

### Pre-Update Checklist

Before beginning any update process:

- [ ] **Create a full database backup**
- [ ] **Test the update path on a testing machine first**
- [ ] **Review release notes for ALL intermediate versions**

### Update Process

- Repeat 3 with the new version tag

### Update Path Example

**Scenario**: Updating from version 1.7.0 → 1.10.0

**Required Update Sequence**:

```
1.7.0 → 1.8.0 → 1.9.0 → 1.10.0
```

**Execution Steps**:

1. **Update 1.7.0 → 1.8.0**

   - Review 1.8.0 release notes
   - Follow any migration instructions

2. **Update 1.8.0 → 1.9.0**

   - Review 1.9.0 release notes
   - Follow any migration instructions

3. **Update 1.9.0 → 1.10.0**
   - Review 1.10.0 release notes
   - Follow any migration instructions
