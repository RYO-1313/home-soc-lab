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
curl -o wazuh-agent.msi https://packages.wazuh.com/4.x/windows/wazuh-agent-4.15.5-1.msi
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

<img width="1919" height="860" alt="Screenshot From 2026-05-06 21-49-23" src="https://github.com/user-attachments/assets/b9f34a6e-5ddd-4cbb-adc1-3de9289a394b" />

---
> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months

---
---

# Wazuh Agent Setup — Debian-based Linux
This guide installs the Wazuh agent on a Debian-based Linux machine (Ubuntu, Debian, Kali, Linux Mint) and connects it to your Wazuh Manager running on the Debian VM. Once connected, Wazuh will monitor your machine and send its alerts to the manager.
<br>

---
## Before you start
By default, VirtualBox uses NAT networking which isolates the VM from your local network. We need to switch to **Bridged Adapter** so your Linux machine and your Debian VM can talk to each other directly.
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

### Verify your Linux machine can reach the VM
On your Linux machine, open a terminal and run:
```bash
ping 192.168.11.132
```
Replace `192.168.11.132` with your SEIM IP.
You should see:
```
64 bytes from 192.168.11.132: icmp_seq=1 ttl=64 time=0.xxx ms
```
If you get timeouts, double check the Bridged Adapter setting and make sure the VM is running before continuing.
<br>

---
## Installation
Open a terminal and run the following steps one by one. All commands require root privileges — prefix them with `sudo` or switch to root with `sudo -i`.
<br>

### Step 1 — Install required packages
```bash
sudo apt-get install gnupg apt-transport-https curl -y
```
<br>

