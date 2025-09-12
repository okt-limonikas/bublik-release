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

## First Deploy

### Preparation Machine

#### Install Dependencies

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

#### Prepare Initial Deploy

You will now create deploy package to transfer to deployment machine

#### Test

```bash
# 1. Setup directories
mkdir -p $HOME/apps/bublik
mkdir -p $HOME/apps/bublik/deps
mkdir -p $HOME/apps/bublik/bin
mkdir -p $HOME/apps/bublik/images
mkdir -p $HOME/apps/bublik/versions
mkdir -p $HOME/apps/bublik/config

# 2. Prepare required binaries
cp $(which task) $HOME/apps/bublik/bin
# Skip if deployment machine have jq already installed
cp $(which jq) $HOME/apps/bublik/bin
# Skip if deployment machine have curl already installed
cp $(which curl) $HOME/apps/bublik/bin
# Skip if deployment machine has git already installed
cp $(which git) $HOME/apps/bublik/bin

# 3. Download Docker packages
cd $HOME/apps/bublik/deps && apt download docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin && cd -
```

#### Setup Preparation Machine

```bash
# 1. Clone repository
# Example: `git clone --branch v2.1.0 --recurse-submodules https://github.com/ts-factory/bublik-docker.git $HOME/apps/bublik/versions/bublik-2.1.0`
git clone --branch <version> --recurse-submodules https://github.com/ts-factory/bublik-docker.git $HOME/apps/bublik/versions/bublik-<version_without_v>

# 2. Setup link to current deploy repo
# Example: `ln -s $HOME/apps/bublik/versions/bublik-2.1.0 $HOME/apps/bublik/current`
ln -s $HOME/apps/bublik/versions/bublik-<version_without_v> $HOME/apps/bublik/current
ls -lha $HOME/apps/bublik | grep "current"

# 3. Setup .env file
cp $HOME/apps/bublik/current/.env.example $HOME/apps/bublik/config/.env
ln -s $HOME/apps/bublik/config/.env $HOME/apps/bublik/current/.env
ls -lha $HOME/apps/bublik/current/.env | grep ".env"

# 4. Setup IDs (needed for correct permissions)
echo "HOST_UID=$(id -u)" >> $HOME/apps/bublik/config/.env
echo "HOST_GID=$(id -g)" >> $HOME/apps/bublik/config/.env

# 5. Set correct version so it pulls correct image versions
# Example: `sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=2.1.0/' $HOME/apps/bublik/config/.env`
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<version_without_v>/' $HOME/apps/bublik/config/.env
# Verify correct image tag is set
cat $HOME/apps/bublik/current/.env | grep "IMAGE_TAG"

# 6. Pull images
cd $HOME/apps/bublik/current && task pull

# 7. Check that images pulled
docker image ls

# 8. Save images
docker save -o $HOME/apps/bublik/images/bublik-<version_without_v>.tar \
  ghcr.io/ts-factory/bublik-nginx:<version_without_v> \
  ghcr.io/ts-factory/bublik-log-server:<version_without_v> \
  ghcr.io/ts-factory/bublik-runner:<version_without_v> \
  redis:latest \
  postgres:latest \
  rabbitmq:3-management
```

#### Configuration

1. Edit the `.env` file: `$HOME/apps/bublik/config/.env`
2. Follow the configuration guide: [Bublik Configuration Documentation](https://ts-factory.github.io/bublik-release/docker/setup#4-configure-environment)

#### What You Have Prepared

At this point, you have successfully created a complete deployment package in `$HOME/apps/bublik/` containing:

```
$HOME/apps/bublik/
├── deps/                      # Docker installation packages (.deb files)
├── bin/                       # Required executables (task, jq, curl, git)
├── images/                    # Docker images saved as .tar files
│   └── bublik-<version>.tar
├── config/
│   └── .env                   # Your configured environment file
├── versions/
│   └── bublik-<version>/      # Complete Bublik repository with all components
└── current -> versions/bublik-<version>/  # Symlink to active version
```

**Key components ready for transfer:**
- ✅ All Docker dependencies and installation packages
- ✅ Required executables (task, jq, curl, git)
- ✅ Pre-pulled Docker images (saved as .tar archive)
- ✅ Complete Bublik application code and configuration
- ✅ Configured environment file with your settings

#### Transfer To Deployment machine

Now we'll package everything and transfer it to your air-gapped deployment machine.

```bash
# 1. Create Archive
tar czf $HOME/bublik-deploy.tar.gz -C $HOME apps

