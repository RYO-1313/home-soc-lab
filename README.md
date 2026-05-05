# SIEM-EDR Lab

![Debian](https://img.shields.io/badge/Debian-13.4.0-A81D33?style=flat-square&logo=debian&logoColor=white)
![Splunk](https://img.shields.io/badge/Splunk-10.2.3-000000?style=flat-square&logo=splunk&logoColor=white)
![Wazuh](https://img.shields.io/badge/Wazuh-4.x-3CAAD9?style=flat-square)
![Maintained](https://img.shields.io/badge/Maintained-Every%202%20Months-green?style=flat-square)
![Security](https://img.shields.io/badge/Category-Security%20Lab-orange?style=flat-square)


A step-by-step guide to building a personal security lab with **Splunk** as a SIEM and **Wazuh** as an EDR, running on a single Debian 13 machine. Written to be clear whether you are new to Linux or already comfortable in the terminal.

> **Note:** All download links in this repository are reviewed and updated every **2 months**.

<br>

---

## Overview

| Tool | Role | What it does |
|------|------|--------------|
| **Splunk** | SIEM | Collects and indexes logs and alerts. Lets you search, investigate, and visualize your security data. |
| **Wazuh** | EDR | Monitors the machine in real time. Detects threats, tracks file integrity, and sends alerts to Splunk. |

Wazuh watches. Splunk reports. Together they give you a working security monitoring setup that reflects what you would find in a real SOC.

<br>

---

## Requirements

Tested on the following configuration:

| | |
|---|---|
| OS | Debian 13 (Trixie) — amd64 |
| RAM | 6 GB &nbsp;*(4 GB minimum)* |
| Disk | 50 GB &nbsp;*(41 GB free recommended)* |
| Internet | Required throughout the installation |

To confirm your machine is ready, run:

```bash
uname -m             # must return x86_64
free -h              # check available RAM
df -h                # check available disk space
cat /etc/os-release  # confirm Debian version
```

<br>

---

## Installation

<br>

### Step 1 — Install Debian 13

#### Download the ISO

This guide uses the Debian **13.4.0 netinstall** image for amd64.
The netinstall image is ~800 MB and contains only the core installer — the rest of the system is pulled from the internet during setup, so you always get up-to-date packages.

```
https://cdimage.debian.org/debian-cd/13.4.0/amd64/iso-cd/debian-13.4.0-amd64-netinst.iso
```

<br>

#### Verify the ISO

Always verify the file before using it. This confirms nothing was corrupted or tampered with during download.

Run this in the folder where you saved the ISO:

```bash
sha512sum debian-13.4.0-amd64-netinst.iso
```

The output must match this checksum **exactly**:

```
3e02de4ed744799350bd4039b137053835238ff9f9f29eee812309cd7eebdb5127b0f2b54d167b76d324530aa16939b41ae2d2f2d1995a2801e19938acdc927f
```

If it matches — you are good to go.
If it does not — delete the file and download it again.

<br>

---

### Step 2 — Install Splunk Enterprise

Splunk is the SIEM — the central place where all logs and alerts land.

<br>

#### Download

```bash
wget -O splunk-10.2.3-4d61cf8a5c0c-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.2.3/linux/splunk-10.2.3-4d61cf8a5c0c-linux-amd64.deb"
```

<br>

#### Install

```bash
sudo dpkg -i splunk-10.2.3-4d61cf8a5c0c-linux-amd64.deb
```

When complete, the output should end with:

```
Setting up splunk (10.2.3) ...
complete
```

<br>

#### Start Splunk

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

On first launch, Splunk will ask you to create an **admin username** and **password**.
Write them down — you will need them to access the web interface.

| Flag | What it does |
|------|-------------|
| `--accept-license` | Skips the interactive license prompt |
| `--run-as-root` | Required when running as root. Produces a deprecation warning — safe to ignore in a lab. |

<br>

#### Enable Splunk on boot

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

Splunk will now start automatically after every reboot.

<br>

#### Verify

```bash
sudo /opt/splunk/bin/splunk status --run-as-root
```

Expected output:

```
splunkd is running (PID: XXXXX)
```

<br>

#### Access the web interface

Open a browser and go to:

```
http://localhost:8000
```

To access from another machine on the same network, go to `http://YOUR_IP:8000` and log in with the admin credentials you created. To find your IP:

```bash
hostname -I
```

<br>

---

### Step 3 — Install Wazuh Manager

Wazuh is the EDR — it monitors the machine for threats, suspicious activity, and file changes, then forwards those alerts to Splunk.

<br>

#### Install curl

```bash
sudo apt install curl -y
```

<br>

#### Add the Wazuh GPG key

This lets your system verify that Wazuh packages are authentic:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
```

Confirm it saved correctly:

```bash
ls -la /usr/share/keyrings/wazuh.gpg
```

<br>

#### Add the Wazuh repository

```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
```

<br>

#### Update and install

```bash
sudo apt update
sudo apt install wazuh-manager -y
```

This may take a few minutes.

<br>

#### Enable and start

```bash
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
```

<br>

#### Verify

```bash
sudo systemctl status wazuh-manager
```

Look for these two lines in the output:

```
Loaded: loaded (... enabled ...)
Active: active (running)
```

| | |
|---|---|
| `enabled` | Wazuh will start automatically on boot |
| `active (running)` | Wazuh is running right now |

Both must be present before moving on.

<br>

---

> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months