### Step 2 — Import the Wazuh GPG key
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg
```
<br>

### Step 3 — Add the Wazuh repository
```bash
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
```
<br>

### Step 4 — Install the Wazuh agent
```bash
sudo apt-get update
WAZUH_MANAGER="192.168.11.132" sudo apt-get install wazuh-agent -y
```
Replace `192.168.11.132` with your SEIM IP. `WAZUH_MANAGER` tells the agent where to find your Wazuh Manager so it knows where to connect.
<br>

### Step 5 — Enable and start the agent
```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
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
ID: 001, Name: YOUR-LINUX-PC, IP: 192.168.11.x, Active
```
If your Linux machine appears as `Active`, the agent is connected and sending data to your Wazuh Manager.
<br>

### Verify alerts are arriving in Splunk
Open the Splunk web interface and run this search, replacing `YOUR-HOSTNAME` with your Linux machine's hostname:
```
index="wazuh-alerts" agent.name="YOUR-HOSTNAME"
```
If no results come back, the agent may not have generated any alerts yet. Force some activity by running on your Linux machine:
```bash
sudo ls /etc
sudo cat /etc/shadow
```
Wait 1-2 minutes, then search again with the time range set to **Last 15 minutes**:
```
index="wazuh-alerts" agent.name="YOUR-HOSTNAME"
```
<br>

---
> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months

---
---

# Wazuh Agent Setup — Red Hat-based Linux
This guide installs the Wazuh agent on a Red Hat-based Linux machine (RHEL, CentOS, Fedora, Amazon Linux) and connects it to your Wazuh Manager running on the Debian VM. Once connected, Wazuh will monitor your machine and send its alerts to the manager.
<br>

---
## Before you start
By default, VirtualBox uses NAT networking which isolates the VM from your local network. We need to switch to **Bridged Adapter** so your Linux machine and your Debian VM can talk to each other directly.
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

### Verify your Linux machine can reach the VM
On your Linux machine, open a terminal and run:
```bash
ping 192.168.11.132
```
Replace `192.168.11.132` with your SEIM IP.
You should see:
```
64 bytes from 192.168.11.132: icmp_seq=1 ttl=64 time=0.xxx ms
```
If you get timeouts, double check the Bridged Adapter setting and make sure the VM is running before continuing.
<br>

---
## Installation
Open a terminal and run the following steps one by one. All commands require root privileges — prefix them with `sudo` or switch to root with `sudo -i`.
<br>

### Step 1 — Import the Wazuh GPG key
```bash
sudo rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
```
<br>

### Step 2 — Add the Wazuh repository
```bash
cat > /etc/yum.repos.d/wazuh.repo << EOF
[wazuh]
gpgcheck=1
gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
enabled=1
name=EL-\$releasever - Wazuh
baseurl=https://packages.wazuh.com/4.x/yum/
protect=1
EOF
```
<br>

### Step 3 — Install the Wazuh agent
```bash
WAZUH_MANAGER="192.168.11.132" sudo yum install wazuh-agent -y
```
Replace `192.168.11.132` with your SEIM IP. `WAZUH_MANAGER` tells the agent where to find your Wazuh Manager so it knows where to connect.

> **Note:** On Fedora or RHEL 8+ you can replace `yum` with `dnf`.
<br>

### Step 4 — Enable and start the agent
```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
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
ID: 001, Name: YOUR-LINUX-PC, IP: 192.168.11.x, Active
```
If your Linux machine appears as `Active`, the agent is connected and sending data to your Wazuh Manager.
<br>

### Verify alerts are arriving in Splunk
Open the Splunk web interface and run this search, replacing `YOUR-HOSTNAME` with your Linux machine's hostname:
```
index="wazuh-alerts" agent.name="YOUR-HOSTNAME"
```
If no results come back, the agent may not have generated any alerts yet. Force some activity by running on your Linux machine:
```bash
sudo ls /etc
sudo cat /etc/shadow
```
Wait 1-2 minutes, then search again with the time range set to **Last 15 minutes**:
```
index="wazuh-alerts" agent.name="YOUR-HOSTNAME"
```
<br>

---
> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months

---
---

# Wazuh Agent Setup — Arch-based Linux
This guide installs the Wazuh agent on an Arch-based Linux machine (Arch Linux, CachyOS, Manjaro, EndeavourOS) and connects it to your Wazuh Manager running on the Debian VM. Once connected, Wazuh will monitor your machine and send its alerts to the manager.

> **Note:** Wazuh does not provide an official package for Arch-based systems. Installation is done via the **AUR (Arch User Repository)** using an AUR helper like `yay` or `paru`.
<br>

---
## Before you start
By default, VirtualBox uses NAT networking which isolates the VM from your local network. We need to switch to **Bridged Adapter** so your Linux machine and your Debian VM can talk to each other directly.
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

### Verify your Linux machine can reach the VM
On your Linux machine, open a terminal and run:
```bash
ping 192.168.11.132
```
Replace `192.168.11.132` with your SEIM IP.
You should see:
```
64 bytes from 192.168.11.132: icmp_seq=1 ttl=64 time=0.xxx ms
```
If you get timeouts, double check the Bridged Adapter setting and make sure the VM is running before continuing.
<br>

---
## Installation
Open a terminal and run the following steps one by one. Do **not** run AUR commands as root — run them as your regular user.
<br>

### Step 1 — Install an AUR helper (if not already installed)
If you don't have `yay` installed:
```bash
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```
<br>

### Step 2 — Install the Wazuh agent from AUR
```bash
yay -S wazuh-agent
```
<br>

### Step 3 — Set the Wazuh Manager IP
```bash
sudo sed -i 's/<address>MANAGER_IP<\/address>/<address>192.168.11.132<\/address>/' /var/ossec/etc/ossec.conf
```
Replace `192.168.11.132` with your SEIM IP. This tells the agent where to find your Wazuh Manager so it knows where to connect.
<br>

### Step 4 — Fix permissions
```bash
sudo chown -R wazuh:wazuh /var/ossec
```
<br>

### Step 5 — Enable and start the agent
```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
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
ID: 001, Name: YOUR-ARCH-PC, IP: 192.168.11.x, Active
```
If your Arch machine appears as `Active`, the agent is connected and sending data to your Wazuh Manager.
<br>

### Verify alerts are arriving in Splunk
Open the Splunk web interface and run this search, replacing `YOUR-HOSTNAME` with your Arch machine's hostname:
```
index="wazuh-alerts" agent.name="YOUR-HOSTNAME"
```
If no results come back, the agent may not have generated any alerts yet. Force some activity by running on your Arch machine:
```bash
sudo ls /etc
sudo cat /etc/shadow
```
Wait 1-2 minutes, then search again with the time range set to **Last 15 minutes**:
```
index="wazuh-alerts" agent.name="YOUR-HOSTNAME"
```
<br>

---
> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months
