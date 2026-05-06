# Wazuh → Splunk Integration

This guide connects Wazuh and Splunk so that every alert Wazuh generates is automatically shipped to Splunk for search and analysis. By the end, you will be able to query Wazuh alerts directly inside Splunk.

<br>

---

## How it works

```
Wazuh Manager
      │
      │  writes alerts to
      ▼
/var/ossec/logs/alerts/alerts.json
      │
      │  monitored by
      ▼
Splunk Universal Forwarder
      │
      │  ships data to port 9999
      ▼
Splunk Enterprise  →  wazuh-alerts index
```

The Splunk Universal Forwarder watches Wazuh's alert file and ships every new line to Splunk in real time.

<br>

---

## Before you start

Make sure both services are up before touching anything else.

<br>

#### Confirm Splunk is running

```bash
sudo /opt/splunk/bin/splunk status --run-as-root
```

Look for `splunkd is running` in the output. If it is not running, start it first:

```bash
sudo /opt/splunk/bin/splunk start --run-as-root
```

<br>

#### Confirm Wazuh Manager is running

```bash
sudo systemctl status wazuh-manager
```

Look for `active (running)` in the output. If it is not running, start it first:

```bash
sudo systemctl start wazuh-manager
```

<br>

#### Confirm Wazuh is generating alerts

```bash
sudo ls -lh /var/ossec/logs/alerts/
```

Then watch the alerts file live:

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json
```

You should see JSON lines flowing in real time. Press `Ctrl+C` to stop.

If the file is empty or missing, wait a few minutes — Wazuh needs some activity on the machine to start generating alerts.

<br>

---

## Configuration

<br>

### Step 1 — Configure Splunk to receive data

This opens a port on Splunk where the forwarder will send Wazuh alerts.

1. Open Splunk at `http://localhost:8000` and log in
2. Go to **Settings** (top menu bar)
3. Click **Forwarding and receiving**
4. Under **Receive data**, click **Configure receiving**
5. Click **New Receiving Port** (top right)
6. Enter `9999` and click **Save**

> You can use any free port. If you choose a different one, replace `9999` with your port in every command that follows.

<br>

### Step 2 — Create the wazuh-alerts index

This is where Splunk will store all Wazuh alerts.

1. Go to **Settings** (top menu bar)
2. Click **Indexes**
3. Click **New Index** (top right)
4. Set the **Index Name** to `wazuh-alerts` and leave everything else as default.

5. Click **Save**

<br>

### Step 3 — Install Splunk Universal Forwarder

The Universal Forwarder is a lightweight agent that reads Wazuh's alerts file and ships it to Splunk.

#### Download

1. Go to `splunk.com/en_us/download/universal-forwarder.html`
2. Log in with your Splunk account
3. Select **Linux → .deb → x86_64**
4. Click the **wget** button to copy the full command with your auth token
5. Paste and run it in your terminal immediately

#### Install

```bash
sudo dpkg -i splunkforwarder-10.2.3-4d61cf8a5c0c-linux-amd64.deb
```

<br>

### Step 4 — Start the forwarder

```bash
sudo /opt/splunkforwarder/bin/splunk start --accept-license --answer-yes
```

It will ask you to create a **username and password** for the forwarder admin. This is separate from your Splunk Enterprise login — set it to anything you will remember.

> **Port conflict:** If you see an error about a port already being in use, enter `y` when prompted and set the new port to `8090`.

<br>

### Step 5 — Point the forwarder to Splunk

Tell the forwarder where to send the data:

```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server 127.0.0.1:9999 --accept-license
```

It will ask for your forwarder admin credentials — use the username and password you just created.

> Replace `9999` with your port if you chose a different one in Step 1.

<br>

### Step 6 — Configure the forwarder to monitor Wazuh alerts

Create the inputs configuration file:

```bash
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```

Paste the following inside:

```ini
[monitor:///var/ossec/logs/alerts/alerts.json]
disabled = 0
host = SEIM
index = wazuh-alerts
sourcetype = wazuh
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`

| Setting | What it does |
|---------|-------------|
| `index = wazuh-alerts` | Tells Splunk which index to store the alerts in |
| `sourcetype = wazuh` | Tags the data so Splunk knows how to parse it |
| `host = SEIM` | Labels the source machine in Splunk |

<br>

### Step 7 — Configure JSON parsing

Create the props configuration so Splunk parses the alerts as JSON instead of plain text:

```bash
sudo nano /opt/splunkforwarder/etc/system/local/props.conf
```

Paste the following inside:

```ini
[wazuh]
DATETIME_CONFIG =
INDEXED_EXTRACTIONS = json
KV_MODE = none
NO_BINARY_CHECK = true
category = Application
disabled = false
pulldown_type = true
```

Save with `Ctrl+O` → `Enter` → `Ctrl+X`

Without this file, Splunk treats each alert as a plain text string. With it, every JSON field — `rule.level`, `agent.name`, `data.srcip` — becomes searchable individually inside Splunk.

<br>

### Step 8 — Enable the forwarder on boot

Enable the forwarder to start automatically on every reboot:

```bash
sudo /opt/splunkforwarder/bin/splunk enable boot-start --run-as-root
sudo /opt/splunkforwarder/bin/splunk start
sudo reboot
```

After the machine comes back up, confirm the forwarder is running:

```bash
sudo /opt/splunkforwarder/bin/splunk status
```

<br>

---

## Verify the connection

Confirm the forwarder is connected to Splunk:

```bash
sudo /opt/splunkforwarder/bin/splunk list forward-server
```

Expected output:

```
Active forwards:
    127.0.0.1:9999
Configured but inactive forwards:
    None
```

If you see `127.0.0.1:9999` under **Active forwards**, the forwarder is connected and shipping data to Splunk.

<br>

To confirm alerts are arriving in Splunk, open the Splunk web interface and run this search:

```
index=wazuh-alerts
```

You should start seeing Wazuh alerts appear within a minute or two.

<br>
<img width="1904" height="894" alt="Screenshot From 2026-05-05 08-05-02" src="https://github.com/user-attachments/assets/74d0aac4-3bd8-440a-893e-3026ca09ea7c" />
---

> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months
