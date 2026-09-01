# LearningSteps Origins — Azure 2-Tier Deployment

## Overview

LearningSteps is a FastAPI application backed by a PostgreSQL database that lets users track their daily learning journey through CRUD journal entries. The goal of this project was to take that application off `localhost` and deploy it as a secure, production-style 2-tier architecture on Microsoft Azure — one internet-facing tier for the application, and one strictly private tier for the database.

Rather than putting everything on a single server, the architecture separates concerns at the network level: the FastAPI server sits in a public subnet where it can accept traffic from the outside world, while PostgreSQL sits in a private subnet with no route in from the internet at all. The only way data reaches the database is through the application server itself, over a controlled internal path. This mirrors how real production systems are typically laid out — the public-facing layer is treated as expendable and replaceable, while the data layer is treated as the asset that needs the most protection.

This document walks through what was built, the reasoning behind the security configuration, the problems encountered along the way, and what the process revealed about how Azure networking actually behaves versus how it looks on paper.

- **Codebase used:** `reference` branch (Reference Implementation)
- **Cloud provider:** Microsoft Azure
- **Region:** Germany West Central

---

## 1. Architecture Diagram

![VNet creation](screenshots/00-Diagram.png)

---

## 2. Resources Created

| Resource | Name | Notes |
|---|---|---|
| Resource Group | `rg-learningsteps` | Region: Germany West Central |
| Virtual Network | `vnet-learningsteps` | Address space `192.168.0.0/16` |
| Public Subnet | `subnet-web-public` | `192.168.1.0/24` |
| Private Subnet | `subnet-db-private` | `192.168.0.0/24` |
| Web NSG | `nsg-web` | Attached to `subnet-web-public` |
| DB NSG | `nsg-db` | Attached to `subnet-db-private` |
| Web VM | `vm-web` | Ubuntu 24.04 LTS, Public IP `20.170.34.70` |
| DB VM | `vm-db` | Ubuntu 24.04 LTS, Private IP `192.168.0.4`, no public IP |

![Resource group created](screenshots/01-rg-created.png)
![Public subnet](screenshots/02-subnet-web-public.png)
![Private subnet](screenshots/03-subnet-db-private.png)
![VNet creation](screenshots/04-vnet-learningsteps-creation.png)
![NSGs created](screenshots/05-NSGs-created.png)
![VM web create](screenshots/10-vm-web-create.png)
![VM db create](screenshots/11-vm-db-create.png)

---

## 3. Network Security Groups

### nsg-web (attached to `subnet-web-public`)

| Priority | Name | Source | Port | Action |
|---|---|---|---|---|
| 100 | Allow-HTTP | Any | 80 | Allow |
| 110 | Allow-SSH-MyIP | Home IP /32 | 22 | Allow |
| 120 | Allow-App-8000 | Any | 8000 | Allow |
| 65500 | Implicit Deny | Any | Any | Deny |

![nsg-web rules](screenshots/06-nsg-web-rules.png)
![Allow port 8000](screenshots/24-allow-port-8000.png)
![NSG web attached to subnet](screenshots/08-nsg-web-subnet.png)

### nsg-db (attached to `subnet-db-private`)

| Priority | Name | Source | Port | Action |
|---|---|---|---|---|
| 100 | Allow-Postgres-FromWeb | `192.168.1.0/24` | 5432 | Allow |
| 110 | Allow-SSH-FromWeb | `192.168.1.0/24` | 22 | Allow |
| 65500 | Implicit Deny | Any | Any | Deny |

![nsg-db rules](screenshots/07-nsg-db-rules.png)
![NSG db attached to subnet](screenshots/09-nsg-db-subnet.png)

---

## 4. Database Server Setup (`vm-db`)

Connected via SSH hop through `vm-web` (since `vm-db` has no public IP):

```bash
ssh azureuser@20.170.34.70
ssh azureuser@192.168.0.4
```

![SSH to vm-db](screenshots/12-ssh-vm-db.png)

