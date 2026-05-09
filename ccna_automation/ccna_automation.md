# 🚀 CCNA Automation (200-901) — 30-Day Study Plan
> **Goal:** Pass the 200-901 exam and become a network automation practitioner.  
> **Daily commitment:** ~2–3 hours | **Format:** Theory + Hands-on every day  
> **GitHub:** Used as your Second Brain — notes, code, and labs all live here.

---

## 🗂️ GitHub Second Brain Setup (Do This on Day 1 — First 30 Minutes)

### Repo Structure
Create a public or private repo called `ccna-automation-notes`:

```
ccna-automation-notes/
├── README.md                    ← Master index + progress tracker
├── 01-software-dev-design/
│   ├── notes.md
│   ├── data-formats/
│   │   ├── xml-example.xml
│   │   ├── json-example.json
│   │   └── yaml-example.yaml
│   └── git-cheatsheet.md
├── 02-apis/
│   ├── notes.md
│   ├── http-codes.md
│   └── scripts/
│       └── rest_api_example.py
├── 03-cisco-platforms/
│   ├── notes.md
│   └── scripts/
├── 04-app-deployment-security/
│   ├── notes.md
│   └── dockerfiles/
├── 05-infrastructure-automation/
│   ├── notes.md
│   ├── ansible-playbooks/
│   └── yang-models/
├── 06-network-fundamentals/
│   └── notes.md
├── labs/
│   └── README.md                ← Lab index with outcomes
└── cheatsheets/
    ├── http-codes.md
    ├── port-numbers.md
    ├── git-commands.md
    └── python-snippets.md
```

### README.md Progress Tracker Template
```markdown
# CCNA Automation Study Progress

| Domain | Weight | Status | Notes |
|--------|--------|--------|-------|
| 1.0 Software Dev & Design | 15% | 🔲 | |
| 2.0 APIs | 20% | 🔲 | |
| 3.0 Cisco Platforms | 15% | 🔲 | |
| 4.0 App Deployment & Security | 15% | 🔲 | |
| 5.0 Infrastructure & Automation | 20% | 🔲 | |
| 6.0 Network Fundamentals | 15% | 🔲 | |

Legend: 🔲 Not started | 🟡 In progress | ✅ Done
```

### Daily Git Workflow (Non-Negotiable Habit)
```bash
# End of every study session:
git add .
git commit -m "Day X: <what you learned/built>"
git push
```
This builds a visible streak and forces you to name what you learned.

---

## 📅 The 30-Day Plan

---

### WEEK 1 — Foundations (Days 1–7)
**Theme:** Get the scaffolding right. Network fundamentals + Python + Git + Data Formats.

---

#### Day 1 — Setup + Network Fundamentals Foundations
**Topics:** 6.1, 6.2, 6.3 | Git setup

| Time | Activity |
|------|----------|
| 0:00–0:30 | Set up GitHub repo (Second Brain structure above) |
| 0:30–1:15 | Study MAC addresses, VLANs, IP addresses, subnets, gateways (6.1, 6.2) |
| 1:15–2:00 | Study switching, routing, firewalls, load balancers (6.3) |
| 2:00–2:30 | **Hands-on:** Draw a small network topology on paper OR use draw.io |

**GitHub commit:** `notes/06-network-fundamentals/notes.md` — summary of OSI-relevant concepts, subnet table, VLAN use cases.

---

#### Day 2 — Network Planes + IP Services + Topology Reading
**Topics:** 6.4, 6.5, 6.6, 6.7

| Time | Activity |
|------|----------|
| 0:00–0:45 | Study management/data/control planes (6.5); IP services: DHCP, DNS, NAT, SNMP, NTP (6.6) |
| 0:45–1:30 | Common port numbers: SSH(22), Telnet(23), HTTP(80), HTTPS(443), NETCONF(830) (6.7) |
| 1:30–2:15 | **Hands-on:** Interpret 2–3 network topology diagrams (find on Cisco DevNet or Google) |
| 2:15–2:30 | Build `cheatsheets/port-numbers.md` in your repo |

**GitHub commit:** Port numbers cheatsheet + topology diagram notes.

---

#### Day 3 — Connectivity Troubleshooting + Data Formats (XML, JSON, YAML)
**Topics:** 6.8, 6.9, 1.1