# 2. Transfer everything to deploy machine
scp $HOME/bublik-deploy.tar.gz <deploy_machine>:~
```

---

### Deploy Machine

```bash
# 1. Extract initial deploy version
tar xzf $HOME/bublik-deploy.tar.gz

# 2. Add packaged binaries to PATH
echo 'export PATH="$HOME/apps/bublik/bin:$PATH"' >> ~/.bashrc

# 3. Remove archive
rm bublik-deploy.tar.gz

# 4. Apply change to bashrc
source ~/.bashrc
```

#### Install docker

:::danger
It's highly recommended to use [official way to install](https://docs.docker.com/engine/install/ubuntu/) <br />
You can skip this step if you follow official guide
:::

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

#### Configuration

1. Edit the `.env` file: `$HOME/apps/bublik/config/.env`
2. Follow the configuration guide: [Bublik Configuration Documentation](https://ts-factory.github.io/bublik-release/docker/setup#4-configure-environment)

#### Finish Initial Deploy

```bash
# 1. Load images
# Example: `docker load -i $HOME/apps/bublik/images/bublik-2.1.0.tar`
docker load -i $HOME/apps/bublik/images/bublik-<version_without_v>.tar

# 2. Verify correct images loaded
docker image ls

# 3. Go to currently linked repo version
cd $HOME/apps/bublik/current

# 4. Link configuration
ln -sfn $HOME/apps/bublik/config/.env $HOME/apps/bublik/current/.env
# Verify link
ls -lha $HOME/apps/bublik/current/.env | grep ".env"

# 5. Adjust UID and GUID in case it's different from prep machine in case they different
sed -i "s/^HOST_UID=.*/HOST_UID=$(id -u)/" $HOME/apps/bublik/config/.env
sed -i "s/^HOST_GID=.*/HOST_GID=$(id -g)/" $HOME/apps/bublik/config/.env

# 6. Verify correct image tag is set
cat $HOME/apps/bublik/current/.env | grep "IMAGE_TAG"

# 7. To start application
task up
```

---

## Update

:::warning
Sequential updates are mandatory <br />
**Never skip minor versions** during updates to prevent system instability and data corruption
:::

#### Pre-Update Checklist

Before beginning any update process:

- [ ] **Create a full database backup**
- [ ] **Test the update path on a preparation machine first**
- [ ] **Review release notes for ALL intermediate versions**

### Preparation Machine

Actions in this section should be done on preparation machine

#### Step 1: Prepare New Version

```bash
# 1. Clone new version repository
# Example: `git clone --branch v2.1.0 --recurse-submodules https://github.com/ts-factory/bublik-docker.git $HOME/apps/bublik/versions/bublik-2.1.0`
git clone --branch <version> --recurse-submodules https://github.com/ts-factory/bublik-docker.git $HOME/apps/bublik/versions/bublik-<version_without_v>

# 2. Update preparation machine to new version
ln -sfn $HOME/apps/bublik/versions/bublik-<version_without_v> $HOME/apps/bublik/current
ls -lha $HOME/apps/bublik | grep "current"

# 3. Re-link .env configuration
ln -sfn $HOME/apps/bublik/config/.env $HOME/apps/bublik/current/.env
ls -lha $HOME/apps/bublik/current/.env | grep ".env"

# 4. Set correct `IMAGE_TAG` to new version
sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<version_without_v>/' $HOME/apps/bublik/config/.env
cat $HOME/apps/bublik/current/.env | grep "IMAGE_TAG"

# 5. Pull new images
cd $HOME/apps/bublik/current && task pull

# 6. Check that images pulled
docker image ls

# 7. Save new version images
docker save -o $HOME/apps/bublik/images/bublik-<version_without_v>.tar \
  ghcr.io/ts-factory/bublik-nginx:<version_without_v> \
  ghcr.io/ts-factory/bublik-log-server:<version_without_v> \
  ghcr.io/ts-factory/bublik-runner:<version_without_v> \
  redis:latest \
  postgres:latest \
  rabbitmq:3-management
