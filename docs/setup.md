# Setup Guide

## Phase 1 — Infrastructure

### Step 1: Network Planning

Open VirtualBox and navigate to **Tools → Network → NAT Networks**. Create a new NAT network with the following settings:

- **Name:** soc-net
- **CIDR:** 10.0.5.0/24
- **DHCP:** Enabled

Once the network is created, add the following port forwarding rules so services running inside the VM are accessible from your host browser:

| Name    | Protocol | Host IP   | Host Port | Guest Port |
|---------|----------|-----------|-----------|------------|
| Cortex  | TCP      | 127.0.0.1 | 9001      | 9001       |
| Wazuh   | TCP      | 127.0.0.1 | 8443      | 443        |
| TheHive | TCP      | 127.0.0.1 | 9000      | 9000       |
| Shuffle | TCP      | 127.0.0.1 | 3001      | 3001       |
| MISP    | TCP      | 127.0.0.1 | 9443      | 9443       |

> Guest IPs are assigned by DHCP. Note the IP of the VM after it boots and update the Guest IP column accordingly.

---

### Step 2: Wazuh Manager Deployment

Create a new VM with the following specifications:

- **OS:** Ubuntu 22.04 LTS Server
- **RAM:** 11GB
- **Disk:** 100GB
- **vCPUs:** 6
- **Network:** soc-net

Boot the VM and run the following to update the system and install Wazuh using the official installation assistant:

```bash
sudo apt update && sudo apt upgrade -y

curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The installer will output admin credentials on completion - save these immediately as they are only shown once.

Once installed, verify all services are running:

```bash
sudo systemctl status wazuh-dashboard wazuh-indexer wazuh-manager
```

The Wazuh dashboard is accessible at `https://127.0.0.1:8443` from your host machine.

---

### Step 3: TheHive & Cortex Installation

TheHive and Cortex are deployed together using StrangeeBee's official Docker Compose repository.

First, install Docker on the VM:

```bash
sudo apt update && sudo apt upgrade -y

curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
```

Verify Docker Compose is available:

```bash
docker compose version
```

Clone the official repository and navigate to the testing profile, which includes Cortex:

```bash
git clone https://github.com/StrangeBeeCorp/docker.git
cd docker/testing
```

Run the initialisation script:

```bash
./scripts/init.sh
```

Before starting the services, open `docker-compose.yml` and change the nginx port to avoid a conflict with Wazuh on port 443:

```bash
nano docker-compose.yml
```

Find the nginx port mapping and change it from `443:443` to `4443:443`.

Start the services:

```bash
docker compose up -d
docker compose ps
```

TheHive is accessible at `http://127.0.0.1:9000/thehive`. Log in with the default credentials `admin@thehive.local / secret`.

---

### Step 4: Shuffle SOAR Installation

Clone the official Shuffle repository and navigate into it:

```bash
git clone https://github.com/Shuffle/shuffle
cd shuffle
```

Open `docker-compose.yml` and change the OpenSearch port mapping from `9200:9200` to `9201:9200` to avoid a port conflict:

```bash
nano docker-compose.yml
```

Start the services:

```bash
sudo docker compose up -d
docker compose ps
```

Shuffle is accessible at `http://127.0.0.1:3001`. Complete the initial setup by creating an admin account.

---

### Step 5: MISP Installation

Clone the official MISP Docker repository and navigate into it:

```bash
git clone https://github.com/MISP/misp-docker.git
cd misp-docker
```

Open `docker-compose.yml` and update the misp-core port to "${CORE_HTTPS_PORT}:443":

Copy the environment template and open it for editing:

```bash
cp template.env .env
nano .env
```

Configure the following values in `.env`:

```env
# NETWORK & PORTS
CORE_HTTPS_PORT=9443
BASE_URL=https://127.0.0.1:9443

# ADMIN ACCOUNT
ADMIN_EMAIL=root@soc.lab
ADMIN_PASSWORD=<your-password>

# DATABASE
MYSQL_ROOT_PASSWORD=<your-root-password>
MYSQL_PASSWORD=<your-password>
```

Start the services:

```bash
docker-compose up -d
docker compose ps
```

MISP is accessible at `https://127.0.0.1:9443`. Log in with the credentials set in `.env`.