| Time | Activity |
|------|----------|
| 0:00–0:30 | NAT problems, blocked ports, proxy, VPN impact on apps (6.8, 6.9) |
| 0:30–1:30 | Compare XML vs JSON vs YAML: syntax, use cases, pros/cons (1.1) |
| 1:30–2:30 | **Hands-on:** Write the same config data in all three formats. Save examples to your repo. |

**Sample task:** Represent a router config (hostname, interfaces, IP addresses) in XML, JSON, and YAML.

**GitHub commit:** `01-software-dev-design/data-formats/` with all three files.

---

#### Day 4 — Parsing Data Formats with Python
**Topics:** 1.2

| Time | Activity |
|------|----------|
| 0:00–0:30 | Read about Python's `xml.etree`, `json`, and `pyyaml` libraries |
| 0:30–2:00 | **Hands-on:** Write Python scripts to parse each format from Day 3 |
| 2:00–2:30 | Document the parsing patterns in `notes.md` |

**Code to write:**
```python
# json_parse.py
import json
with open("config.json") as f:
    data = json.load(f)
print(data["hostname"])

# yaml_parse.py
import yaml
with open("config.yaml") as f:
    data = yaml.safe_load(f)

# xml_parse.py
import xml.etree.ElementTree as ET
tree = ET.parse("config.xml")
root = tree.getroot()
```

**GitHub commit:** Three parsing scripts in `01-software-dev-design/data-formats/`.

---

#### Day 5 — Git Deep Dive
**Topics:** 1.7, 1.8 (a–g)

| Time | Activity |
|------|----------|
| 0:00–1:00 | Study version control concepts + all Git operations (clone, add, commit, push/pull, branch, merge, diff) |
| 1:00–2:30 | **Hands-on Lab:** Practice every Git operation on your study repo |

**Git lab sequence:**
```bash
git clone <your-repo>
git checkout -b feature/day5-git-lab
# make changes
git add .
git commit -m "Day 5: Git lab practice"
git diff main feature/day5-git-lab
git checkout main
git merge feature/day5-git-lab
# Introduce a conflict intentionally and resolve it
git log --oneline --graph
```

**GitHub commit:** `cheatsheets/git-commands.md` with every command and what it does.

---

#### Day 6 — Software Dev Methods + Design Patterns + TDD
**Topics:** 1.3, 1.4, 1.5, 1.6

| Time | Activity |
|------|----------|
| 0:00–0:45 | Agile vs Lean vs Waterfall — compare with a table (1.4) |
| 0:45–1:15 | Functions/classes/modules — why structure code this way (1.5) |
| 1:15–1:45 | MVC and Observer design patterns (1.6) |
| 1:45–2:30 | **Hands-on:** Write a tiny Python module with a class, then import it from another script |

**GitHub commit:** Python module example + comparison table for dev methods.

---

#### Day 7 — Week 1 Review + Second Brain Update
| Time | Activity |
|------|----------|
| 0:00–1:00 | Review all Week 1 notes. Fill in any gaps. |
| 1:00–1:45 | Take 20-question practice quiz (use Cisco DevNet Learning Labs or Anki) |
| 1:45–2:30 | **Update your GitHub README** — mark Week 1 topics done, write 3-bullet summary per domain |

**Reflection prompt in your notes:** *"What from this week do I need to revisit? What clicked immediately?"*

---

### WEEK 2 — APIs (Days 8–14)
**Theme:** The heaviest exam weight (20%). Master REST, HTTP, Python requests.

---

#### Day 8 — HTTP Fundamentals + REST Concepts
**Topics:** 2.4, 2.6, 2.8

| Time | Activity |
|------|----------|
| 0:00–0:45 | HTTP response codes: 2xx, 3xx, 4xx, 5xx — know them cold (2.4) |
| 0:45–1:30 | HTTP response anatomy: status line, headers, body (2.6) |
| 1:30–2:30 | **Hands-on:** Use `curl` or browser DevTools to inspect real HTTP responses |

```bash
curl -I https://httpbin.org/get         # headers only
curl -v https://httpbin.org/status/404  # verbose — see the full exchange
curl https://httpbin.org/get | python3 -m json.tool
```

**GitHub commit:** `cheatsheets/http-codes.md` — table of codes with meanings and common API causes.

---

#### Day 9 — REST API Requests + Authentication
**Topics:** 2.1, 2.7

