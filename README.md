# 🛡️ Wazuh Detection Engineering Home Lab

### `SIEM Deployment • Custom Decoders & Rules • MITRE ATT&CK Mapping • Least Privilege`

<p align="center">
  <img src="https://img.shields.io/badge/Wazuh-4.14.7-005792?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Ubuntu-26.04%20LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" />
  <img src="https://img.shields.io/badge/Kali%20Linux-2026.2-000000?style=for-the-badge&logo=kalilinux&logoColor=white" />
  <img src="https://img.shields.io/badge/VirtualBox-7.2-183A61?style=for-the-badge&logo=virtualbox&logoColor=white" />
  <img src="https://img.shields.io/badge/MITRE%20ATT%26CK-T1110-6E2C8B?style=for-the-badge" />
</p>

<p align="center">
  <b>A two-machine Wazuh lab taken end to end: deploy the platform, onboard an endpoint,
  teach it to understand a log format it has never seen, and prove the detection fires.</b>
</p>

---

## ⚡ Overview

Reading about security monitoring and operating it are two different things. Documentation shows the
finished state of a system. It rarely shows the disk filling at the worst possible moment, or a
pattern that looks correct and silently matches nothing.

This project builds a working detection pipeline from raw log line to classified, queryable security
event, breaks it in the ordinary ways a real deployment breaks, and explains each failure.

**Raw log → Agent → Decoder → Rule → Alert → Indexer → Dashboard**

The centrepiece is a custom decoder and rule written for a fictional application log, mapped to a
recognised adversary technique, validated offline before deployment, and confirmed firing against
live activity from a monitored endpoint.

---

## 🎯 What This Lab Demonstrates

| Area | Implementation |
| --- | --- |
| 🖥️ Virtualisation | Two-machine isolated segment on a shared VirtualBox NAT Network |
| 🌐 Network Design | Fixed server address through netplan, automatic endpoint addressing |
| 🛰️ SIEM Deployment | Wazuh manager, indexer and dashboard installed by the official assistant |
| 📡 Endpoint Onboarding | Agent enrolled from the dashboard wizard, path proven on port 1514 |
| 🔎 Detection Validation | Authentication failure and file integrity monitoring tested against live activity |
| 🧩 Detection Engineering | Custom decoder and rule for an unrecognised log format |
| 🎯 Threat Framework | Detection mapped to MITRE ATT&CK technique T1110, Credential Access |
| 🧪 Offline Testing | Full pipeline validated with `wazuh-logtest` before generating traffic |
| 📊 Presentation | Saved search, trend visualisation and monitoring dashboard |
| 🔐 Access Control | Read-only analyst role created and tested against least privilege |
| 🧯 Failure Analysis | Five build failures documented with symptom, cause, fix and lesson |

---

# 🏗️ Lab Architecture

```text
                 Windows host  —  Oracle VirtualBox 7.2
 ┌──────────────────────────────────────────────────────────────────────┐
 │                                                                      │
 │      VirtualBox NAT Network  "NatNetwork"  —  10.0.2.0/24            │
 │                                                                      │
 │   ┌────────────────────────────┐      ┌────────────────────────────┐ │
 │   │   Kali Linux 2026.2        │      │   Ubuntu 26.04 LTS         │ │
 │   │   10.0.2.3   •  4096 MB    │      │   10.0.2.20  •  6144 MB    │ │
 │   │                            │      │                            │ │
 │   │   Wazuh Agent 4.14.7       │─────▶│   Wazuh Manager            │ │
 │   │   Log collection           │ 1514 │   Wazuh Indexer            │ │
 │   │   File integrity monitor   │      │   Wazuh Dashboard          │ │
 │   │   Activity generator       │      │                            │ │
 │   └────────────────────────────┘      └─────────────┬──────────────┘ │
 │                                                     │                │
 └─────────────────────────────────────────────────────┼────────────────┘
                                                       │  HTTPS
                                                   Analyst browser
```

> **Why a NAT Network and not NAT?**
> The two options sit next to each other in the same dropdown and differ by one word. Under plain
> NAT each machine sits behind its own private router and cannot see the other, so an agent can
> never reach a manager. A named NAT Network puts both machines on one segment. This single
> distinction cost the first hours of the build.

