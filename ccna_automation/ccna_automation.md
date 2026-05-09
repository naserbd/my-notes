# 🚀 CCNA Automation (200-901) — 14-Day Study Plan

> **Goal:** Become an automation practitioner, not just pass the exam.
> **Daily commitment:** ~3–4 hours (1.5h theory + 1.5–2h hands-on + 30min notes to GitHub)
> **Structure:** Theory → Lab → Document to GitHub

---

## 🗂️ GitHub Second Brain Setup (Do This on Day 0)

Before starting, create a GitHub repo called `ccna-auto-notes`. Use this structure:

```
ccna-auto-notes/
├── README.md                  # Your master index
├── 01-software-dev-design/
│   ├── data-formats.md
│   ├── git-cheatsheet.md
│   └── design-patterns.md
├── 02-apis/
│   ├── rest-api-notes.md
│   ├── http-codes.md
│   └── scripts/
├── 03-cisco-platforms/
│   ├── meraki.md
│   ├── catalyst-center.md
│   └── scripts/
├── 04-app-deployment-security/
│   ├── docker-notes.md
│   ├── cicd.md
│   └── owasp.md
├── 05-infrastructure-automation/
│   ├── ansible-notes.md
│   ├── terraform-notes.md
│   ├── yang-restconf-netconf.md
│   └── playbooks/
├── 06-network-fundamentals/
│   └── networking-review.md
└── daily-logs/
    ├── day-01.md
    └── ...
```

### Daily GitHub Ritual (30 min each evening)
1. Open today's `daily-logs/day-XX.md`
2. Write: *What I learned, commands I used, things that confused me, 3 key takeaways*
3. Copy any scripts/configs into the relevant folder
4. `git add . && git commit -m "Day XX: <topic>" && git push`

### Note Template (save as `_template.md`)
```markdown
# Topic Name
**Date:** YYYY-MM-DD | **Exam Section:** X.X

## What It Is (1-sentence definition)

## How It Works (bullet points)

## Key Commands / Code
```bash
# paste your actual commands here
```

## Real-World Analogy

## Gotchas & Exam Tips

## Resources Used
```

---

## 📅 Day-by-Day Study Plan

---

### DAY 1 — Git & Version Control + Environment Setup
**Exam coverage:** 1.7, 1.8 (15% domain)

#### Theory (1.5h)
- Why version control matters for automation engineers
- Git workflow: working tree → staging → commit → remote
- Branching strategies (feature branches, main/dev split)
- Merge vs. rebase, handling conflicts

#### Hands-On (2h)
```bash
# Install Git, Python 3, VS Code, Docker (get these ready today)
git init my-ccna-lab
cd my-ccna-lab
git remote add origin https://github.com/YOUR_USERNAME/ccna-auto-notes
git checkout -b feature/day1-test
echo "# Test" > test.md
git add test.md
git commit -m "feat: initial commit"
git push origin feature/day1-test
git checkout main
git merge feature/day1-test
# Intentionally create a merge conflict and resolve it
git diff HEAD~1 HEAD
```

#### GitHub Note to Write
- `01-software-dev-design/git-cheatsheet.md` — every command you used today with a one-line explanation

---

### DAY 2 — Data Formats: XML, JSON, YAML + Python Parsing
**Exam coverage:** 1.1, 1.2

#### Theory (1.5h)
- JSON: key-value, arrays, nesting — used by REST APIs
- XML: tags, attributes, namespaces — used by NETCONF
- YAML: indentation-based — used by Ansible playbooks
- When each format is used in network automation