| Time | Activity |
|------|----------|
| 0:00–0:45 | REST principles: stateless, resource-based URLs, HTTP verbs (GET/POST/PUT/DELETE/PATCH) |
| 0:45–1:30 | Auth mechanisms: Basic auth, API keys, Bearer tokens (2.7) |
| 1:30–2:30 | **Hands-on:** Make authenticated requests to a public API |

**Lab:** Use `https://api.github.com` (free, needs only a token):
```bash
# Get your own profile
curl -H "Authorization: token YOUR_TOKEN" https://api.github.com/user

# Create a repo via API (POST)
curl -X POST -H "Authorization: token YOUR_TOKEN" \
  -d '{"name":"api-test","private":false}' \
  https://api.github.com/user/repos
```

**GitHub commit:** Document the auth patterns with code examples.

---

#### Day 10 — Python Requests Library
**Topics:** 2.9

| Time | Activity |
|------|----------|
| 0:00–0:30 | Read the `requests` library docs overview |
| 0:30–2:30 | **Hands-on:** Build a Python script that does GET, POST, handles auth, and parses response |

**Code to build:**
```python
import requests

BASE_URL = "https://httpbin.org"

# GET with query params
resp = requests.get(f"{BASE_URL}/get", params={"name": "cisco"})
print(resp.status_code)
print(resp.json())

# POST with JSON body
resp = requests.post(f"{BASE_URL}/post",
    json={"device": "router1", "ip": "10.0.0.1"},
    headers={"Authorization": "Bearer mytoken123"})

# Error handling
resp.raise_for_status()  # raises on 4xx/5xx
```

**GitHub commit:** `02-apis/scripts/rest_client.py` — reusable REST client template.

---

#### Day 11 — API Troubleshooting + Webhooks + Constraints
**Topics:** 2.2, 2.3, 2.5

| Time | Activity |
|------|----------|
| 0:00–0:45 | Webhooks: push vs pull, event-driven, use cases (2.2) |
| 0:45–1:15 | API constraints: rate limiting, pagination, timeouts, versioning (2.3) |
| 1:15–2:30 | **Hands-on:** Build a script that handles rate limiting with retry logic |

```python
import requests, time

def api_call_with_retry(url, retries=3):
    for i in range(retries):
        resp = requests.get(url)
        if resp.status_code == 429:
            wait = int(resp.headers.get("Retry-After", 2))
            time.sleep(wait)
            continue
        resp.raise_for_status()
        return resp.json()
    raise Exception("Max retries exceeded")
```

**GitHub commit:** Retry pattern + webhook diagram in notes.

---

#### Day 12 — API Styles + Cisco DevNet Sandbox Intro
**Topics:** 2.8, 3.7

| Time | Activity |
|------|----------|
| 0:00–0:45 | REST vs RPC vs Synchronous vs Asynchronous APIs — comparison table (2.8) |
| 0:45–2:30 | **Hands-on:** Sign up at https://devnetsandbox.cisco.com — reserve the "Always-On" Cisco Catalyst Center or Meraki sandbox |

**DevNet Resources to bookmark:**
- Sandbox: `https://devnetsandbox.cisco.com`
- Code Exchange: `https://developer.cisco.com/codeexchange/`
- Learning Labs: `https://developer.cisco.com/learning/`
- API Docs: `https://developer.cisco.com/docs/`

**GitHub commit:** Resource index in `03-cisco-platforms/devnet-resources.md`.

---

#### Day 13 — Practice API Calls Against Real Cisco APIs
**Topics:** 2.1, 2.5 (applied)

| Time | Activity |
|------|----------|
| All session | **Hands-on Lab:** Use Cisco DevNet Always-On Catalyst Center sandbox |

**Lab tasks:**
```python
# Authenticate to Cisco Catalyst Center
import requests, base64

DNAC = "https://sandboxdnac.cisco.com"  # DevNet always-on
creds = base64.b64encode(b"devnetuser:Cisco123!").decode()

auth = requests.post(f"{DNAC}/dna/system/api/v1/auth/token",
    headers={"Authorization": f"Basic {creds}"})
token = auth.json()["Token"]

# Get network devices
devices = requests.get(f"{DNAC}/dna/intent/api/v1/network-device",
    headers={"X-Auth-Token": token})
print(devices.json())
```

**GitHub commit:** Working Catalyst Center script in `03-cisco-platforms/scripts/`.

---

