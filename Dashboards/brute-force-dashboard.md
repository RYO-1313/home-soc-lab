# Brute Force Attack Monitor Dashboard

This dashboard is focused entirely on detecting and analyzing brute force attacks across your monitored endpoints. It gives you everything you need to identify an active attack, track its progress, and determine whether the attacker succeeded.

<br>

---

## Import the Dashboard via XML

If you want to skip the manual steps below, you can import the dashboard directly using the XML file included in this repository: [brute-force-dashboard.xml](brute-force-dashboard.xml)

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

1. Click **Dashboards** in the top menu on Search and Reporting
2. Click **Create New Dashboard**
3. Fill in the following:

- **Title:** Brute Force Attack Monitor
- **Description:** Real-time brute force detection and analysis

4. Choose **Classic Dashboard**
5. Click **Create**

<br>

---

## Step 2 — Total Brute Force Attempts

This panel gives you a single number showing the total brute force attempts in the last 24 hours.

1. Click **Add Panel**
2. Click **New → Single Value**
3. Set the **Content Title** to `Total Brute Force Attempts`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Multiple Windows Logon Failures", "Multiple authentication failures", "Windows audit failure event", "Logon Failure - Unknown user or bad password") | stats count as "Total Brute Force Attempts"
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

---

## Step 3 — Brute Force Timeline

This panel shows attack volume over time grouped by 5 minute intervals.

1. Click **Add Panel**
2. Click **New → Line Chart**
3. Set the **Content Title** to `Brute Force Timeline`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Multiple Windows Logon Failures", "Multiple authentication failures", "Windows audit failure event", "Logon Failure - Unknown user or bad password") | timechart span=5m count as "Attempts"
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

---

## Step 4 — Targeted Machines

This panel shows which machines are being attacked and what share of total attacks each one is receiving.

1. Click **Add Panel**
2. Click **New → Pie Chart**
3. Set the **Content Title** to `Targeted Machines`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Multiple Windows Logon Failures", "Multiple authentication failures", "Windows audit failure event", "Logon Failure - Unknown user or bad password") | stats count by agent.name
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

---

## Step 5 — Successful Logins After Failures

This panel shows successful logins that occurred during the same period as brute force activity.

1. Click **Add Panel**
2. Click **New → Single Value**
3. Set the **Content Title** to `Successful Logins (Possible Compromise)`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Windows Logon Success","Successful sudo to ROOT executed.") | stats count as "Successful Logins"
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

---

## Step 6 — Top Targeted Usernames

This panel shows which usernames are being targeted the most.

1. Click **Add Panel**
2. Click **New → Bar Chart**
3. Set the **Content Title** to `Top Targeted Usernames`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Multiple Windows Logon Failures", "Multiple authentication failures", "Windows audit failure event", "Logon Failure - Unknown user or bad password") | stats count by data.win.eventdata.targetUserName | sort -count | head 10
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

---

## Step 7 — Brute Force by Attack Type

This panel breaks down which type of brute force alerts Wazuh triggered.

1. Click **Add Panel**
2. Click **New → Bar Chart**
3. Set the **Content Title** to `Brute Force by Attack Type`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Multiple Windows Logon Failures", "Multiple authentication failures", "Windows audit failure event", "Logon Failure - Unknown user or bad password") | stats count by rule.description | sort -count
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

---

## Step 8 — Brute Force Events Detail Table

This panel gives you a full table of every brute force event with all the detail you need for investigation and reporting.

1. Click **Add Panel**
2. Click **New → Table**
3. Set the **Content Title** to `Brute Force Events Detail`
4. Paste the following into the search box:

```
index="wazuh-alerts" rule.description IN ("Multiple Windows Logon Failures", "Multiple authentication failures", "Windows audit failure event", "Logon Failure - Unknown user or bad password") | table timestamp agent.name rule.description rule.level full_log | sort -timestamp
```

5. Set the time range to **Last 24 hours**
6. Click **Apply**

<br>

Then click **Save Dashboard**.

<br>

---

## Dashboard Breakdown

Here is what each panel is telling you and how to use it as a SOC analyst.

<br>

### Panel 1 — Total Brute Force Attempts

Your first glance metric. Normal environments sit at zero or very low numbers. Anything above 50 in 24 hours needs investigation. Above 500 means there is a serious active attack in progress.

<br>

### Panel 2 — Brute Force Timeline

Tells you when the attack happened and how long it lasted. A sharp spike means an automated tool is running. A slow steady line means a manual or low-and-slow attack deliberately trying to stay under the radar and avoid detection thresholds.

<br>

### Panel 3 — Targeted Machines

If one machine is receiving 90% of all brute force attempts, that is the primary target and where your investigation should start. If attacks are spread evenly across all machines, the attacker is scanning the entire network rather than going after a specific endpoint.

<br>

### Panel 4 — Successful Logins After Failures

This is your most critical panel. A successful login appearing after hundreds of failures means the attacker got in. Even a single hit here triggers an immediate incident response. Do not ignore this panel.

<br>

### Panel 5 — Top Targeted Usernames

Attackers always try predictable usernames first — Administrator, admin, root. If your actual username appears in this list, it means the attacker did reconnaissance beforehand and already has some intelligence about your environment. That changes the severity of the situation significantly.

<br>

### Panel 6 — Brute Force by Attack Type

Helps you identify which protocol or service is being attacked. Windows audit failure events point to RDP or SMB. Multiple authentication failures point to SSH. Logon Failure with unknown user means the attacker is guessing usernames in addition to passwords.

<br>

### Panel 7 — Brute Force Events Detail Table

Your investigation table. When you need to write an incident report or escalate to a senior analyst, this is where you pull the evidence. It shows you the exact time the attack started, which machine was hit, the severity of each event, and the full raw log for deeper analysis.

<br>

---

## How to use this dashboard during a real incident

Work through the panels in this order:

1. **Panel 1** — Is there an active attack right now?
2. **Panel 2** — When did it start and is it still ongoing?
3. **Panel 3** — Which machine should I focus on?
4. **Panel 6** — What protocol is being attacked?
5. **Panel 5** — Which usernames are being targeted?
6. **Panel 4** — Did the attacker succeed? This is the most important question.
7. **Panel 7** — Gather the evidence for the incident report.

<br>
<img width="1916" height="856" alt="Screenshot From 2026-05-10 06-39-03" src="https://github.com/user-attachments/assets/42725e78-ef1c-436c-b60c-733478b23ebc" />

<img width="1916" height="856" alt="Screenshot From 2026-05-10 06-39-16" src="https://github.com/user-attachments/assets/7c0bed94-a272-4964-ac3e-62e94ccb548c" />

---

> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months
