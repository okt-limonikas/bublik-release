---
title: Network-Restricted Deployment
---

This guide walks you through deploying Bublik in an network-restricted environment using two machines:

- **Preparation Machine**: Has internet access, used for downloading and preparing components
- **Deploy Machine**: Air-gapped Ubuntu 24.04 LTS system where Bublik will be deployed

Both machines should be **Ubuntu 24.04 LTS**
We support `arm64` and `amd64`
We assume `amd64` in below instructions

---

## Preparation Machine

### Install Dependencies

```bash
sudo apt-get update && sudo apt-get install jq git curl

# A restart may or may not be required for the command to be recognized depending on your system.
sudo snap install --classic go

# Add GOBIN to path so task works
echo 'export PATH="$HOME/go/bin:$PATH"' >> ~/.bashrc

# Install task from source
go install github.com/go-task/task/v3/cmd/task@latest

# Install Docker and Docker Compose
sudo curl -fsSL https://get.docker.com | sh

sudo usermod -aG docker $USER
# Apply group changes (allows running docker without sudo)
newgrp docker

source $HOME/.bashrc
```

**Verify installation:**

```bash
curl --version
jq --version
go version
task --version
git --version
docker --version
docker compose version
```

### 1.3 Prepare Initial Deploy

You will now create deploy package to transfer to deployment machine

```
$HOME/apps/bublik
├── deps
│   ├── docker-buildx-plugin_0.27.0-1~ubuntu.24.04~noble_amd64.deb
│   └── containerd.io_1.7.27-1_amd64.deb
├── bin/
│   └── task                   # Task executable
├── config/
│   └── .env                   # Environment file
├── versions/
│   ├── bublik-2.1.0/          # Version-specific repository
│   ├── bublik-1.8.0/          # Version-specific repository
│   └── ...
└── current -> versions/2.1.0/ # Symlink to active version
```

```bash
mkdir -p $HOME/apps/bublik
mkdir -p $HOME/apps/bublik/deps
mkdir -p $HOME/apps/bublik/bin
mkdir -p $HOME/apps/bublik/images
mkdir -p $HOME/apps/bublik/versions
mkdir -p $HOME/apps/bublik/config

# Transfer task
cp $(which task) $HOME/apps/bublik/bin

# Skip if deployment machine have jq already installed
cp $(which jq) $HOME/apps/bublik/bin
# Skip if deployment machine have curl already installed
cp $(which curl) $HOME/apps/bublik/bin
# Skip if deployment machine has curl already installed
cp $(which git) $HOME/apps/bublik/bin

# Download Docker packages
cd $HOME/apps/bublik/deps && apt download docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin && cd -
```

#### Setup Sources

```bash
# Version is with TAG e.g v2.1.0
# Example: `git clone --branch v2.1.0 --recurse-submodules https://github.com/ts-factory/bublik-docker.git $HOME/apps/bublik/versions/bublik-2.1.0`
git clone --branch <version> --recurse-submodules https://github.com/ts-factory/bublik-docker.git $HOME/apps/bublik/versions/bublik-<version>

# Setup link to current deploy repo
# Example: `ln -s $HOME/apps/bublik/versions/bublik-2.1.0 $HOME/apps/bublik/current`
ln -s $HOME/apps/bublik/versions/bublik-<version> $HOME/apps/bublik/current

# Setup .env file
cp $HOME/apps/bublik/current/.env.example $HOME/apps/bublik/config/.env
ln -s $HOME/apps/bublik/config/.env $HOME/apps/bublik/current/.env
# Setup IDs (needed for correct permissions)
echo "HOST_UID=$(id -u)" >> $HOME/apps/bublik/current/.env
echo "HOST_GID=$(id -g)" >> $HOME/apps/bublik/current/.env

# Check Symlinks
ls -lha $HOME/apps/bublik | grep "current"
ls -lha $HOME/apps/bublik/current/.env | grep ".env"

# Set correct version so it pulls correct image versions
# Example: `sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=2.1.0/' $HOME/apps/bublik/current/.env`
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<version_without_v>/' $HOME/apps/bublik/current/.env
# Verify correct image tag is set
cat $HOME/apps/bublik/current/.env | grep "IMAGE_TAG"

cd $HOME/apps/bublik/current && task pull

# Check that images pulled
docker image ls

# Save images
docker save -o $HOME/apps/bublik/images/bublik-<version>.tar \
  ghcr.io/ts-factory/bublik-nginx:<version> \
  ghcr.io/ts-factory/bublik-log-server:<version> \
  ghcr.io/ts-factory/bublik-runner:<version> \
  redis:latest \
  postgres:latest \
  rabbitmq:3-management
```

```bash
# Create Archive
tar czf $HOME/bublik-deploy.tar.gz -C $HOME apps

# Transfer everything to deploy machine
scp $HOME/bublik-deploy.tar.gz <deploy_machine>:~
```

---

## Deploy Machine

```bash
tar xzf $HOME/bublik-deploy.tar.gz

echo 'export PATH="$HOME/apps/bublik/bin:$PATH"' >> ~/.bashrc

rm bublik-deploy.tar.gz

source ~/.bashrc
```

### Install docker

```bash
sudo dpkg -i $HOME/apps/bublik/deps/*.deb
sudo usermod -aG docker $USER
newgrp docker
```

**Verify installation:**

```bash
curl --version
jq --version
task --version
git --version
docker --version
docker compose version
```

```bash
# Load images
# Example: `docker load -i $HOME/apps/bublik/images/bublik-2.1.0.tar`
docker load -i $HOME/apps/bublik/images/bublik-<version>.tar

# Verify correct images loaded
docker image ls

# Go to currently linked repo version
cd $HOME/apps/bublik/current

# Adjust UID and GUID in case it's different from prep machine in case they different
sed -i "s/^HOST_UID=.*/HOST_UID=$(id -u)/" .env
sed -i "s/^HOST_GID=.*/HOST_GID=$(id -g)/" .env
# Verify correct image tag is set
cat $HOME/apps/bublik/current/.env | grep "IMAGE_TAG"

# To setup and check
task setup

# To start application
task up
```

## 3: Update

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
```

**On target machine:**

```bash
```

### 3.3 Configure Environment

1. Edit the `.env` file on the target machine
2. Follow the configuration guide: [Bublik Configuration Documentation](https://ts-factory.github.io/bublik-release/docker/setup#4-configure-environment)

:::warning
You only need **step 4** from instructions if you are migration to docker from bare deploy
:::

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
- Default ports are defined in your `$HOME/apps/bublik/config/.env` file

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