---

# 🔬 The Detection Pipeline

```text
        /var/log/myapp.log                       (monitored endpoint)
                 │
                 │   ALERT: User admin failed login from 192.168.1.50
                 ▼
        ┌─────────────────┐
        │   Wazuh Agent   │   localfile block, syslog format
        └────────┬────────┘
                 │  encrypted channel, port 1514
                 ▼
        ┌─────────────────┐
        │     DECODER     │   custom_failed_login
        │                 │   extracts  →  username , srcip
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │      RULE       │   id 100003  •  level 10  •  T1110
        │                 │   decides whether this matters
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │     INDEXER     │   wazuh-alerts-*
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │    DASHBOARD    │   saved search → visualisation → dashboard
        └─────────────────┘
```

**A decoder answers what the log says, field by field. A rule answers whether that matters.**
A decoder without a rule produces structured data nobody looks at. A rule without a working decoder
never fires, because the fields it tests for do not exist.

---

# 🧰 Technology Stack

**Monitoring platform**

* **Wazuh 4.14.7** — manager, indexer and dashboard on a single host
* **OpenSearch** — indexing and search layer beneath the dashboard
* **MITRE ATT&CK** — classification vocabulary for the custom detection

**Endpoint and activity generation**

* **Kali Linux 2026.2** — monitored endpoint
* **Nmap**, **Secure Shell**, shell loops — activity generators

**Infrastructure**

* **Oracle VirtualBox 7.2** — hypervisor
* **Ubuntu 26.04 LTS** — server operating system
* **netplan / NetworkManager** — fixed addressing

---

# 🧪 Lab Modules

## 01 — Building the Virtual Environment

Processor virtualisation extensions confirmed on the host, a named NAT Network created once at the
application level and attached to both machines, and a snapshot taken of each machine before and
after installation. A snapshot of a working state costs a minute and removes the fear of
experimenting with the configuration afterwards.

## 02 — Deploying the Wazuh Server

A fixed address assigned through netplan, then all three central components deployed together with
the official installation assistant.

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Two addressing details are easy to miss: the gateway on a NAT Network is the `.1` address while
plain NAT uses `.2`, and netplan rejects tab characters without saying so plainly. The renderer must
also be named explicitly on the desktop edition, or the configuration is handed to a backend service
that is not running.

Service state is checked in one command rather than three:

```bash
systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager | grep -E "Active:"
```

## 03 — Onboarding the Endpoint

The agent was deployed from the dashboard wizard, which fills in the matching version automatically
and removes the most common cause of agent problems.

```bash
sudo WAZUH_MANAGER='10.0.2.20' WAZUH_AGENT_NAME='kali' dpkg -i ./wazuh-agent_4.14.7-1_amd64.deb
sudo systemctl daemon-reload && sudo systemctl enable --now wazuh-agent
```

**Prove the network path before blaming the application.** If port 1514 answers, the network is
fine and any remaining problem is configuration. If it does not, no amount of agent debugging helps.

```bash
ping -c 4 10.0.2.20
nc -zvn 10.0.2.20 1514
sudo grep -A3 "<server>" /var/ossec/etc/ossec.conf
```

## 04 — Validating Built-In Detection

An installed platform is not a working platform. Two shipped detections were tested against real
activity.

| Test | Activity | Outcome |
| --- | --- | --- |
| Authentication failure | Five Secure Shell logins as an account that does not exist | Seven events, rule `5710`, level 5, mapped to Password Guessing |
| File integrity | File created in a directory switched to real-time monitoring | Rule `554`, reported within seconds |

**A finding worth repeating:** those authentication alerts belong to the *server*, not to the
endpoint that generated the traffic. Failed logins are recorded by the machine being attacked,
because that is where the authentication service writes its log. Filtering the view by the attacking
endpoint returns an empty table.

## 05 — A Test That Produced Nothing

A stealth port scan with service and operating system detection was run from the endpoint against
the server. The scan worked and enumerated open ports. The platform recorded nothing.

