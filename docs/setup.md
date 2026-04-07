# Setup Guide

## Phase 1 - Infrastructure

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

## Phase 2 - Endpoint Configuration

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

## Phase 3 - Integrations

### Step 1: TheHive Organisation Setup

Access TheHive at `http://127.0.0.1:9000/thehive` and log in with the default credentials `admin@thehive.local / secret`.

#### Create the Organisation

Navigate to **Organisation Management** and click **Create Organisation**. Set the name to `SOC-Lab` and the description to `Primary SOC`.

#### Create Users

Inside the SOC-Lab organisation, create the following three users. After saving each user, click into their profile to set a password. For the `shuffle` service account, generate an API key instead and save it somewhere secure.

| Login         | Role      | Purpose                        |
|---------------|-----------|--------------------------------|
| admin         | org-admin | Organisation administrator     |
| analyst       | analyst   | SOC analyst account            |
| shuffle       | analyst   | Shuffle SOAR service account   |

---

### Step 2: Cortex Setup

Access Cortex at `http://127.0.0.1:9001/cortex`. On first access, click **Update the database** and then create an admin account.

#### Create the Organisation

Click **Add Organisation** and set the name to `SOC-Lab` and the description to `Primary SOC`.

#### Create an Org Admin User

Click on the SOC-Lab organisation, navigate to the **Users** tab and click **Add User** with the following details:

- **Login:** admin@soc.lab
- **Name:** SOC Admin
- **Roles:** org-admin, read, analyze

Save the user, then click into their profile and set a password.

#### Enable Analyzers

Log out of the global admin account and log back in as `admin@soc.lab`. Navigate to **Organisation → Analyzers** and enable the following:

| Analyzer                  | Version | Purpose               | API Key Required |
|---------------------------|---------|-----------------------|------------------|
| VirusTotal_GetReport_3_1  | 3.1     | Hash/IP/domain lookup | Yes (free)       |
| URLhaus_2_0               | 2.0     | Malicious URL check   | Yes (free)       |
| Abuse_Finder_3_0          | 3.0     | Abuse contact lookup  | No               |
| FileInfo_8_0              | 8.0     | File metadata         | No               |

When enabling VirusTotal and URLhaus, enter the respective free API keys in the configuration fields.

#### Fix Cortex Job Directory Permissions

The Cortex container runs as UID 1000 but the job output directory is owned by root, which prevents analyzers from writing results. Run the following on the soc-core VM to fix this:

```bash
cd ~/docker/testing

sudo chown -R $USER:$USER ./cortex/cortex-jobs
chmod 777 ./cortex/cortex-jobs

docker compose restart cortex
```

#### Connect Cortex to TheHive

In TheHive, navigate to **Platform Management → Connectors → Cortex** and click **Add**. Configure as follows:

- **URL:** `http://10.0.5.x:9001/cortex` (replace with your soc-core VM's IP)
- **API Key:** Generate a new API key in Cortex under **Organisation → Users**, copy it and paste it here
- **Check certificate authority:** Unchecked

Click **Confirm**.

---

### Step 3: MISP Initial Configuration

Access MISP at `https://127.0.0.1:9443` and log in with the credentials set in your `.env` file during installation.

#### Enable Threat Intelligence Feeds

Navigate to **Sync Actions → Feeds** and enable the following feeds:

- CIRCL OSINT Feed
- Botvrij.eu Data

Once enabled, click **Fetch and store all feed data** to perform the initial pull.

#### Generate a MISP API Key

Navigate to **Administration → List Auth Keys → Add Authentication Key**. Generate a key for the admin account and save it.

#### Connect MISP to TheHive

In TheHive, navigate to **Platform Management → Connectors → MISP** and click **Add**. Configure as follows:

- **Server URL:** `https://10.0.5.x:9443` (replace with your soc-core VM's IP)
- **API Key:** The key generated in the previous step
- **Purpose:** Import and Export
- **Proxy:** Leave empty
- **Certificate Authority:** Unchecked
- **Host name verification:** Unchecked
- **Export case tags:** Checked
- **Observable tags:** Checked
- **Export Hive URL:** Checked

Click **Confirm**.

## Phase 4 — Detection Engineering

### Step 5: Custom Wazuh Rules

Custom rules are written in `local_rules.xml` on the Wazuh Manager. Open the file for editing:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the following rule group beneath the existing example content:

```xml
<group name="atomic_red_team,">

  <!-- Process Creation w/ Suspicious PowerShell Download -->
  <rule id="100010" level="14">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)(Invoke-WebRequest|iwr|Net.WebClient|curl)</field>
    <description>Process creation w/ Suspicious PowerShell download</description>
    <mitre>
      <id>T1105</id>
    </mitre>
  </rule>

  <!-- Suppress false positives from Microsoft Edge Update -->
  <rule id="100011" level="0">
    <if_sid>100010</if_sid>
    <field name="win.eventdata.image">\\MicrosoftEdgeUpdate\.exe</field>
    <description>Ignore False Positives from Microsoft Edge Update pings</description>
  </rule>

  <!-- Registry Run(Once)(Ex) Key Modification -->
  <rule id="100012" level="13">
    <if_group>sysmon_event_13</if_group>
    <field name="win.eventdata.targetObject" type="pcre2">(?i)HKU\\.*\\CurrentVersion\\Run(Once)?(EX)?</field>
    <description>Registry Run keys modified - Possible persistence mechanism</description>
    <mitre>
      <id>T1547.001</id>
    </mitre>
  </rule>

  <!-- Suspicious Scheduled Task Creation via Command Line -->
  <rule id="100013" level="12">
    <if_group>sysmon_event1</if_group>
    <field name="win.eventdata.commandLine" type="pcre2">(?i)(schtasks.*\/create|New-ScheduledTask|Register-ScheduledTask)</field>
    <description>Suspicious Scheduled Task Creation via Command Line</description>
    <mitre>
      <id>T1053.005</id>
    </mitre>
  </rule>

</group>
```

Once saved, restart the Wazuh Manager to apply the new rules:

```bash
sudo systemctl restart wazuh-manager
```