Installed PostgreSQL (temporary public IP was attached during install only, then removed — see Challenges below):

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

![Temporary public IP](screenshots/13a-templP.png)
![PostgreSQL active](screenshots/13b-postgresql-active.png)

Created the database and application user:

```sql
CREATE DATABASE learning_journal;
CREATE USER user123 WITH ENCRYPTED PASSWORD 'password123';
GRANT ALL PRIVILEGES ON DATABASE learning_journal TO user123;
```

![Database created](screenshots/14-create-db.png)

Created the `entries` table:

```sql
CREATE TABLE IF NOT EXISTS entries (
    id VARCHAR PRIMARY KEY,
    data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_entries_created_at ON entries(created_at);
CREATE INDEX IF NOT EXISTS idx_entries_data_gin ON entries USING GIN (data);

GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO user123;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO user123;
```

![Entries table output](screenshots/15-entries-output.png)

Configured PostgreSQL for remote connections from the web subnet only:

`/etc/postgresql/16/main/postgresql.conf`
```
listen_addresses = '*'
```

![listen_addresses config](screenshots/16-listen_addresses.png)

`/etc/postgresql/16/main/pg_hba.conf`
```
host    all    all    192.168.1.0/24    md5
```

![pg_hba.conf edit](screenshots/17-edit-pg_hba.png)

Restarted and verified:

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

![PostgreSQL status](screenshots/18-postgresql-status.png)

---

## 5. Application Server Setup (`vm-web`)

Verified network connectivity to the database before deploying the app:

```bash
sudo apt install telnet -y
telnet 192.168.0.4 5432
```

![Telnet connect success](screenshots/19-telnet-connect.png)

Installed dependencies and cloned the reference implementation:

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv git -y
git clone -b reference https://github.com/CyberstepsDE/learningsteps.git
cd learningsteps
```

![Cloned repository](screenshots/20-cloned-git.png)

Configured environment variables:

```bash
mv .env-sample .env
nano .env
```

```
POSTGRES_USER=user123
POSTGRES_PASSWORD=password123
POSTGRES_DB=learning_journal
DATABASE_URL=postgresql://user123:password123@192.168.0.4:5432/learning_journal
```

![.env file](screenshots/21-env-file.png)

Started the application:

```bash
chmod +x start.sh
sudo ./start.sh
```

![FastAPI startup log](screenshots/22-FastAPI-start.png)

The app started on port `8000` — an inbound NSG rule was added for this port (see Challenges).

![Connection denied before NSG fix](screenshots/23-deny-web.png)

---

## 6. Testing & Validation

Accessed the interactive Swagger UI:

```
http://20.170.34.70:8000/docs
```

![Swagger UI](screenshots/25-Swagger-browser-app.png)

Verified all CRUD operations:

- `POST /entries` — created a new journal entry
- `GET /entries` — listed all entries
- `PATCH /entries/{entry_id}` — updated an entry
- `DELETE /entries/{entry_id}` — deleted an entry

![POST response](screenshots/26-POST-response.png)
![GET response](screenshots/27-GET-response.png)
![PATCH response](screenshots/28-PATCH-response.png)

Confirmed the database is **not reachable from the public internet**:

- `vm-db` has no public IP address at all
- From a local machine, `Test-NetConnection` to the web server's public IP on port 5432 fails

```powershell
Test-NetConnection -ComputerName 20.170.34.70 -Port 5432
```

![vm-db has no public IP](screenshots/29-vm-db-no-public-ip.png)
![Test-NetConnection fails](screenshots/30-Test-NetConnection-fail.png)

---

## 7. Success Criteria — Status

| Criteria | Status |
|---|---|
| API accessible via public IP | ✅ `http://20.170.34.70:8000/docs` |
| Database strictly internal | ✅ No public IP, NSG restricts to web subnet only |
| All CRUD operations work | ✅ POST, GET, PATCH, DELETE tested via Swagger UI |
| Data persists after reboot | ✅ Stored in PostgreSQL on a persistent VM disk |
| NSGs tightly scoped (least privilege) | ✅ Web NSG limited to 80/8000/22(home IP); DB NSG limited to 5432/22 from web subnet only |

