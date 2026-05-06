# Wazuh Agent Setup — Windows

This guide installs the Wazuh agent on a Windows machine and connects it to your Wazuh Manager running on the Debian VM. Once connected, Wazuh will monitor your Windows machine and send its alerts to the manager.

<br>

---

## Before you start

By default, VirtualBox uses NAT networking which isolates the VM from your local network. We need to switch to **Bridged Adapter** so your Windows machine and your Debian VM can talk to each other directly.

<br>

### Fix VirtualBox network settings

1. In VirtualBox, select your VM → **Settings → Network**
2. Change **Attached to:** from `NAT` to `Bridged Adapter`
3. Under **Name**, select your physical network card (WiFi or Ethernet)
4. Click **OK**
5. Start the VM again

<br>

### Find your Debian VM's IP address

Once the VM is back up, run:

```bash
ip a
```

Look for an IP that starts with `192.168.x.x` — that is your SEIM VM's IP. In this guide we use `192.168.11.132` as an example. Replace it with your own IP in every command that follows.

<br>

### Verify your Windows machine can reach the VM

On your Windows machine, open **CMD** and run:

```cmd
ping 192.168.11.132
```

Replace `192.168.11.132` with your SEIM IP.

You should see:

```
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

If you get timeouts, double check the Bridged Adapter setting and make sure the VM is running before continuing.

<br>

---

## Installation

Open **CMD as Administrator** on your Windows machine and run the following steps one by one.

<br>

### Step 1 — Download the Wazuh agent

```cmd
curl -o wazuh-agent.msi https://packages.wazuh.com/4.15.5/windows/wazuh-agent-4.14.5-1.msi
```

<br>

### Step 2 — Install the agent

```cmd
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER="192.168.11.132"
```

Replace `192.168.11.132` with your SEIM IP.

`/q` runs the installer silently with no pop-up windows. `WAZUH_MANAGER` tells the agent where to find your Wazuh Manager so it knows where to connect.

<br>

### Step 3 — Start the agent

```cmd
NET START WazuhSvc
```

<br>

---

## Verify the connection

On your **Debian VM**, run:

```bash
sudo /var/ossec/bin/agent_control -l
```

You should see both machines listed and active:

```
ID: 000, Name: SEIM, IP: 127.0.0.1, Active
ID: 001, Name: WINDOWS-PC, IP: 192.168.11.x, Active
```

If your Windows machine appears as `Active`, the agent is connected and sending data to your Wazuh Manager.

<br>

### Verify alerts are arriving in Splunk

Open the Splunk web interface and run this search, replacing `DESKTOP-OGUH4L2` with your Windows machine's hostname:

```
index="wazuh-alerts" agent.name="Your agent name"
```

If no results come back, the Windows agent may not have generated any alerts yet. Force some activity by opening **CMD as Administrator** on your Windows machine and running:

```cmd
net user testuser Test@1234 /add
net user testuser /delete
```

Wait 1-2 minutes, then search again with the time range set to **Last 15 minutes**:

```
index="wazuh-alerts" agent.name="Your agent name"
```

<br>

---

> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months
