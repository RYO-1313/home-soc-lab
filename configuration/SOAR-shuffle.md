# Shuffle SOAR Setup


Shuffle is the SOAR layer in this lab — it sits between Wazuh and Splunk and automates analyst responses using visual no-code playbooks. This guide covers installation on the same Debian 13 VM running Wazuh and Splunk.

> **Note:** All download links in this repository are reviewed and updated every **2 months**.

---

## Why Shuffle

The lab stack consists of Wazuh as the EDR and Splunk as the SIEM, both running on a single Debian 13 machine. Shuffle was added to bridge these tools and introduce real SOAR capability without leaving the self-hosted, open source model.

Shuffle was selected over alternatives for the following reasons. It is fully open source and free when self-hosted, with no feature limits behind a paywall. It deploys via Docker Compose, which is straightforward on Debian. It has a native Wazuh integration, meaning Wazuh can push alerts directly to Shuffle via webhook with minimal configuration. Its visual no-code workflow builder makes it approachable at L1 level while still teaching real automation concepts. It also supports over 2500 app integrations via OpenAPI, which means it can later connect to TheHive, VirusTotal, Slack, and other tools as the lab grows.

The alternative considered was TheHive, but TheHive is a case management platform, not a workflow automation engine. The decision was to use both: Shuffle handles automation playbooks, and TheHive handles case management. Shuffle will create cases in TheHive automatically as part of a playbook.

The intended data flow is: **Wazuh detects → Splunk indexes → Shuffle automates → TheHive tracks cases.**

---

## Environment

| | |
| --- | --- |
| OS | Debian 13 (Trixie) |
| VM IP | 192.168.11.132 |
| Hypervisor | VirtualBox — Bridged Adapter |
| Install path | `/opt/Shuffle` |
| Deployment | Docker Compose |

---

## Installation


### Step 1 — Install Docker

Install the Docker Engine and the Compose plugin from Docker's official repository.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify Docker is working before continuing:

```bash
sudo docker run hello-world
```

You should see a message confirming Docker pulled and ran the test container successfully.

---

### Step 2 — Clone Shuffle

```bash
cd /opt
sudo git clone https://github.com/Shuffle/Shuffle
cd Shuffle
```

---

### Step 3 — Fix OpenSearch Memory Requirement

OpenSearch is Shuffle's internal database. It requires a minimum kernel memory map count of `262144` or it will crash on startup. Set this now and persist it across reboots.

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

The first command applies the setting immediately. The second ensures it survives a reboot.

---

### Step 4 — Create the Database Directory

OpenSearch must be able to write to its data directory. The directory must be owned by UID 1000 or OpenSearch will fail to start.

```bash
sudo mkdir -p /opt/Shuffle/shuffle-database
sudo chown -R 1000:1000 /opt/Shuffle/shuffle-database
```

---

### Step 5 — Start Shuffle

```bash
cd /opt/Shuffle
sudo docker compose up -d
```

Follow the logs to watch the containers come up:

```bash
sudo docker compose logs -f
```

Press `Ctrl+C` to exit the log stream once all containers report healthy.

---

### Step 6 — Verify Containers

```bash
sudo docker ps
```

You should see four containers running:

| Container | Role |
| --- | --- |
| `shuffle-frontend` | Web UI |
| `shuffle-backend` | API and workflow engine |
| `shuffle-orborus` | Container executor |
| `shuffle-opensearch` | Internal database |

If any container shows `Restarting` or `Exited`, check OpenSearch first — it is the most common failure point:

```bash
sudo docker compose logs shuffle-opensearch
```

---

### Step 7 — Enable Shuffle on Boot

Register a cron job to bring the containers up automatically after a reboot:

```bash
sudo crontab -e
```

Add this line at the bottom of the file:

```bash
@reboot cd /opt/Shuffle && docker compose up -d
```

Save and exit. To confirm it was saved:

```bash
sudo crontab -l
```

---

## Access

Open a browser and go to:

```
http://192.168.11.132:3001
```

On first launch, Shuffle will prompt you to create an admin account. This is the only account — write down the credentials.

Shuffle is accessible from any machine on the `192.168.11.x` network thanks to the bridged adapter. To find your VM's current IP if it has changed:

```bash
ip a
```

---

## Default Ports

| Port | Service |
| --- | --- |
| `3001` | Web UI |
| `5001` | Backend API |
| `9200` | OpenSearch |

Verify that port `3001` is free before starting the containers. No conflicts were encountered in this environment, but a pre-existing service on `3001` will prevent the frontend from binding.

---

> Last updated: May 2026 · Links reviewed every 2 months