#### Day 14 — Week 2 Review + API Cheatsheet
| Time | Activity |
|------|----------|
| 0:00–1:00 | Review all API notes. Write 1-paragraph summary per topic. |
| 1:00–2:00 | Build the ultimate API cheatsheet in your repo |
| 2:00–2:30 | 20-question practice quiz focused on HTTP/REST |

**GitHub commit:** `cheatsheets/api-patterns.md` — the document you'll open on exam day morning.

---

### WEEK 3 — Cisco Platforms + App Deployment & Security (Days 15–21)

---

#### Day 15 — Cisco Network Platforms Overview
**Topics:** 3.2

| Time | Activity |
|------|----------|
| 0:00–1:30 | Study each platform: Meraki, Catalyst Center (DNAC), ACI, SD-WAN, NSO — capabilities, use cases, API style |
| 1:30–2:30 | **Hands-on:** Browse each platform's API docs and note 3 key endpoints per platform |

**Platform Summary Table to build in notes:**

| Platform | What it manages | API type | Key endpoint example |
|----------|----------------|----------|---------------------|
| Meraki | Cloud-managed networks | REST | `/organizations/{orgId}/devices` |
| Catalyst Center | Campus networks | REST | `/dna/intent/api/v1/network-device` |
| ACI | Data center fabric | REST | `/api/node/class/fvTenant.json` |
| SD-WAN | WAN automation | REST | `/dataservice/device` |
| NSO | Multi-vendor orchestration | RESTCONF/NETCONF | `/restconf/data/devices` |

---

#### Day 16 — Cisco Compute + Collaboration + Security Platforms
**Topics:** 3.3, 3.4, 3.5

| Time | Activity |
|------|----------|
| 0:00–0:45 | UCS Manager + Intersight (3.3) |
| 0:45–1:30 | Webex, CUCM/AXL/UDS APIs (3.4) |
| 1:30–2:30 | **Hands-on:** Send a Webex message via API |

```python
import requests

WEBEX_TOKEN = "YOUR_BOT_TOKEN"  # Create free bot at developer.webex.com
ROOM_ID = "YOUR_ROOM_ID"

requests.post("https://webexapis.com/v1/messages",
    headers={"Authorization": f"Bearer {WEBEX_TOKEN}"},
    json={"roomId": ROOM_ID, "text": "Hello from my automation script! 🤖"})
```

**GitHub commit:** Webex messaging script in `03-cisco-platforms/scripts/`.

---

#### Day 17 — Model-Driven Programmability (YANG, RESTCONF, NETCONF)
**Topics:** 3.6, 3.8

| Time | Activity |
|------|----------|
| 0:00–0:45 | YANG data modeling basics — modules, leaves, containers, lists |
| 0:45–1:30 | NETCONF (XML over SSH, port 830) vs RESTCONF (HTTP-based) |
| 1:30–2:30 | **Hands-on:** Make a RESTCONF call to IOS XE DevNet sandbox |

```python
import requests

DEVICE = "https://sandbox-iosxe-latest-1.cisco.com"
# Credentials from DevNet sandbox page

resp = requests.get(
    f"{DEVICE}/restconf/data/ietf-interfaces:interfaces",
    auth=("developer", "C1sco12345"),
    headers={"Accept": "application/yang-data+json"}
)
print(resp.json())
```

**GitHub commit:** RESTCONF example + YANG model notes with annotated examples.

---

#### Day 18 — Cloud Deployment Models + CI/CD Concepts
**Topics:** 4.1, 4.2, 4.3, 4.4, 4.12

| Time | Activity |
|------|----------|
| 0:00–0:45 | Private/public/hybrid cloud + edge computing (4.1, 4.2) |
| 0:45–1:15 | VMs vs bare metal vs containers (4.3) |
| 1:15–2:00 | CI/CD pipeline components: source control → build → test → deploy (4.4) |
| 2:00–2:30 | DevOps principles: collaboration, automation, feedback loops (4.12) |

**GitHub commit:** Diagram/table comparing deployment models in `04-app-deployment-security/notes.md`.

---

#### Day 19 — Docker Hands-On
**Topics:** 4.6, 4.7

| Time | Activity |
|------|----------|
| 0:00–0:30 | Understand Dockerfile structure: FROM, RUN, COPY, EXPOSE, CMD |
| 0:30–2:30 | **Hands-on:** Build and run a Docker container with a Python script inside |

**Lab:**
```dockerfile
# Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY rest_client.py .
CMD ["python", "rest_client.py"]
```
```bash
docker build -t my-automation-app .
docker run my-automation-app
docker ps
docker logs <container_id>
docker exec -it <container_id> /bin/bash
```