#### Hands-On (2h)
```python
# parse_formats.py — save this to 01-software-dev-design/scripts/
import json, yaml
from xml.etree import ElementTree as ET

# JSON
device = '{"hostname": "R1", "ip": "10.0.0.1", "interfaces": ["Gi0/0", "Gi0/1"]}'
data = json.loads(device)
print(data["interfaces"][0])  # Gi0/0

# YAML (pip install pyyaml)
yaml_data = """
hostname: R1
interfaces:
  - name: Gi0/0
    ip: 10.0.0.1
"""
parsed = yaml.safe_load(yaml_data)
print(parsed["interfaces"][0]["ip"])

# XML
xml_data = """<device><hostname>R1</hostname><ip>10.0.0.1</ip></device>"""
root = ET.fromstring(xml_data)
print(root.find("hostname").text)
```

#### GitHub Note to Write
- `01-software-dev-design/data-formats.md` — comparison table of JSON vs XML vs YAML with examples

---

### DAY 3 — Software Dev Methods + Design Patterns + TDD
**Exam coverage:** 1.3, 1.4, 1.5, 1.6

#### Theory (1.5h)
- **Agile:** iterative sprints, working software over documentation
- **Lean:** eliminate waste, continuous improvement
- **Waterfall:** sequential phases, rigid requirements upfront
- **TDD:** write test first → code fails → make it pass → refactor
- **MVC:** Model (data), View (UI), Controller (logic) — separation of concerns
- **Observer pattern:** event-driven, publisher/subscriber — like webhooks

#### Hands-On (2h)
```python
# tdd_example.py — write the TEST before the function
import unittest

class TestNetworkValidator(unittest.TestCase):
    def test_valid_ip(self):
        self.assertTrue(is_valid_ip("192.168.1.1"))
    def test_invalid_ip(self):
        self.assertFalse(is_valid_ip("999.999.999.999"))
    def test_private_range(self):
        self.assertTrue(is_private_ip("10.0.0.1"))

# NOW write the function to make tests pass
import ipaddress
def is_valid_ip(ip):
    try:
        ipaddress.ip_address(ip)
        return True
    except ValueError:
        return False

def is_private_ip(ip):
    return ipaddress.ip_address(ip).is_private

if __name__ == "__main__":
    unittest.main()
```

#### GitHub Note to Write
- `01-software-dev-design/design-patterns.md` — MVC diagram, Observer example, TDD cycle

---

### DAY 4 — REST APIs Deep Dive
**Exam coverage:** 2.1, 2.3, 2.4, 2.5, 2.6, 2.8

#### Theory (1.5h)
- REST constraints: stateless, client-server, uniform interface, cacheable
- HTTP methods: GET, POST, PUT, PATCH, DELETE
- HTTP response codes to memorise:
  - 200 OK, 201 Created, 204 No Content
  - 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
  - 429 Too Many Requests (rate limiting), 500 Internal Server Error
- REST vs RPC, synchronous vs asynchronous APIs
- HTTP response anatomy: status line, headers (Content-Type, Auth), body

#### Hands-On (2h)
```python
# Use a free public API — no setup needed
import requests

# GET request
response = requests.get("https://jsonplaceholder.typicode.com/users/1")
print(f"Status: {response.status_code}")
print(f"Headers: {response.headers['Content-Type']}")
print(f"Body: {response.json()['name']}")

# POST request
new_device = {"name": "Router1", "ip": "10.0.0.1"}
r = requests.post(
    "https://jsonplaceholder.typicode.com/posts",
    json=new_device,
    headers={"Content-Type": "application/json"}
)
print(f"Created: {r.status_code} — {r.json()}")

# Error handling pattern you'll use everywhere
def api_call(url, token):
    try:
        r = requests.get(url, headers={"Authorization": f"Bearer {token}"}, timeout=10)
        r.raise_for_status()
        return r.json()
    except requests.exceptions.HTTPError as e:
        print(f"HTTP Error {r.status_code}: {e}")
    except requests.exceptions.ConnectionError:
        print("Cannot reach server")
```

#### GitHub Note to Write
- `02-apis/rest-api-notes.md` — HTTP methods table, status codes cheat sheet, your error-handling template

---

