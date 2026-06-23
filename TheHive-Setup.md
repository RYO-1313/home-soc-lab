# TheHive — SOC Case Management Setup

![TheHive](https://img.shields.io/badge/TheHive-5.1.4-FEAE00?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.x-3CAAD9?style=flat-square)
![Windows](https://img.shields.io/badge/Host-Windows%2010%20Pro-0078D6?style=flat-square&logo=windows&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-WSL2%20%2B%20Docker-informational?style=flat-square)

> **A production-style TheHive deployment integrated with Wazuh — turning raw EDR alerts into structured, trackable security cases.**

---

## Objective

In a real SOC, alerts don't live in isolation. Analysts triage them, escalate them, and document every step. This lab adds that layer to the existing Wazuh + Splunk stack by deploying **TheHive** as a case management platform.

Wazuh detects. TheHive tracks. The result is a full alert-to-case pipeline that mirrors how a blue team actually operates — from initial detection to documented resolution.

**Skills demonstrated:**
- Docker-based service deployment on Windows (WSL2)
- Multi-service orchestration with Compose (TheHive + Cassandra + Elasticsearch)
- Custom Wazuh integration scripting (Python)
- SOC workflow: alert intake → triage → case management

---

## Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| SIEM / EDR | Debian (Wazuh Manager) | `192.168.11.132` |
| TheHive Host | Windows 10 Pro | `192.168.11.50` |
| Victim Endpoint | Windows | `192.168.11.116` |
| Attacker | Kali Linux | — |

---

## Stack

| Component | Role |
|-----------|------|
| **TheHive 5.1.4** | Case management — receives alerts and tracks analyst workflow |
| **Apache Cassandra 4** | Backend database — stores cases, observables, and audit trail |
| **Elasticsearch 7.17** | Search index — powers TheHive's query and filter engine |
| **Docker Compose** | Orchestrates all three services as a single deployable stack |
| **Wazuh (custom integration)** | Pushes level 7+ alerts from the EDR into TheHive as structured alerts |

---

## Deployment Overview

TheHive runs inside Docker on a Windows 10 Pro machine using WSL2 as the Linux backend. Three containers run together: Cassandra handles persistent storage, Elasticsearch handles indexing, and TheHive sits on top of both. A custom Python script on the Wazuh manager POSTs alerts to TheHive's API whenever a rule fires at level 7 or above.

---

## Installation

### Part 1 — Prepare the Windows Machine

#### Enable WSL2

Open PowerShell as Administrator:

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

Restart the machine, then install Ubuntu and set WSL2 as default:

```powershell
wsl --install -d Ubuntu
wsl --set-default-version 2
```

#### Install Docker Desktop

Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop). During installation, select **Use WSL2 instead of Hyper-V**. Once Docker Desktop is running and the whale icon stops animating:

```powershell
docker --version
docker compose version
```

Both commands must return a version number before continuing.

---

### Part 2 — Deploy TheHive with Docker

#### Create the project folder

```powershell
mkdir C:\thehive
cd C:\thehive
```

#### Create the Compose file

```powershell
notepad docker-compose.yml
```

Paste the following:

```yaml
version: "3.8"
services:

  cassandra:
    image: cassandra:4
    container_name: cassandra
    restart: unless-stopped
    environment:
      CASSANDRA_CLUSTER_NAME: thehive
    volumes:
      - cassandra_data:/var/lib/cassandra
    healthcheck:
      test: ["CMD", "cqlsh", "-e", "describe keyspaces"]
      interval: 30s
      timeout: 10s
      retries: 10

  elasticsearch:
    image: elasticsearch:7.17.12
    container_name: elasticsearch
    restart: unless-stopped
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  thehive:
    image: strangebee/thehive:5.1.4-1
    container_name: thehive
    restart: unless-stopped
    depends_on:
      - cassandra
      - elasticsearch
    ports:
      - "9000:9000"
    environment:
      - JVM_OPTS=-Xms512m -Xmx512m
    command:
      - --secret
      - "MySecretForTheHiveLabKey123"
      - "--cql-hostnames"
      - "cassandra"
      - "--index-backend"
      - "elasticsearch"
      - "--es-hostnames"
      - "elasticsearch"

volumes:
  cassandra_data:
  elasticsearch_data:
```

#### Start the stack

```powershell
docker compose up -d
```

Wait 2–3 minutes for all containers to initialize, then verify:

```powershell
docker ps
```

All three containers — `cassandra`, `elasticsearch`, `thehive` — must show status `Up`.

#### Confirm TheHive is listening

```powershell
docker logs thehive --tail 20
```

Look for: `Listening for HTTP on /0:0:0:0:0:0:0:0:9000`

---

### Part 3 — Configure TheHive

#### First login

Navigate to `http://192.168.11.50:9000` and log in with the default credentials:

| Field | Value |
|-------|-------|
| Email | `admin@thehive.local` |
| Password | `secret` |

Change the admin password immediately after first login.

#### Create the SOC organization

**Admin → Organizations → + → New Organization**

| Field | Value |
|-------|-------|
| Name | `SOC-Lab` |
| Description | `Home SOC Lab` |

#### Create users

Inside the `SOC-Lab` organization, create two accounts:

**Analyst account** (daily use):

| Field | Value |
|-------|-------|
| Login | `analyst@soc.lab` |
| Full Name | `SOC Analyst` |
| Profile | `analyst` |

**Wazuh service account** (API integration):

| Field | Value |
|-------|-------|
| Login | `wazuh@soc.lab` |
| Full Name | `wazuh` |
| Profile | `analyst` |

After creating the Wazuh account: click `...` → **Create API Key** → copy and save the key. You will need it in the next part.

---

### Part 4 — Connect Wazuh to TheHive

#### Create the integration script

SSH into the Wazuh manager (`192.168.11.132`) and create the script:

```bash
sudo nano /var/ossec/integrations/custom-thehive
```

Paste the following, replacing `YOUR_API_KEY_HERE` with the key you saved:

```python
#!/usr/bin/env python3
import json
import sys
import requests
from datetime import datetime

THEHIVE_URL = "http://192.168.11.50:9000"
THEHIVE_API_KEY = "YOUR_API_KEY_HERE"

def create_alert(alert):
    url = THEHIVE_URL + "/api/v1/alert"
    headers = {
        "Authorization": "Bearer " + THEHIVE_API_KEY,
        "Content-Type": "application/json"
    }
    rule = alert.get("rule", {})
    agent = alert.get("agent", {})
    title = "Wazuh Alert: " + rule.get("description", "Unknown")
    description = "Rule ID: " + str(rule.get("id", "N/A")) + "\nAgent: " + agent.get("name", "N/A")
    level = int(rule.get("level", 1))
    severity = min(max(level // 4 + 1, 1), 4)
    payload = {
        "title": title,
        "description": description,
        "type": "wazuh",
        "source": "wazuh",
        "sourceRef": str(alert.get("id", "N/A")),
        "severity": severity,
        "tags": ["wazuh", "rule-" + str(rule.get("id", "N/A"))]
    }
    try:
        response = requests.post(url, headers=headers, json=payload, verify=False)
        response.raise_for_status()
    except Exception as e:
        with open("/var/ossec/logs/thehive-error.log", "a") as f:
            f.write(str(datetime.now()) + " - Error: " + str(e) + "\n")

if __name__ == "__main__":
    try:
        with open(sys.argv[1]) as f:
            alert = json.load(f)
        create_alert(alert)
    except Exception as e:
        with open("/var/ossec/logs/thehive-error.log", "a") as f:
            f.write(str(datetime.now()) + " - Error: " + str(e) + "\n")
```

#### Set permissions

```bash
sudo chmod 750 /var/ossec/integrations/custom-thehive
sudo chown root:wazuh /var/ossec/integrations/custom-thehive
```

#### Register the integration with Wazuh

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add the following block just before the closing `</ossec_config>` tag:

```xml
<integration>
  <name>custom-thehive</name>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

#### Restart Wazuh

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

#### Confirm the integration loaded

```bash
sudo grep -i "thehive\|integration" /var/ossec/logs/ossec.log | tail -5
```

Look for: `Enabling integration for: 'custom-thehive'`

---

### Part 5 — Test

#### Trigger an alert

From any machine on the network, attempt a failed SSH login to the Wazuh manager:

```bash
ssh fakeuser@192.168.11.132
```

This fires a Wazuh authentication failure rule at level 5+. Depending on the number of attempts, it escalates to level 10 (multiple failures), which clears the level 7 threshold.

#### Verify in TheHive

1. Open `http://192.168.11.50:9000`
2. Log in as `analyst@soc.lab`
3. Click **Alerts** in the left sidebar
4. Wazuh alerts should appear within 30–60 seconds

---

## Alert-to-Case Flow

```
Wazuh detects event (rule level ≥ 7)
        ↓
custom-thehive script fires
        ↓
POST /api/v1/alert → TheHive
        ↓
Alert appears in TheHive UI
        ↓
Analyst reviews, promotes to Case
        ↓
Case tracked through triage → investigation → close
```

---

## Screenshots

| | |
|--|--|
| TheHive alert inbox | `screenshots/thehive-alerts.png` |
| Wazuh → TheHive alert in UI | `screenshots/wazuh-alert-in-thehive.png` |
| Case view | `screenshots/case-view.png` |

*[TODO: add screenshots from your lab]*

---

## Takeaways

This setup demonstrates three things relevant to a L1 SOC role:

**Alert pipeline ownership.** Building the Wazuh-to-TheHive integration from scratch — not clicking a pre-built connector — means understanding how alerts move between tools at the API level.

**Case management workflow.** TheHive mirrors what analysts use in production (SIRP platforms like TheHive, IBM SOAR, or Jira Service Management). Knowing how to intake an alert, enrich it, and close it with documentation is core L1 work.

**Multi-platform lab management.** Running TheHive on Windows (WSL2 + Docker) while Wazuh runs on Debian reflects real heterogeneous environments — not every SOC tool runs on the same OS.

---

## Repo Structure

```
thehive-setup/
├── README.md
├── docker-compose.yml
├── integrations/
│   └── custom-thehive          # Wazuh integration script
├── config/
│   └── ossec-integration.xml   # Config snippet for ossec.conf
└── screenshots/
    ├── thehive-alerts.png
    ├── wazuh-alert-in-thehive.png
    └── case-view.png
```

---

## Next Steps

- [ ] Add Cortex for automated observable enrichment (VirusTotal, Shodan)
- [ ] Connect Splunk as a second alert source
- [ ] Build case templates for common Wazuh rule categories
- [ ] Document a full triage walkthrough using a real alert

---

> Part of the [SIEM-EDR Home Lab](../README.md) · Last updated: June 2026