**GitHub commit:** Dockerfile + Docker cheatsheet in `04-app-deployment-security/dockerfiles/`.

---

#### Day 20 — Security + Python Unit Testing + Bash
**Topics:** 4.5, 4.8, 4.9, 4.10, 4.11

| Time | Activity |
|------|----------|
| 0:00–0:30 | OWASP top threats: XSS, SQLi, CSRF — definitions + prevention (4.10) |
| 0:30–1:00 | Secret protection, encryption at rest/transit, env vars (4.8) |
| 1:00–1:30 | Firewall, DNS, load balancers, reverse proxies in deployment (4.9) |
| 1:30–2:30 | **Hands-on:** Write Python unit tests + Bash file management scripts |

```python
# test_parser.py
import unittest
from my_parser import parse_device_config

class TestParser(unittest.TestCase):
    def test_hostname_extracted(self):
        data = '{"hostname": "Router1", "ip": "10.0.0.1"}'
        result = parse_device_config(data)
        self.assertEqual(result["hostname"], "Router1")

if __name__ == "__main__":
    unittest.main()
```

```bash
#!/bin/bash
# Bash lab: file management
mkdir -p /tmp/lab/backups
for f in /tmp/lab/*.cfg; do cp "$f" /tmp/lab/backups/; done
export BACKUP_PATH="/tmp/lab/backups"
echo "Backed up to $BACKUP_PATH"
ls -la $BACKUP_PATH
```

**GitHub commit:** Unit test file + bash script in their respective folders.

---

#### Day 21 — Week 3 Review + Platform Summary Cards
| Time | Activity |
|------|----------|
| 0:00–1:30 | Review Weeks 1–3. Focus on weak spots. |
| 1:30–2:30 | Create one-page "platform cards" for each Cisco platform in your notes |

**GitHub commit:** Platform summary cards + update README progress tracker.

---

### WEEK 4 — Infrastructure Automation + Final Review (Days 22–30)

---

#### Day 22 — Ansible Fundamentals
**Topics:** 5.6, 5.8

| Time | Activity |
|------|----------|
| 0:00–0:45 | Ansible concepts: inventory, playbooks, modules, tasks, roles |
| 0:45–2:30 | **Hands-on:** Install Ansible, write and run a basic playbook |

```yaml
# site.yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
```bash
ansible-playbook -i inventory.ini site.yaml --check  # dry run
ansible-playbook -i inventory.ini site.yaml
```

**GitHub commit:** Ansible playbook in `05-infrastructure-automation/ansible-playbooks/`.

---

#### Day 23 — Terraform + Infrastructure as Code
**Topics:** 5.5, 5.6

| Time | Activity |
|------|----------|
| 0:00–0:45 | IaC principles: declarative, idempotent, version-controlled infrastructure |
| 0:45–2:30 | **Hands-on:** Install Terraform, write a basic config (local file provider or AWS free tier) |

```hcl
# main.tf - local file example (no cloud needed)
terraform {
  required_providers {
    local = { source = "hashicorp/local" }
  }
}

resource "local_file" "device_config" {
  filename = "./device.cfg"
  content  = "hostname Router1\ninterface GigabitEthernet0/0\n ip address 10.0.0.1 255.255.255.0"
}
```
```bash
terraform init
terraform plan
terraform apply
terraform destroy
```

**GitHub commit:** Terraform example in `05-infrastructure-automation/`.

---

#### Day 24 — YANG Models + RESTCONF/NETCONF Results
**Topics:** 5.1, 5.10, 5.11

| Time | Activity |
|------|----------|
| 0:00–1:00 | Read and interpret YANG model syntax (module, container, leaf, list, types) |
| 1:00–2:30 | **Hands-on:** Query IOS XE via RESTCONF, interpret the JSON/XML output |

**YANG Reading Guide:**
```yang
module ietf-interfaces {
  container interfaces {           // group of things
    list interface {               // repeatable item
      key "name";
      leaf name { type string; }   // single value
      leaf enabled { type boolean; }
      container statistics {
        leaf in-octets { type uint64; }
      }
    }
  }
}
```

**GitHub commit:** Annotated YANG model + RESTCONF query results in notes.

---

#### Day 25 — CI/CD for Infrastructure + Code Review + Sequence Diagrams
**Topics:** 5.4, 5.13, 5.14

| Time | Activity |
|------|----------|
| 0:00–0:45 | CI/CD for infrastructure: lint → test → deploy stages, pyATS integration (5.4) |
| 0:45–1:15 | Code review principles: what to look for, why it matters (5.13) |
| 1:15–2:30 | **Hands-on:** Interpret 2–3 sequence diagrams showing API flows |

**Sequence diagram reading checklist:**
- Who is the initiator? (client, user, system)
- What HTTP verb + endpoint is called?
- What does each response contain?
- Where are auth tokens passed?
- What happens on error?

**GitHub commit:** Sequence diagram interpretation examples in notes.

---

#### Day 26 — Network Simulation + pyATS + Unified Diff
**Topics:** 5.3, 5.12, 1.8.g

| Time | Activity |
|------|----------|
| 0:00–0:45 | Cisco Modeling Labs (CML) and pyATS roles in network testing (5.3) |
| 0:45–1:15 | Reading unified diffs: + lines (added), - lines (removed), @@ context markers (5.12) |
| 1:15–2:30 | **Hands-on:** Use `git diff` on your own repo, read the output carefully |

```diff
@@ -10,7 +10,8 @@
 def get_devices(token):