### DAY 5 — API Authentication + Webhooks + Python Requests Library
**Exam coverage:** 2.2, 2.7, 2.9

#### Theory (1h)
- **Basic Auth:** Base64(username:password) in Authorization header — avoid in production
- **API Keys:** passed in header or query param (`?api_key=xxx`)
- **Token Auth (Bearer):** login → get token → use token in requests
- **Webhooks:** server calls YOU when event happens (reverse API) — used in Webex, Meraki alerts

#### Hands-On (2h)
```python
# auth_patterns.py
import requests
import base64

BASE_URL = "https://sandboxdnac.cisco.com"  # Cisco DevNet Always-On Sandbox

# 1. Basic Auth → get token
def get_token():
    credentials = base64.b64encode(b"devnetuser:Cisco123!").decode()
    r = requests.post(
        f"{BASE_URL}/dna/system/api/v1/auth/token",
        headers={"Authorization": f"Basic {credentials}",
                 "Content-Type": "application/json"}
    )
    return r.json()["Token"]

# 2. Use token for subsequent calls
token = get_token()
r = requests.get(
    f"{BASE_URL}/dna/intent/api/v1/network-device",
    headers={"X-Auth-Token": token}
)
devices = r.json()["response"]
for d in devices[:3]:
    print(f"{d['hostname']} — {d['managementIpAddress']}")
```

> **DevNet Sandbox:** Sign up free at developer.cisco.com/sandbox — always-on sandboxes need no reservation

#### GitHub Note to Write
- `02-apis/http-codes.md` + add `02-apis/scripts/auth_patterns.py`

---

### DAY 6 — Cisco Catalyst Center + Meraki APIs
**Exam coverage:** 3.2, 3.9.a, 3.9.c

#### Theory (1.5h)
- **Cisco Catalyst Center:** network controller for campus/branch, intent-based networking
  - APIs: `/dna/intent/api/v1/` — devices, topology, issues, site management
- **Meraki:** cloud-managed networking, Dashboard API
  - API key in `X-Cisco-Meraki-API-Key` header
  - `/api/v1/organizations` → `/networks` → `/devices`

#### Hands-On (2h)
```python
# meraki_devices.py — use Meraki DevNet Sandbox
import requests

MERAKI_KEY = "6bec40cf957de430a6f1f2baa056b99a4fac9ea"  # DevNet demo key
BASE = "https://api.meraki.com/api/v1"

headers = {"X-Cisco-Meraki-API-Key": MERAKI_KEY, "Content-Type": "application/json"}

# Get organizations
orgs = requests.get(f"{BASE}/organizations", headers=headers).json()
org_id = orgs[0]["id"]
print(f"Org: {orgs[0]['name']}")

# Get networks
networks = requests.get(f"{BASE}/organizations/{org_id}/networks", headers=headers).json()
net_id = networks[0]["id"]

# Get devices
devices = requests.get(f"{BASE}/networks/{net_id}/devices", headers=headers).json()
for d in devices:
    print(f"{d.get('name','?')} — {d.get('model')} — {d.get('lanIp','N/A')}")

# Get clients (exam topic 3.9.c)
clients = requests.get(f"{BASE}/networks/{net_id}/clients",
                       params={"timespan": 86400}, headers=headers).json()
print(f"Clients seen in last 24h: {len(clients)}")
```

#### GitHub Note to Write
- `03-cisco-platforms/meraki.md` and `catalyst-center.md` with API endpoint patterns

---

### DAY 7 — Review Day + Webex API + ACI Overview
**Exam coverage:** 3.4, 3.9.b, 3.2 (ACI)

#### Morning: Review (1h)
- Re-read your GitHub notes from Days 1–6
- Quiz yourself on HTTP status codes and Git commands
- Fix any gaps or errors in your notes

#### Theory (1h)
- **Webex API:** manage rooms (spaces), members, messages
  - Bearer token auth, base URL: `https://webexapis.com/v1/`
