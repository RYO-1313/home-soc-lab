# Splunk Dashboards

This guide walks you through building a Wazuh security dashboard inside Splunk. By the end you will have seven panels giving you a real-time view of everything happening across your monitored endpoints.

<br>

---

## Import the Dashboard via XML

If you want to skip the manual steps below, you can import the dashboard directly using the XML file included in this repository.
- [ ] [Splunk Default Dashboards](configuration/wazuh-default-dashboard.xml)

# How to
1. Go to **Dashboards** in Splunk
2. Click **Create New Dashboard**
3. Give it a title
4. Click **Classic Dashboard**
5. Click **Create**
6. Once inside, click **Edit → Source**
7. Delete the existing XML
8. Paste the XML from the file
9. Click **Save**

> **Before importing, make sure you have** Wazuh alerts flowing into a `wazuh-alerts` index. Otherwise the panels will show **No results** since there is no data.

<br>

---

## Step 1 — Create a new Dashboard

1. Click **Dashboards** in the top menu
2. Click **Create New Dashboard** at the top left
3. Fill in the following:

- **Title:** Wazuh Security Overview
- **Description:** Real-time security alerts from Wazuh EDR

4. Choose **Classic Dashboard**
5. Click **Create**

<br>

---

## Step 2 — Total Alerts Count

This panel gives you a single number showing how many alerts fired in the last 24 hours.

1. Click **Add Panel**
2. Click **New → Single Value**
3. Set the **Content Title** to `Total Alerts`
4. Paste the following into the search box:

```
index="wazuh-alerts" | stats count as "Total Alerts"
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Step 3 — Alerts by Severity Level

This panel breaks down how many alerts fired at each severity level.

1. Click **Add Panel**
2. Click **New → Bar Chart**
3. Set the **Content Title** to `Alerts by Severity Level`
4. Paste the following into the search box:

```
index="wazuh-alerts" | stats count by rule.level | sort -rule.level
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Step 4 — Top 10 Alert Types

This panel shows the most frequently triggered alert rules — useful for spotting patterns and recurring issues.

1. Click **Add Panel**
2. Click **New → Bar Chart**
3. Set the **Content Title** to `Top 10 Alert Types`
4. Paste the following into the search box:

```
index="wazuh-alerts" | stats count by rule.description | sort -count | head 10
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Step 5 — Alerts by Agent

This panel shows which machine is generating the most alerts.

1. Click **Add Panel**
2. Click **New → Pie Chart**
3. Set the **Content Title** to `Alerts by Agent`
4. Paste the following into the search box:

```
index="wazuh-alerts" | stats count by agent.name
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Step 6 — Alerts Timeline

This panel shows alert volume over time, grouped by hour.

1. Click **Add Panel**
2. Click **New → Line Chart**
3. Set the **Content Title** to `Alerts Timeline`
4. Paste the following into the search box:

```
index="wazuh-alerts" | timechart span=1h count as "Alerts per Hour"
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Step 7 — MITRE ATT&CK Techniques

This panel shows which MITRE ATT&CK techniques were detected across your endpoints.

1. Click **Add Panel**
2. Click **New → Bar Chart**
3. Set the **Content Title** to `Top MITRE ATT&CK Techniques`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.mitre.technique!="" | stats count by rule.mitre.technique | sort -count | head 10
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Step 8 — High Severity Alerts

This panel is a detailed table of every alert at level 7 or above — your primary action list as an analyst.

1. Click **Add Panel**
2. Click **New → Statistics Table**
3. Set the **Content Title** to `High Severity Alerts`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.level>=7 | table timestamp agent.name rule.level rule.description full_log | sort -rule.level
```

5. Set the time range to **Last 24 hours**
6. Click **Add to Dashboard**

<br>

---

## Dashboard Breakdown

Here is what each panel is actually telling you and why it matters.

<br>

### Panel 1 — Total Alerts

Gives you a quick pulse of your environment. A sudden spike means something unusual is happening — it could be an attack, a misconfiguration, or a new agent generating noise.

<br>

### Panel 2 — Alerts by Severity Level

Wazuh classifies every alert with a severity level between 0 and 15. Levels 0 to 3 are informational, 4 to 6 are low threat, 7 to 10 are medium, and 11 to 15 are critical. As a SOC analyst your focus always starts at level 7 and above.

<br>

### Panel 3 — Top 10 Alert Types

Shows the most frequently triggered rules. If "Failed SSH login" appears 500 times that is a brute force attack. If "Sudo to root" is at the top someone is escalating privileges repeatedly. Patterns here tell you what kind of activity to investigate.

<br>

### Panel 4 — Alerts by Agent

Shows which machine is generating the most alerts. If your Windows machine suddenly accounts for 80% of all alerts, that endpoint is likely compromised or under active attack. This panel tells you where to look first.

<br>

### Panel 5 — Alerts Timeline

Attacks have patterns. Brute force attacks show a sudden spike. Malware beaconing shows alerts at regular intervals. A healthy system has a flat, predictable baseline. Any deviation from that baseline is worth investigating.

<br>

### Panel 6 — MITRE ATT&CK Techniques

MITRE ATT&CK is a globally recognized framework that classifies attacker behavior. This panel does not just tell you that something happened — it tells you what the attacker is trying to do. T1078 means stolen credentials are being used. T1548 means privilege escalation. T1110 is brute force. T1055 is process injection. These labels map directly to real attack playbooks.

<br>

### Panel 7 — High Severity Alerts Table

This is your action list. Every row is an alert that needs to be looked at. It shows the timestamp, the affected machine, the severity level, a description of what triggered it, and the raw log. If you only have time to check one panel, make it this one.

<br>

<img width="1919" height="895" alt="Screenshot From 2026-05-07 11-49-07" src="https://github.com/user-attachments/assets/fbda6637-2472-4718-b70e-4b9776ab70cb" />
<img width="1919" height="895" alt="Screenshot From 2026-05-07 11-49-17" src="https://github.com/user-attachments/assets/dc1bba76-ee38-4e30-b4dc-5adc028969c2" />

---

> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months

> **Note:** This is a default starter dashboard covering the most essential panels. More customized dashboards focused on specific use cases — such as brute force detection, privilege escalation tracking, and Windows event monitoring — will be added in the future.