-    url = "http://old-api/devices"
+    url = "https://new-api/v2/devices"
+    # Updated to use HTTPS and v2 endpoint
     resp = requests.get(url, headers={"X-Token": token})
```

**GitHub commit:** Diff interpretation guide in `cheatsheets/`.

---

#### Day 27 — Python Script Interpretation + Ansible Playbook Reading
**Topics:** 5.7, 5.8, 5.9

| Time | Activity |
|------|----------|
| 0:00–1:30 | Practice reading Python scripts that use ACI, Meraki, RESTCONF — identify the workflow |
| 1:30–2:30 | Practice reading Ansible playbooks and bash scripts — identify every action |

**Exam skill:** Given a script, answer: *What is it doing? In what order? What API does it call?*

Practice source: `https://github.com/CiscoDevNet/` — read real examples there.

**GitHub commit:** Your own annotated versions of 2–3 scripts you found and understood.

---

#### Day 28 — Full Domain Review Sprint
| Time | Activity |
|------|----------|
| All session | Speed-run all 6 domains. One page of notes per domain from memory, then compare to your repo. |

**Focus on highest-weight domains:**
- Domain 2 (APIs) — 20%
- Domain 5 (Infrastructure) — 20%

---

#### Day 29 — Timed Practice Exam + Gap Analysis
| Time | Activity |
|------|----------|
| 0:00–1:30 | Take a full 60–80 question practice test (use Boson, MeasureUp, or Cisco U) |
| 1:30–2:30 | For every wrong answer: find the topic, go to your notes, write a one-line correction |

**GitHub commit:** `EXAM_READINESS.md` — honest assessment of strong vs weak areas.

---

#### Day 30 — Light Review + Second Brain Final Update
| Time | Activity |
|------|----------|
| 0:00–1:30 | Review your cheatsheets only — no new material |
| 1:30–2:00 | Update README: mark all topics ✅ |
| 2:00–2:30 | Rest. You're ready. |

**Final GitHub commit:** `"Day 30: Ready to take the exam 🚀"`

---

## 📌 Key Resources

| Resource | URL |
|----------|-----|
| Cisco DevNet Sandbox | https://devnetsandbox.cisco.com |
| DevNet Learning Labs | https://developer.cisco.com/learning/ |
| Cisco API Docs | https://developer.cisco.com/docs/ |
| httpbin (API testing) | https://httpbin.org |
| Cisco Code Exchange | https://developer.cisco.com/codeexchange/ |
| pyATS docs | https://developer.cisco.com/docs/pyats/ |
| YANG Catalog | https://yangcatalog.org |
| Webex Developer | https://developer.webex.com |

---

## 🧠 Second Brain Rules (Commit to These)

1. **Write before you copy.** Summarize in your own words first, then add code.
2. **Every concept gets an example.** No naked definitions in your notes.
3. **Commit every day.** Even if it's just a single line. The streak is the habit.
4. **One cheatsheet per domain.** The cheatsheet is what you open the morning of the exam.
5. **Link your scripts to the topic.** Each script in your repo should have a comment at the top: `# Covers exam topic: 2.9 — Python REST API with requests library`

---

*Good luck. The second brain you build during these 30 days will serve you long after the exam.*