```bash
sudo nmap -sS -sV -O 10.0.2.20
```

This is not a misconfiguration. Wazuh is host based — it reads logs that monitored machines write.
It does not inspect network packets. A stealth scan never completes a connection, so no service
writes a log entry, so there is nothing to analyse.

Detecting a scan needs a different class of tool: a network intrusion detection system such as
Suricata or Snort watching the traffic itself, or firewall logs forwarded in so that connection
attempts become log lines. **Knowing what falls outside a platform's field of view is as important
as knowing what falls inside it.**

## 06 — Writing a Custom Detection

The core of the project: teaching the platform to understand a log format it has never seen, and to
make a security decision about it.

**The log format, fixed first and held to at every later step:**

```text
ALERT: User admin failed login from 192.168.1.50
```

**The decoder** — `/var/ossec/etc/decoders/custom_myapp_decoder.xml`

```xml
<decoder name="custom_failed_login">
  <prematch>ALERT: </prematch>
  <regex>ALERT: User (\w+) failed login from (\S+)</regex>
  <order>username,srcip</order>
</decoder>
```

**The rule** — `/var/ossec/etc/rules/custom_myapp_rules.xml`

```xml
<group name="my_app">
  <rule id="100003" level="10">
    <decoded_as>custom_failed_login</decoded_as>
    <description>Custom App: Failed login attempt detected for user
      $(username) from $(srcip)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>
</group>
```

**The log source on the endpoint** — added to `/var/ossec/etc/ossec.conf`

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/myapp.log</location>
</localfile>
```

> ⚠️ **The pattern trap that cost the most time.** Wazuh's default expression engine is a
> deliberately simplified one. Square bracket character classes are **not supported at all**, so a
> pattern like `([0-9.]+)` for an address — valid nearly everywhere else — silently matches
> nothing. There is no error. Writing `(\S+)` avoids the problem entirely.

**Validate offline before generating any traffic:**

```bash
sudo /var/ossec/bin/wazuh-logtest
```

Three phases, each failure pointing somewhere different:

| Phase | Meaning | If it fails |
| --- | --- | --- |
| 1 — Pre-decoding | Raw line accepted | Malformed input |
| 2 — Decoding | Decoder matched, fields extracted | The pattern does not match the line |
| 3 — Rule matching | Rule fired with level and framework mapping | Decoder name mismatch, or manager not restarted |

Three lines appended to the live log produced three alerts within seconds, each carrying the account
name and source address extracted by the decoder and rendered into the description by the rule. The
detection then appeared under **Brute Force** in the MITRE ATT&CK view alongside detections that
shipped with the product.

## 07 — Visualising the Detection

Alerts in a table answer what happened. A dashboard answers whether the pattern is changing, which
is the question an analyst actually asks during a shift.

* **Saved search** — filter `rule.id:100003` stored as a reusable object, so everything built on top
  stays tied to the detection rather than to a query someone has to remember
* **Visualisation** — vertical bar chart, count over a date histogram
* **Dashboard** — trend chart and underlying event table combined in one view

## 08 — Applying Least Privilege

A monitoring platform holds sensitive data and controls the rules that decide what counts as an
attack. An analyst who only needs to read alerts should not be able to rewrite detections.

| Permission type | Granted |
| --- | --- |
| Cluster | `cluster_composite_ops_ro` — read operations only, nothing that writes |
| Index | Read on `wazuh-alerts-*` only |
| Tenant | Read-only on the global tenant |

The role was then **tested**, not assumed. Logging in as the analyst in a private browsing window
produced an instructive result: the account could not load the administrative console at all,
because the console reads configuration objects the role deliberately does not grant. Alert data
remained reachable through the search interface.

That illustrates a real tension in access control design — a role tightened purely against data
access can end up too narrow for the vendor interface to function. The production answer is to grant
read access to the interface configuration index as well, landing in the middle ground where an
analyst can work comfortably but still cannot change a rule.

---

# 📊 Validation Results

| # | Test | Expected | Result |
| --- | --- | --- | --- |
| 1 | Agent enrollment | Endpoint active on the manager | ✅ Agent `001`, active, version 4.14.7 |
| 2 | Failed authentication | Alerts raised and classified | ✅ 7 events, rule 5710, level 5 |
| 3 | Brute force correlation | Rule 5712 fires | ⚠️ Did not fire — five manual attempts fall below the frequency threshold |
| 4 | File integrity monitoring | Addition reported in real time | ✅ Rule 554 within seconds |
| 5 | Stealth port scan | Scan alerts | ❌ None — host-based platform cannot see packet-level activity |
| 6 | Offline detection test | Three phases pass | ✅ Rule 100003, level 10, T1110 |
| 7 | Live custom detection | Alerts from real log activity | ✅ 3 alerts with extracted fields |
| 8 | Framework mapping | Detection visible under Brute Force | ✅ |
| 9 | Least privilege role | Administrative access refused | ✅ Console blocked, alert data readable |

The two results that are not a tick mark are the most useful ones in the set. Both are explained in
full in the report, and neither is a misconfiguration.

---

# 🧯 Problems Encountered and How They Were Resolved

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | Both machines online, no traffic between them | Adapters set to NAT instead of NAT Network | Created a named NAT Network and attached both machines |
| 2 | Interface up but holding no address | netplan handed the configuration to a backend that was not running | Declared `renderer: NetworkManager` explicitly |
| 3 | Desktop would not load, indexer refused to start | Root filesystem full — 17 GB of vulnerability database inside the manager queue | Cleared the database, disabled the module, switched to a console-only boot |
| 4 | Decoder accepted but matching nothing | Square bracket character class unsupported by the default engine — fails silently | Replaced with an escaped class the engine supports |
| 5 | Confirmed alerts returning no search results | Query entered from the endpoint page, pinning it to the wrong agent | Removed the pinned filter and widened the time range |

A second effect of problem 3 is worth recording: once a disk crosses the flood-stage watermark the
indexer protects itself by marking indices read-only, and it does **not** lift that block
automatically when space is returned. It has to be cleared explicitly.

---

# 🧠 Key Takeaways

```text
A SIEM IS A PIPELINE, NOT A PRODUCT
        │
        ├── Collection, decoding, detection, storage and presentation are separate stages
        ├── A fault in any one of them looks identical from the dashboard: nothing appears
        │
        ├── Decoders and rules answer different questions, and both are required
        ├── Validate offline before you generate live traffic
        ├── Know where the tool cannot see — no log entry means no alert, ever
        ├── Alerts belong to the machine that logged them, not the one that caused them
        ├── Least privilege is a claim until an account has been logged in and tested
        └── Most of the learning was in the failures, not the installation