- **ACI:** application-centric infrastructure, policy-based SDN
  - REST API, JSON/XML, tenant → app profile → EPG hierarchy

#### Hands-On (2h)
```python
# webex_bot.py — get free token at developer.webex.com
import requests

TOKEN = "YOUR_WEBEX_TOKEN"
BASE = "https://webexapis.com/v1"
headers = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

# List rooms (spaces) — exam 3.9.b
rooms = requests.get(f"{BASE}/rooms", headers=headers).json()
for r in rooms["items"][:3]:
    print(f"Room: {r['title']} — ID: {r['id']}")

# Send a message
room_id = rooms["items"][0]["id"]
msg = requests.post(f"{BASE}/messages",
    headers=headers,
    json={"roomId": room_id, "text": "Hello from my automation script!"}
)
print(f"Message sent: {msg.status_code}")

# List participants
members = requests.get(f"{BASE}/memberships",
                       params={"roomId": room_id}, headers=headers).json()
for m in members["items"]:
    print(f"  Member: {m.get('personDisplayName')}")
```

#### GitHub Note to Write
- `03-cisco-platforms/webex-api.md` with endpoint reference

---

### DAY 8 — Docker, Containers, and Application Deployment
**Exam coverage:** 4.1–4.7, 4.12

#### Theory (1.5h)
- **VMs vs Containers vs Bare Metal** — resource isolation levels
- **Docker:** image (template) → container (running instance)
- **Dockerfile:** FROM, RUN, COPY, EXPOSE, CMD/ENTRYPOINT
- **Cloud models:** private, public, hybrid, edge
- **DevOps principles:** culture of collaboration, automation, fast feedback
- **Edge computing:** process data near source, reduce latency

#### Hands-On (2h)
```dockerfile
# Dockerfile — containerize your Meraki script
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY meraki_devices.py .
ENV MERAKI_API_KEY=""
CMD ["python", "meraki_devices.py"]
```

```bash
# Build and run
docker build -t meraki-checker .
docker run -e MERAKI_API_KEY=your_key_here meraki-checker

# Useful Docker commands to know
docker images          # list images
docker ps              # running containers
docker ps -a           # all containers
docker logs <id>       # container output
docker exec -it <id> bash  # shell into container
docker stop <id>
docker rm <id>
docker rmi <image>
```

#### GitHub Note to Write
- `04-app-deployment-security/docker-notes.md` — Dockerfile anatomy, commands cheat sheet

---

### DAY 9 — Security: OWASP, Secrets, Encryption + CI/CD
**Exam coverage:** 4.4, 4.8, 4.9, 4.10

#### Theory (1.5h)
- **OWASP Top threats for automation:**
  - XSS (Cross-Site Scripting): inject malicious scripts into web output
  - SQL Injection: malicious SQL in user input fields
  - CSRF: trick user's browser into making unwanted requests
- **Secret protection:** never hardcode API keys — use env vars or vaults
- **Encryption:** TLS in transport, AES/RSA at rest
- **CI/CD Pipeline stages:** Source → Build → Test → Deploy
- **Firewall/LB/Proxy in deployments:** DMZ, health checks, SSL termination

#### Hands-On (2h)
```python
# SECURE script — secrets from environment variables
import os, requests, unittest

# WRONG — never do this:  API_KEY = "abc123secretkey"

# RIGHT — read from environment
API_KEY = os.environ.get("MERAKI_API_KEY")
if not API_KEY:
    raise EnvironmentError("MERAKI_API_KEY not set. Export it first.")

class TestSecretHandling(unittest.TestCase):
    def test_no_hardcoded_key(self):
        key = os.environ.get("MERAKI_API_KEY", "")
        self.assertNotEqual(key, "", "API key must be set in environment")

    def test_https_only(self):
        url = "https://api.meraki.com/api/v1/organizations"
        self.assertTrue(url.startswith("https://"), "Always use HTTPS")

if __name__ == "__main__":
    unittest.main()
```