---

## 8. Security Decisions

### Why two subnets instead of one

The single most important decision in this architecture was splitting the network into `subnet-web-public` (192.168.1.0/24) and `subnet-db-private` (192.168.0.0/24) instead of running both the API and the database on one VM or one subnet. A single flat network would mean that anything exposed to the internet is only one misconfigured rule away from also exposing the database. By physically separating the tiers, the database's security doesn't depend on the web server being configured correctly — it has its own network boundary, its own NSG, and its own set of rules that don't care what happens on the web side. This is defense in depth: even if `nsg-web` were accidentally opened to `Any` on every port tomorrow, `vm-db` would still be unreachable from the internet because it isn't on a subnet that has one.

### Why `vm-db` has no public IP at all

The most effective control in this whole setup isn't actually a firewall rule — it's the absence of a public IP address on `vm-db`. An NSG rule can be misconfigured, disabled, or overridden by a higher-priority rule. A VM with no public IP simply has no interface for the internet to reach, full stop, regardless of what any NSG says. This was chosen as the primary line of defense, with NSG rules acting as a second, independent layer on top of it.

### Why `nsg-db` explicitly allows only the web subnet, not "VNet default"

Azure VMs on the same VNet can already reach each other by default, which technically makes the explicit `Allow-Postgres-FromWeb` rule (port 5432, source `192.168.1.0/24`) redundant on day one. It was added anyway, deliberately, for two reasons:

1. **Auditability** — anyone reviewing the NSG later can see at a glance exactly which subnet is allowed to talk to the database on which port, without needing to know or trust the VNet's default behavior.
2. **Future-proofing** — if another subnet or VM were added to this VNet later (e.g. a monitoring server, a bastion host, a second application tier), that new resource would **not** automatically get access to Postgres, because the rule is scoped to the web subnet specifically, not "anything inside the VNet." Least privilege should hold even as the network grows, not just on day one.

### Why SSH access is scoped the way it is

SSH into `vm-db` is only possible by first connecting to `vm-web` and hopping across the private network — `vm-db`'s NSG only allows port 22 from `192.168.1.0/24`. This means the database server has no direct SSH exposure to the outside world under any circumstance; an administrator has to already be inside the trusted network boundary (via the web server) to reach it. `vm-web`, being the one server that has to be internet-facing, has its SSH rule scoped even further — restricted to a single home IP address (`/32`) rather than `Any` — since it's the one point in the whole architecture where an open port faces the public internet, and it's the highest-value target for brute-force or credential-stuffing attempts.

### Why no explicit "deny all" rule was added

Azure NSGs already include an implicit deny-all rule at priority 65500 that blocks any traffic not explicitly allowed. Adding a redundant manual deny rule wouldn't change the actual security posture, but it would clutter the rule set. Keeping the NSGs to a short, explicit allow-list (and trusting the platform's default deny) makes it easier to read the security posture of each NSG at a glance — every rule listed is a deliberate exception, not noise.

---

## 9. Challenges

### The private VM couldn't reach the internet to install its own software

The first real obstacle came almost immediately after SSHing into `vm-db`: running `sudo apt install postgresql` failed with repeated connection timeouts to Ubuntu's package mirrors. At first this looked like a broken package repository, but it was actually the private subnet doing exactly what it was designed to do — `vm-db` has no public IP and no NAT gateway configured, so it has no outbound path to the internet at all, not just no inbound path. A completely airgapped-from-the-internet private subnet is more locked down than the project actually required at the install stage, since PostgreSQL itself still needs to be downloaded from somewhere.