```

---

# 🚧 Scope and Next Steps

**Deliberately out of scope**

* Network-level detection — no packet inspection anywhere in this build
* Vulnerability detection — the module was disabled to recover disk space, so that capability is
  not demonstrated here
* Distributed deployment — all three central components run on one host, which is correct for a lab
  and wrong for production

**Clearest next extensions**

* Feed Suricata or firewall logs into the platform to close the gap that the port scan exposed
* Configure active response so a detection can block rather than only report
* Allocate sixty gigabytes of storage and run the server without a graphical session from the start

---

# 📂 Repository Structure

```text
wazuh-detection-engineering-lab/
│
├── README.md
├── LICENSE
│
├── docs/
│   └── Detection-Engineering-Home-Lab.pdf     # full 30-page report with figures
│
└── detections/
    ├── custom_myapp_decoder.xml               # parses the custom log format
    └── custom_myapp_rules.xml                 # classifies it, mapped to T1110
```

---

# 📚 Full Documentation

The complete report in **`docs/`** contains the full build with thirty labelled figures: every
configuration value used, every command in the order it was run, the reasoning behind each fix
rather than only the final answer, and a command reference appendix so the build can be repeated
without rereading the document.

---

# 👨‍💻 Author

### Muhammad Usman

**Cybersecurity Student | Security Operations | Blue Team | Detection Engineering**

---

<p align="center">

### 🛡️ Collect → Decode → Detect → Validate → Visualise → Restrict

**⭐ Star this repository if the failure analysis saved you an afternoon.**

</p>