```bash
# .env approach (NEVER commit .env to git!)
echo "MERAKI_API_KEY=yourkey" > .env
echo ".env" >> .gitignore      # critical!
export $(cat .env | xargs)     # load into shell
```

#### GitHub Note to Write
- `04-app-deployment-security/owasp.md` + `cicd.md` with pipeline diagram

---

### DAY 10 — YANG, NETCONF, and RESTCONF
**Exam coverage:** 3.8, 5.1, 5.10, 5.11

#### Theory (1.5h)
- **YANG:** data modelling language — defines the structure of config/state data
  - `container`, `list`, `leaf`, `leaf-list` are core constructs
- **NETCONF:** XML-based protocol over SSH (port 830) to configure devices
  - Operations: `<get>`, `<get-config>`, `<edit-config>`, `<commit>`
  - Datastores: `running`, `candidate`, `startup`
- **RESTCONF:** REST interface for YANG models (HTTP/HTTPS)
  - GET/PUT/POST/DELETE on YANG paths
  - Media type: `application/yang-data+json`

#### Hands-On (2h)
```python
# restconf_query.py — Cisco DevNet Always-On IOS XE sandbox
import requests
requests.packages.urllib3.disable_warnings()

DEVICE = "sandbox-iosxe-latest-1.cisco.com"
AUTH = ("developer", "lastorangerestoreball8876")
BASE = f"https://{DEVICE}/restconf/data"
HEADERS = {
    "Accept": "application/yang-data+json",
    "Content-Type": "application/yang-data+json"
}

# Get all interfaces
r = requests.get(f"{BASE}/ietf-interfaces:interfaces",
                 auth=AUTH, headers=HEADERS, verify=False)
interfaces = r.json()["ietf-interfaces:interfaces"]["interface"]
for iface in interfaces[:5]:
    print(f"{iface['name']} — enabled: {iface.get('enabled', 'N/A')}")

# Get specific interface
r2 = requests.get(
    f"{BASE}/ietf-interfaces:interfaces/interface=GigabitEthernet1",
    auth=AUTH, headers=HEADERS, verify=False)
print(r2.json())
```

```yang
# Basic YANG model to interpret (exam 5.11)
module cisco-device {
  namespace "http://cisco.com/device";
  prefix dev;

  container interfaces {
    list interface {
      key "name";
      leaf name { type string; }
      leaf description { type string; }
      leaf enabled { type boolean; default true; }
      leaf ip-address { type string; }
    }
  }
}
```

#### GitHub Note to Write
- `05-infrastructure-automation/yang-restconf-netconf.md` with protocol comparison table

---

### DAY 11 — Ansible for Network Automation
**Exam coverage:** 5.6, 5.8

#### Theory (1.5h)
- **Ansible concepts:** agentless, SSH/API based, idempotent
- **Inventory:** list of managed hosts
- **Playbook structure:** play → tasks → modules
- **Key modules:** `ios_command`, `ios_config`, `uri`, `debug`, `copy`, `service`
- **Variables:** `vars`, `host_vars`, `group_vars`, `register`

#### Hands-On (2h)
```yaml
# network_playbook.yml — interpret this for the exam
---
- name: Configure and verify network devices
  hosts: routers
  gather_facts: no
  vars:
    ntp_server: "10.0.0.1"

  tasks:
    - name: Set NTP server
      cisco.ios.ios_config:
        lines:
          - "ntp server {{ ntp_server }}"

    - name: Get interface status
      cisco.ios.ios_command:
        commands: "show ip interface brief"
      register: int_output

    - name: Display interfaces
      debug:
        var: int_output.stdout_lines

    - name: Save config
      cisco.ios.ios_config:
        save_when: always
```