```

#### Step 2: Create Update Package

At this point, you have successfully created a complete update package in `$HOME` containing:

```
$HOME/apps/bublik/
├── images/                    # Docker images saved as .tar files
│   └── bublik-<version>.tar
└── versions/
    └── bublik-<version>/      # Complete Bublik repository with all components
```

```bash
# 1. Create update archive with new version repository and images
tar czf $HOME/bublik-update-<version_without_v>.tar.gz -C $HOME apps/bublik/versions/bublik-<version_without_v> apps/bublik/images/bublik-<version_without_v>.tar

# 2. Transfer to deploy machine
scp $HOME/bublik-update-<version_without_v>.tar.gz <deploy_machine>:~
```

### Deployment Machine

Actions in this section should be done on deployment machine

#### Step 3: Backup Current System (On Deployment Machine)

```bash
# 1. Go to current deploy
cd $HOME/apps/bublik/current

# 2. Create DB backup
task backup:create

# 3. Copy backup to safe location
cp backups/backup_*.sql $HOME/bublik-backup-$(date +%Y%m%d_%H%M%S).sql
```

#### Step 4: Apply Update (On Deployment Machine)

```bash
# 1. Extract archive with update
cd $HOME
tar xzf bublik-update-<version_without_v>.tar.gz

# 2. Load new images
docker load -i $HOME/apps/bublik/images/bublik-<version_without_v>.tar

# 3. Verify new images loaded
docker image ls

# 4. Re-link to new version
ln -sfn $HOME/apps/bublik/versions/bublik-<version_without_v> $HOME/apps/bublik/current
# Verify link
ls -lha $HOME/apps/bublik | grep "current"

# 5. Re-link old configuration
ln -sfn $HOME/apps/bublik/config/.env $HOME/apps/bublik/current/.env
# Verify link
ls -lha $HOME/apps/bublik/current/.env | grep ".env"

# 6. Update IMAGE_TAG in configuration
sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=<version_without_v>/" $HOME/apps/bublik/config/.env
# Verify IMAGE_TAG is correctly set
cat $HOME/apps/bublik/config/.env | grep "IMAGE_TAG"
```

#### Step 5: Start Updated Application (On Deployment Machine)

```bash
# 1. Navigate to new current version
cd $HOME/apps/bublik/current

# 2. Start updated application
task up

# 3. Monitor logs for successful startup
docker compose logs -f

# 4. Verify application is running correctly
docker compose ps
```

### Update Path Example

**Scenario**: Updating from version 1.7.0 → 1.10.0

**Required Update Sequence**:

```
1.7.0 → 1.8.0 → 1.9.0 → 1.10.0
```

**Execution Steps**:

1. **Update 1.7.0 → 1.8.0**
   - Review 1.8.0 release notes
   - Test functionality before proceeding

2. **Update 1.8.0 → 1.9.0**
   - Review 1.9.0 release notes
   - Test functionality before proceeding

3. **Update 1.9.0 → 1.10.0**
   - Review 1.10.0 release notes
   - Final testing and validation

---

## Post-Deployment

**Startup Information:**
- Initial startup may take up to 30 seconds
- Check logs with `docker compose logs -f` if needed
- By default Bublik **will be served on port 80**, you can adjust this by editing `BUBLIK_DOCKER_PROXY_PORT` in `$HOME/apps/bublik/config/.env`

### Access Your Application

- Navigate to `http://<target_machine_ip>` in your browser
- Default ports are defined in your `$HOME/apps/bublik/config/.env` file

### Database Migration (If Applicable)

If you're migrating to docker deployment from a bare metal installation:

```bash
# 1. Run these commands on your old database server
pg_dump -h <db_host> -U <username> -d bublik -f backup_$(date +%Y%m%d_%H%M%S).sql

# 2. Transfer backup to deploy machine machine
scp backup_*.sql <deploy_machine>:~/

# 3. On deploy machine - restore database
cd $HOME/apps/bublik/current
task backup:restore -- <path_to_backup.sql>

# 4. Restart application
task up
```
