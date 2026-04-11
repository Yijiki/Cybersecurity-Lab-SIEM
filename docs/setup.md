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

Before starting the services, open `docker-compose.yml` and change the nginx port mapping from from `443:443` to `4443:443` to avoid a port conflict:

```bash
nano docker-compose.yml
```

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

Open `docker-compose.yml` and update the misp-core port mapping to `"${CORE_HTTPS_PORT}:443"`:

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

### Step 1: Windows Endpoint Setup

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

### Step 2: Linux Endpoint Setup

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

#### Create an Organisation

Navigate to **Organisation Management** and click **Create Organisation**. Set the name to `SOC-Lab` and the description to `Primary SOC`.

#### Create Users

Inside the SOC-Lab organisation, create the following three users. After saving each user, click into their profile to set a password. For the `shuffle` service account, generate an API key instead and save it somewhere secure.

| Login   | Type    | Role      | Purpose                      |
|---------|---------|-----------|------------------------------|
| admin   | Normal  | org-admin | Organisation administrator   |
| analyst | Normal  | analyst   | SOC analyst account          |
| shuffle | Service | analyst   | Shuffle SOAR service account |

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

When enabling VirusTotal and URLhaus, enter their respective API keys in the configuration fields.

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

### Step 3: MISP Setup

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

## Phase 4 - Detection Engineering

### Step 1: Custom Wazuh Rules

Custom rules are written in `local_rules.xml` on the Wazuh Manager VM. Open the file for editing:

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

  <!-- Suppression: False Positive from Microsoft Edge Update -->
  <rule id="100011" level="0">
    <if_sid>100010</if_sid>
    <field name="win.eventdata.image">\\MicrosoftEdgeUpdate\.exe</field>
    <description>Suppression: False Positive from Microsoft Edge Update</description>
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

> For a breakdown of each rule's detection logic, see [`docs/rules.md`](rules.md).

Once saved, restart the Wazuh Manager to apply the new rules:

```bash
sudo systemctl restart wazuh-manager
```

## Phase 5 - SOAR Automation (Shuffle)

### Overview

The completed workflow follows this logic:

```
Endpoints (Win / Ubuntu)
      ↓ sends events
Wazuh SIEM
      ↓ detects & alerts
Shuffle SOAR
      ↓ receives webhook — parses JSON (rule, agent, command data)
Regex node extracts SHA256 hash from Sysmon event
      ↓
Cortex queries VirusTotal for the hash
      ↓
TheHive alert created with host info and VT score
      ↓
Condition: VT malicious count > 3?
      ↓ YES
Wazuh active response isolates the host
      ↓
SOC analyst reviews the TheHive alert             
```

---

### Step 1: Connect Wazuh to Shuffle

#### Create the Webhook

In Shuffle, create a **AutoResponse** workflow and drag a **Webhook** node onto the canvas. Name it `wazuh-alert`. Copy the webhook URL - it will look like:

```
http://127.0.0.1:3001/api/v1/hooks/webhook_xxxxxxxx
```

#### Point Wazuh at Shuffle

On the Wazuh Manager VM, open `ossec.conf` for editing:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following block, replacing the URL with your copied webhook URL. The `rule_id` tag ties this integration to the PowerShell download rule:

```xml
<integration>
  <name>shuffle</name>
  <hook_url>http://127.0.0.1:3001/api/v1/hooks/YOUR_WEBHOOK_ID</hook_url>
  <rule_id>100010</rule_id>
  <alert_format>json</alert_format>
</integration>
```

Restart the Wazuh Manager:

```bash
sudo systemctl restart wazuh-manager
```

---

### Step 2: Regex Node - Extract SHA256 Hash

Wazuh sends Sysmon hash data in the following format:

```
MD5=abc123...,SHA256=def456...,IMPHASH=A1B2C3D4...
```

Only the SHA256 value is needed for VirusTotal enrichment.

Drag a **Shuffle Tools** node onto the canvas and connect it to the Webhook node. Configure it as follows:

| Field      | Value                              |
|------------|------------------------------------|
| Action     | Regex capture group                |
| Input data | `$exec.text.win.eventdata.hashes`  |
| Regex      | `SHA256=([A-Fa-f0-9]{64})`         |

The captured hash is referenced in subsequent nodes as `$regex_capture_group.group_0`.

---

### Step 3: Cortex Node - VirusTotal Hash Lookup

Search for **Cortex** in the Shuffle app search bar and drag it onto the canvas. Connect it to the Regex node. When prompted, enter your Cortex API key and URL to authenticate the node. Configure it as follows:

| Field    | Value                           |
|----------|---------------------------------|
| Action   | Run analyzer                    |
| Analyzer | VirusTotal_GetReport_3_1        |
| Data     | `$regex_capture_group.group_0`  |

Cortex submits the hash to VirusTotal and returns a JSON result containing a `malicious` count - the number of antivirus engines that flagged the file. This value is referenced in subsequent nodes as `$run_analyzer.summary.malicious`.

---

### Step 4: TheHive Node - Create Alert

Search for **TheHive** in the Shuffle app search bar and drag it onto the canvas. Connect it to the Cortex node. When prompted, enter your TheHive API key and URL to authenticate the node. Configure it as follows:

| Field       | Value                                                         |
|-------------|---------------------------------------------------------------|
| Action      | Create alert                                                  |
| Title       | `[Automated] Suspicious PowerShell on $exec.text.agent.name`  |
| Severity    | 3 (High)                                                      |
| Description | See template below                                            |

Use the following as the description template, which pulls in all relevant data from earlier nodes:

```
Alert ID: $exec.text.id
Host: $exec.text.agent.name ($exec.text.agent.ip)
Rule: $exec.text.rule.description (ID: $exec.text.rule.id)
Command: $exec.text.win.eventdata.commandLine
VirusTotal Score: $run_analyzer.summary.malicious / 72
Action Taken: Automated isolation pending...
```

---

### Step 5: Wazuh Node - Active Response (Host Isolation)

Search for **Wazuh** in the Shuffle app search bar and drag it onto the canvas. Connect it to the TheHive node. When prompted, enter your Wazuh API credentials to authenticate the node. Configure it as follows:

| Field    | Value                  |
|----------|------------------------|
| Action   | Run command            |
| Command  | `drop-firewall`        |
| Agent ID | `$exec.text.agent.id`  |

`drop-firewall` is a built-in Wazuh active response command that blocks all inbound and outbound traffic on the agent using Windows Firewall, isolating the host from the network while leaving the Wazuh agent connection intact.

The `agent.id` value pulled from the webhook payload ensures the isolation targets the specific host that triggered the alert rather than applying broadly.

---

### Step 6: Condition - VirusTotal Score Gate

Rather than isolating a host on every alert, the workflow uses a condition to gate the active response. This helps prevent automated isolation from triggering on false positives.

Click the **connector line** between the TheHive node and the Wazuh node. Add a condition with the following logic:

| Field    | Value                             |
|----------|-----------------------------------|
| Variable | `$run_analyzer.summary.malicious` |
| Operator | GREATER THAN                      |
| Value    | `3`                               |

The Wazuh isolation node only executes if 3 or more VirusTotal engines flag the hash as malicious.

---

## Phase 6 - Adversary Simulation & Validation

### Step 1: Prepare the Windows Endpoint

Before running any simulations, disable Windows Defender on the Windows 11 VM to prevent it from blocking Atomic Red Team test payloads before Wazuh has a chance to detect them. Navigate to **Windows Security → Virus & Threat Protection → Manage Settings** and disable the following:

- **Real-time protection**
- **Cloud-delivered protection**
- **Automatic sample submission**
- **Tamper protection**

### Step 2: Install Atomic Red Team

Open PowerShell as Administrator on the Windows 11 VM and run the following to set the execution policy and install Atomic Red Team along with its test library:

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing);

Install-AtomicRedTeam -getAtomics
```

### Step 3: Run Attack Simulations

Each test below maps to one of the custom Wazuh rules written in Phase 4. Run them one at a time and validate the full pipeline after each before moving to the next.

#### T1105 - Suspicious PowerShell Download (Rule 100010)

```powershell
Invoke-AtomicTest T1105 -TestNumbers 10
```

This simulates a file download using native PowerShell download methods, triggering rule 100010. This is also the rule tied to the Shuffle webhook integration, so this test exercises the full automated pipeline end to end.

**Expected result:** Wazuh fires a level 14 alert → Shuffle receives the webhook → Cortex queries VirusTotal → TheHive alert is created → if the VirusTotal score exceeds 3, the host is isolated via `drop-firewall`.

#### T1547.001 - Registry Run Key Modification (Rule 100012)

```powershell
Invoke-AtomicTest T1547.001 -TestNumbers 1
```

This writes a value to a `CurrentVersion\Run` registry key, simulating a persistence mechanism that survives user login. Triggers rule 100012.

**Expected result:** Wazuh fires a level 13 alert visible in the dashboard.

#### T1053.005 - Scheduled Task Creation (Rule 100013)

```powershell
Invoke-AtomicTest T1053.005 -TestNumbers 1
```

This creates a scheduled task via command line using `schtasks /create`, simulating attacker-driven persistence or execution scheduling. Triggers rule 100013.

**Expected result:** Wazuh fires a level 12 alert visible in the dashboard.

> Full execution details and results are documented in [`attack_simulations.md`](attack_simulations.md).

---

### Step 4: Validate the Pipeline

After running the T1105 simulation, verify each stage of the pipeline completed successfully:

**Wazuh** - Navigate to the Wazuh dashboard and confirm a level 14 alert fired for rule 100010 against the `vic-win11` agent. The alert should include the full command line in the event data.

**Shuffle** - Open the AutoResponse workflow and check the execution history. Each node should show a green status. Click through the Regex, Cortex and TheHive nodes to confirm data was passed correctly between them.

**TheHive** - Log in as the analyst user and confirm a new case was created automatically with the correct title, host details, rule information and VirusTotal score populated in the description.

**Cortex** - Navigate to the Cortex job history and confirm a VirusTotal analyzer job ran and returned a result.

**Active Response** - If the VirusTotal score exceeded the condition threshold, confirm the Windows VM lost network connectivity while remaining visible as an active agent in the Wazuh dashboard.