```yaml
# inventory.yml
all:
  children:
    routers:
      hosts:
        R1:
          ansible_host: 10.0.0.1
          ansible_user: admin
          ansible_network_os: ios
```

```bash
# Ansible commands
ansible-playbook -i inventory.yml network_playbook.yml
ansible-playbook -i inventory.yml network_playbook.yml --check  # dry run
ansible-playbook -i inventory.yml network_playbook.yml -v       # verbose
```

#### GitHub Note to Write
- `05-infrastructure-automation/ansible-notes.md` — playbook anatomy + module reference

---

### DAY 12 — Terraform + NSO + Infrastructure as Code
**Exam coverage:** 5.5, 5.6, 5.2, 5.3, 5.4

#### Theory (1.5h)
- **Infrastructure as Code (IaC):** define infra in version-controlled files
- **Terraform:** declarative, provider-based, `plan` → `apply` → `destroy`
  - HCL syntax, state file, providers (AWS, Meraki, ACI)
- **NSO:** multi-vendor service orchestration, YANG models, NEDs
- **Controller-level vs device-level management**
- **Cisco Modeling Labs (CML) + pyATS:** network simulation and testing
- **CI/CD for infrastructure:** commit → lint → test → deploy

#### Hands-On (2h)
```hcl
# main.tf — Terraform HCL to interpret for exam
terraform {
  required_providers {
    meraki = {
      source  = "cisco-open/meraki"
      version = "~> 0.1"
    }
  }
}

provider "meraki" {
  meraki_api_key = var.meraki_api_key
}

variable "meraki_api_key" {
  description = "Meraki Dashboard API key"
  type        = string
  sensitive   = true        # won't show in logs
}

resource "meraki_networks_switch_port" "access_port" {
  network_id  = "L_123456"
  port_id     = "5"
  name        = "Employee-Desk-5"
  vlan        = 100
  type        = "access"
  enabled     = true
}

output "port_name" {
  value = meraki_networks_switch_port.access_port.name
}
```

```bash
# Terraform workflow
terraform init      # download providers
terraform plan      # show what will change
terraform apply     # make changes
terraform destroy   # tear down
terraform show      # current state
```

#### GitHub Note to Write
- `05-infrastructure-automation/terraform-notes.md` — IaC principles, Terraform workflow, NSO vs Ansible comparison

---

### DAY 13 — Network Fundamentals + Bash Scripting
**Exam coverage:** 5.9, 6.0 (all), 4.11

#### Theory (1.5h)
- **Networking review:** MAC/VLANs, IP/subnets, routing
- **Management/Control/Data planes:** how network devices process traffic
- **IP Services:** DHCP, DNS, NAT, SNMP, NTP — purpose and port numbers
- **Key ports:** SSH(22), Telnet(23), HTTP(80), HTTPS(443), NETCONF(830), SNMP(161)
- **Connectivity troubleshooting:** NAT, blocked ports, proxy bypass, VPN split-tunnel

#### Hands-On (2h)
```bash
#!/bin/bash
# network_check.sh — bash automation script (exam 5.9, 4.11)

DEVICES=("10.0.0.1" "10.0.0.2" "10.0.0.3")
LOG_DIR="/var/log/network-checks"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="$LOG_DIR/check_$TIMESTAMP.log"

# File management + directory navigation
mkdir -p "$LOG_DIR"
echo "Network check started at $TIMESTAMP" | tee "$LOG_FILE"

# Loop and check devices
for DEVICE in "${DEVICES[@]}"; do
    if ping -c 2 -W 2 "$DEVICE" &>/dev/null; then
        echo "[OK]   $DEVICE is reachable" | tee -a "$LOG_FILE"
    else
        echo "[FAIL] $DEVICE is UNREACHABLE" | tee -a "$LOG_FILE"
    fi
done

# Environmental variables
echo "Running as: $USER"
echo "Python path: $(which python3)"
echo "Log saved to: $LOG_FILE"

# Cleanup logs older than 7 days
find "$LOG_DIR" -name "check_*.log" -mtime +7 -delete
echo "Old logs cleaned up."
```

