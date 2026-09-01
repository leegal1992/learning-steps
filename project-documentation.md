# LearningSteps Origins — Azure 2-Tier Deployment

This document describes the deployment of the **LearningSteps API** (FastAPI + PostgreSQL) to a secure 2-tier architecture on Microsoft Azure, built as part of the LearningSteps Origins individual project.

- **Codebase used:** `reference` branch (Reference Implementation)
- **Cloud provider:** Microsoft Azure
- **Region:** Germany West Central

---

## 1. Architecture Diagram

```
                          Internet
                             │
                             │ HTTP (port 8000)
                             ▼
                 ┌───────────────────────┐
                 │   nsg-web (NSG)       │
                 │  Allow: 8000, 22*     │
                 └──────────┬────────────┘
                             │
                 ┌───────────────────────┐
                 │ subnet-web-public     │
                 │ 192.168.1.0/24        │
                 │                       │
                 │  ┌─────────────────┐  │
                 │  │   vm-web        │  │
                 │  │  Public IP:     │  │
                 │  │  20.170.34.70   │  │
                 │  │  FastAPI :8000  │  │
                 │  └────────┬────────┘  │
                 └───────────┼───────────┘
                             │ TCP 5432
                             │ (private VNet traffic only)
                 ┌───────────▼───────────┐
                 │ subnet-db-private     │
                 │ 192.168.0.0/24        │
                 │                       │
                 │  ┌─────────────────┐  │
                 │  │   vm-db         │  │
                 │  │  No Public IP   │  │
                 │  │  PostgreSQL     │  │
                 │  │  192.168.0.4    │  │
                 │  └─────────────────┘  │
                 └───────────┬───────────┘
                             │
                 ┌───────────────────────┐
                 │   nsg-db (NSG)        │
                 │  Allow: 5432, 22 from │
                 │  192.168.1.0/24 only  │
                 └───────────────────────┘

        VNet: vnet-learningsteps (192.168.0.0/16)
        Resource Group: rg-learningsteps

*SSH on nsg-web restricted to admin's home IP only.
```

![VNet creation](screenshots/04-vnet-learningsteps-creation.png)
![Public subnet](screenshots/02-subnet-web-public.png)
![Private subnet](screenshots/03-subnet-db-private.png)

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

- **Two isolated subnets** separate the internet-facing tier from the data tier, so the database is never directly routable from the internet regardless of NSG misconfiguration on the web side.
- **`nsg-db` only allows inbound traffic from `192.168.1.0/24`** (the web subnet) on ports 5432 and 22, rather than relying solely on default VNet connectivity — this keeps the rule explicit and auditable, and would still protect the DB tier if a future VM were added to the web subnet without authorization.
- **SSH to `vm-db` is only reachable by hopping through `vm-web`**, since `vm-db` has no public IP and its NSG only allows SSH from the web subnet.
- **SSH to `vm-web` is restricted to the admin's home IP** rather than left open to `Any`, reducing brute-force exposure on the one VM that is internet-facing.
- **No explicit "deny all" rules were added** — Azure's implicit deny-all at priority 65500 already blocks everything not explicitly allowed, keeping the rule set minimal (least privilege by default, not by an extra rule).

---

## 9. Challenges

- **`vm-db` had no outbound internet access**, since it lives in a private subnet with no public IP — this blocked `apt install postgresql` initially. Resolved by temporarily attaching a public IP to `vm-db` purely for outbound package downloads, then removing it immediately afterward. This didn't compromise the security requirement, since the NSG still blocked all *inbound* traffic from the internet during that window.
- **The application listens on port 8000, not port 80** as initially assumed from the brief — the initial NSG only allowed port 80, causing `ERR_CONNECTION_REFUSED` in the browser. Resolved by adding an explicit inbound allow rule for port 8000 on `nsg-web`.
- **Azure's newer "Trusted launch" security type** rejected the originally planned Ubuntu 22.04 LTS image; switched to Ubuntu 24.04 LTS Gen2, which is fully compatible and required no changes to any of the setup commands.

---

## 10. Key Learnings

- Subnetting alone doesn't secure anything — it's the **combination of subnet placement + NSG rules + no public IP** that actually enforces the private tier.
- NSGs attached at the **subnet level** apply to every resource in that subnet automatically, which is simpler to reason about and audit than attaching NSGs per-NIC.
- Azure's implicit deny-all rule means a minimal, explicit allow-list is sufficient for least privilege — no need to manually add deny rules.
- Provisioning a private VM sometimes still needs **temporary, deliberate, and reversible** exceptions (like brief outbound internet access) to complete setup — the key is ensuring inbound exposure is never opened, and that any temporary exception is closed again immediately after use.

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
