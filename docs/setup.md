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

## Phase 2 — Endpoint Configuration

### Step 6: Windows Endpoint Setup

Create a new VM with the following specifications:

- **OS:** Windows 11 Enterprise Evaluation LTSC
- **RAM:** 4GB (reduced to 2GB after initial installation)
- **Disk:** 45GB
- **vCPUs:** 2
- **Network:** soc-net

#### Install Wazuh Agent

Log into the Wazuh dashboard from your host at `https://127.0.0.1:8443`. Navigate to **Deploy new agent**, select **Windows (MSI 32/64 bits)**, enter your Wazuh Manager's IP (soc-core VM's IP) and give the agent a name such as `vic-win11`. Assign it to the `default` group.

Copy the generated PowerShell command, open PowerShell as Administrator inside the Windows VM and paste it. Once the installation completes, start the agent service:

```powershell
Start-Service -Name "Wazuh"
```

#### Install Sysmon

Download the [Sysinternals Sysmon ZIP](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) and extract it. Download the [SwiftOnSecurity Sysmon config](https://github.com/SwiftOnSecurity/sysmon-config) and place `sysmonconfig-export.xml` in the same folder as the extracted Sysmon files.

In an Administrator PowerShell, navigate to the folder and install Sysmon with the config:

```powershell
.\Sysmon64.exe -i sysmonconfig-export.xml -accepteula
```

#### Configure Wazuh to Ingest Sysmon Logs

By default, Wazuh does not monitor Sysmon logs. The agent config needs to be updated to tell it where to look.

Open the Wazuh agent config in an Administrator PowerShell:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Add the following block before the closing `</ossec_config>` tag:

```xml
<localfile>
  <log_format>eventchannel</log_format>
  <location>Microsoft-Windows-Sysmon/Operational</location>
</localfile>
```

Save the file and restart the Wazuh agent:

```powershell
Restart-Service -Name "Wazuh"
```

---

### Step 7: Linux Endpoint Setup

Create a new VM with the following specifications:

- **OS:** Ubuntu 22.04 Desktop
- **RAM:** 2GB
- **Disk:** 25GB
- **vCPUs:** 2
- **Network:** soc-net

#### Install Wazuh Agent

Log into the Wazuh dashboard from your host at `https://127.0.0.1:8443`. Navigate to **Deploy new agent**, select **Linux (DEB amd64)**, enter your Wazuh Manager's IP and give the agent a name such as `vic-ubuntu`. Assign it to the `default` group.

Copy the generated command, open a terminal inside the Ubuntu VM and paste it. Once the installation completes, enable and start the agent service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Both agents should now appear as active in the Wazuh dashboard under **Agents**.