```bash
# Other Bash skills to practice
ls -la /etc/network/              # directory navigation
cp config.yml config.yml.bak      # file management
grep "error" /var/log/syslog      # search logs
export API_KEY="mykey"            # set env variable
echo $PATH                        # read env variable
chmod +x script.sh && ./script.sh # make executable and run
```

#### GitHub Note to Write
- `06-network-fundamentals/networking-review.md` — ports table, plane definitions, troubleshooting checklist

---

### DAY 14 — Full Review + Mock Exam + GitHub Polish
**Exam coverage:** All domains

#### Morning: Weak Area Blitz (2h)
Go through your GitHub notes. Find any topics with sparse notes — that's a weak area. Focus on whichever of these you're least confident about:
- Unified diff interpretation (5.12)
- Sequence diagrams with API calls (5.14)
- Code review principles (5.13)
- Cisco security platforms overview (3.5) — XDR, Firepower, ISE, Secure Endpoint
- OWASP threats recap (4.10)
- ACI hierarchy: Tenant → App Profile → EPG → Contract

#### Afternoon: GitHub Second Brain Polish (1.5h)
```bash
# Polish your repo before the exam
cd ccna-auto-notes
# Update README.md with a full topic index
# Create QUICK-REFERENCE.md as your personal cheat sheet:
#   - HTTP status codes you always forget
#   - Git commands in order
#   - Docker commands
#   - Port numbers table
#   - REST vs NETCONF vs RESTCONF comparison
git add . && git commit -m "Day 14: Final review + polish" && git push
```

#### Evening: Simulate Exam Conditions (1h)
- Take a timed practice test (Boson, Pearson, or DevNet practice questions)
- Review wrong answers against your GitHub notes
- For any gaps, add a note to the relevant file immediately

---

## 📊 Domain Coverage Summary

| Domain | % Weight | Days Covered |
|--------|----------|-------------|
| 1.0 Software Development & Design | 15% | 1, 2, 3 |
| 2.0 Understanding & Using APIs | 20% | 4, 5 |
| 3.0 Cisco Platforms & Development | 15% | 6, 7 |
| 4.0 Application Deployment & Security | 15% | 8, 9 |
| 5.0 Infrastructure & Automation | 20% | 10, 11, 12 |
| 6.0 Network Fundamentals | 15% | 13 |
| Review & Integration | — | 7, 14 |

---

## 🔑 DevNet Sandboxes (Free — Use These for Labs)

| Resource | URL | Used On |
|----------|-----|---------|
| Catalyst Center Always-On | `sandboxdnac.cisco.com` | Day 5, 6 |
| IOS XE Always-On | `sandbox-iosxe-latest-1.cisco.com` | Day 10 |
| Meraki Dashboard | `api.meraki.com` (demo key) | Day 6 |
| Webex Developer | `developer.webex.com` | Day 7 |
| DevNet Sandbox Portal | `developer.cisco.com/sandbox` | All |

---

## 💡 GitHub Second Brain — Pro Tips

1. **Commit daily, no exceptions** — even just a daily log entry counts
2. **Use GitHub Issues** as a study TODO list — open an issue for each weak topic, close it when solid
3. **Use GitHub Actions** to auto-lint your Python scripts on every push — free CI/CD practice
4. **Pin your repo** on your GitHub profile — shows employers you're an active practitioner
5. **Add a `QUICK-REFERENCE.md`** — writing it IS the studying

```yaml
# .github/workflows/lint.yml — free CI/CD for your notes repo
name: Lint Python Scripts
on: [push]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with: {python-version: '3.11'}
      - run: pip install flake8
      - run: flake8 --max-line-length=100 **/*.py
```

---

*Good luck! The goal is to become an automation practitioner — the exam is just proof of the work.*