The fix was to temporarily attach a public IP address directly to `vm-db`'s network interface, purely to give it outbound internet access long enough to run `apt update && apt install postgresql`, and then immediately disassociate that public IP once the install finished. This is a meaningfully different thing from leaving the database permanently internet-facing: the NSG on `nsg-db` still only allowed **inbound** traffic on 5432/22 from the web subnet the entire time, so even with a public IP momentarily attached, nothing outside the VNet could actually connect in on the database port. The temporary IP only enabled outbound traffic initiated by the VM itself, which is a fundamentally different risk than an open inbound port. This was a good reminder that "private subnet" and "database security" aren't quite the same axis — a subnet can be fully isolated from inbound internet traffic while still needing a deliberate, temporary, and reversible outbound exception to be usable at all.

### The application didn't come up where expected

After deploying the FastAPI app and NSG rules for port 80 (as assumed from the initial planning), the browser returned `ERR_CONNECTION_REFUSED` when hitting `http://<public-ip>/docs`. Reading through `start.sh`'s actual console output showed the real cause: Uvicorn was binding to `0.0.0.0:8000`, not port 80 — the application's own startup script hardcodes port 8000, which hadn't been obvious from just reading the project brief. This wasn't a networking failure at all; it was a mismatch between an assumption (port 80) and what the code actually did (port 8000). The fix was straightforward once identified — add an explicit inbound allow rule for TCP 8000 on `nsg-web`, and access the app on the correct port. It was a useful reminder to verify the actual running behavior of an application rather than inferring its configuration from documentation or convention.

### An incompatible VM image on first attempt

When creating the VMs, the originally planned image (Ubuntu Server 22.04 LTS) was rejected by Azure with a message that it wasn't compatible with the "Trusted launch" security type selected by default for new VMs. Switching to Ubuntu Server 24.04 LTS (Gen2) resolved this immediately, and every subsequent command (`apt`, `systemctl`, PostgreSQL installation, Python setup) behaved identically to what was expected on 22.04 — the newer LTS release didn't require any changes to the planned setup steps, just a different starting image.

---

## 10. Key Learnings

**Security in Azure networking is layered, not singular.** Going into this project, it was easy to think of "putting a VM in a private subnet" as the security control. In practice, the real protection came from three independent layers stacked together: no public IP (nothing to connect to from outside at all), subnet placement (isolated address space with its own NSG), and explicit NSG rules (only the web subnet, only on the ports actually needed). Any one of these failing on its own wouldn't have exposed the database, because the other two were still in place. That redundancy is the point — it's not about finding the one perfect rule, it's about making sure no single misconfiguration is catastrophic.

**NSGs behave differently depending on where you attach them, and that matters for auditing.** Attaching an NSG to a subnet (rather than to each VM's individual network interface) means every resource placed in that subnet automatically inherits the same rules, without needing to remember to configure each new VM individually. This makes the security posture of a whole tier reviewable in one place instead of scattered across every NIC.

**The platform's defaults are often already doing useful work.** Azure's implicit deny-all rule, and the default connectivity between VMs on the same VNet, both quietly did a lot of the heavy lifting in this architecture. Understanding what's already true by default (and why) made it possible to write a much shorter, more intentional set of explicit rules instead of over-specifying everything defensively.

**Isolating a resource from inbound traffic and isolating it from outbound traffic are two separate problems.** The `apt install` failure on `vm-db` was a good practical lesson here — a private subnet with no NAT gateway and no public IP is isolated in both directions by default, but a real deployment sometimes needs a narrow, temporary exception for outbound package installation without ever compromising the inbound security guarantee. Recognizing that these are two different axes of "private" made it much easier to reason about what was actually safe to change temporarily.

**Reading actual runtime output beats assuming configuration.** The port 8000 vs. port 80 issue wasn't a networking bug at all — it was an assumption that didn't match what the application was actually doing. The fastest way to debug it was to go back and read the literal console output from `start.sh` rather than trying to reason from the project brief alone. This is a habit that will matter well beyond this one project: verify what's actually running, don't just trust what should be running.

---

## Repository Structure

```
.
├── api/                  # FastAPI application code
├── .env-sample           # Environment variable template
├── start.sh              # Application startup script
├── README.md             # This file
└── screenshots/          # Deployment evidence (referenced above)
```
