# <img src="https://d1r5llqwmkrl74.cloudfront.net/notebooks/fsi/fs-lakehouse-logo-transparent.png" width="600px">
# Teradata → Databricks Security Migration Accelerator
 
[![Databricks](https://img.shields.io/badge/Databricks-Solution%20Accelerator-FF3621)](https://marketplace.databricks.com)
[![DBR](https://img.shields.io/badge/DBR-16.2%20LTS%2B-blue)](https://docs.databricks.com/release-notes/runtime/16.2.html)
[![Unity Catalog](https://img.shields.io/badge/Unity%20Catalog-Required-green)](https://docs.databricks.com/data-governance/unity-catalog/index.html)
[![SCIM API](https://img.shields.io/badge/SCIM%20API-Enabled-blueviolet)](https://docs.databricks.com/administration-guide/users-groups/scim/index.html)
[![Python](https://img.shields.io/badge/Python-3.10%2B-yellow)](https://www.python.org)
[![Tests](https://img.shields.io/badge/Tests-45%20passing-brightgreen)](tests/)
[![License](https://img.shields.io/badge/License-Databricks-orange)](https://databricks.com/db-license-source)
 
---
 
## Overview
 
The **Teradata → Databricks Security Migration Accelerator** automates the **complete, end-to-end migration of Teradata security objects** to **Databricks Unity Catalog**. It extracts users, roles, role memberships, and object-level privileges directly from Teradata system tables (`DBC.*`), translates them to their Databricks equivalents, provisions everything via the **SCIM API** and **SQL `GRANT` statements**, and then runs a full reconciliation to verify migration completeness.
 
Security migration is one of the most complex, error-prone, and time-consuming phases of any data platform modernization project. This accelerator eliminates weeks of manual work — typically involving spreadsheets, manual API calls, and tedious privilege auditing — and replaces it with a **single, observable, repeatable pipeline**.
 
| Teradata Concept | Databricks Equivalent | Provisioning Method |
|---|---|---|
| User (`DBC.UsersV`) | Account-level user | SCIM API `POST /Users` |
| Role (`DBC.RoleInfo`) | Account-level group (Unity Catalog-aware) | SCIM API `POST /Groups` |
| User-role assignment (`DBC.RoleMembersV`) | Group membership | SCIM API `PATCH /Groups` |
| Object privilege (`DBC.AllRightsV`) | `GRANT` statement in Unity Catalog | SQL `GRANT ON CATALOG/SCHEMA/TABLE` |
 
---
 
## Architecture
 
```
Teradata DBC Tables          Accelerator Phases               Databricks
─────────────────    ─────────────────────────────────    ─────────────────────
DBC.UsersV       ──► 1. EXTRACT      Read DBC system tables
DBC.RoleInfo         2. CLASSIFY     Label users (human / service / admin / dormant)
DBC.RoleMembersV     3. PROVISION    Create users & groups ──────────────────► SCIM API
DBC.AllRightsV       4. GRANT        Apply UC privileges   ──────────────────► SQL GRANT
                     5. RECONCILE    Verify completeness   ◄── query back & compare
                     6. ALERT        Email on failure (Mailgun)
```
 
---
 
## Key Features
 
- **Automated user classification** — Tags every Teradata user as `human`, `service_account`, `admin`, or `dormant` based on configurable patterns; dormant users can be skipped
- **Privilege translation** — Maps Teradata access rights (`R`, `I`, `U`, `D`, `E`, `A`) to Unity Catalog `SELECT`, `MODIFY`, `EXECUTE`, `ALL PRIVILEGES`
- **SCIM-native provisioning** — Uses the Databricks Account SCIM API to create users and groups, ensuring proper SSO federation support
- **Unity Catalog GRANT statements** — Generates and executes fine-grained `GRANT` SQL for catalog, schema, and table-level privileges
- **Full reconciliation** — Verifies user count, group count, group memberships, and privilege spot-checks post-migration
- **JSON security registry** — All extracted security objects serialized to `security_registry.json` for auditability and re-runs
- **Dry run mode** — Logs every planned action without writing anything to Databricks
---
 
## Prerequisites
 
Before running the accelerator, ensure the following are in place:
 
| Requirement | Details |
|---|---|
| **Databricks Runtime** | 16.2 LTS or above |
| **Unity Catalog** | Must be enabled on the target workspace |
| **Account Admin** | Databricks account-level admin token (for SCIM API calls) |
| **Workspace Admin** | Workspace-level token (for SQL `GRANT` statements) |
| **Network** | Connectivity from the execution environment to the Teradata host and Databricks workspace |
 
---
 
## Installation & Setup
 
### Step 1 — Configure INI Files
 
Open the `securitypipeline.dist/config/` folder and fill in your credentials and settings in each `.ini` file. Also configure the metadata files in `securitypipeline.dist/metadata/`.
 
```
securitypipeline.dist/
├── config/
│   ├── sources.ini          ← Teradata host, port, username, password
│   ├── databricks.ini       ← Workspace URL, account ID, catalog, warehouse path
│   ├── security.ini         ← User classification rules, group naming, exclusions
│   ├── paths.ini            ← Output paths for logs and artifacts
│   └── alerts.ini           ← Mailgun API key, from/to addresses
└── metadata/
    ├── privilege_mapping.ini    ← Teradata → UC privilege override mappings
    ├── security_scope.ini       ← Scope of objects to migrate
    └── user_email_mapping.csv   ← Manual username → email overrides
```
 
Refer to the [Configuration Reference](#configuration-reference) section below for details on each setting.
 
> **Security note:** Never store passwords or tokens directly in INI files. Use environment variables — the pipeline reads credentials from `TERADATA_PASSWORD`, `DATABRICKS_ACCOUNT_TOKEN`, and `DATABRICKS_WORKSPACE_TOKEN` automatically.
 
---
 
### Step 2 — Export Users for Email Mapping
 
Before running the full pipeline, export the Teradata user list to generate the email mapping file. Replace `smartbots.ai` with your organisation's email domain:
 
```bash
.\export_users_for_mapping.dist\export_users_for_mapping.exe --source teradata --email-domain smartbots.ai
```
 
This produces a `user_email_mapping.csv` file in the `metadata/` folder. Review the file and make any manual corrections to usernames or email addresses before proceeding.
 
> This step ensures every Teradata user is correctly mapped to a corporate email address before accounts are provisioned in Databricks.
 
---
 
### Step 3 — Run the Security Migration Pipeline
 
```bash
.\securitypipeline.dist\securitypipeline.exe
```
 
The pipeline will execute all migration phases automatically — preflight, extraction, classification, user and group provisioning, privilege grants, reconciliation, and alerts.
 
#### Optional flags
 
```bash
# Verify connectivity only — no changes made
.\securitypipeline.dist\securitypipeline.exe --test-connection
 
# Dry run — logs all planned actions, writes nothing to Databricks
.\securitypipeline.dist\securitypipeline.exe --dry-run
 
# Extract only — writes security_registry.json, no provisioning
.\securitypipeline.dist\securitypipeline.exe --extract-only
```
 
---
 
## Configuration Reference
 
### `config/sources.ini`
 
Defines the Teradata connection details.
 
```ini
[teradata]
host     = your-teradata-host.company.com
port     = 1025
database = DBC
username = your_username
# Do not store passwords in files. Use env var TERADATA_PASSWORD instead.
password =
```
 
---
 
### `config/databricks.ini`
 
Defines your Databricks workspace and account connection details.
 
```ini
[databricks]
workspace_host   = https://adb-xxxx.azuredatabricks.net
account_id       = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
catalog          = main
http_path        = /sql/1.0/warehouses/xxxx
# Appended to Teradata usernames to form email addresses
email_domain     = yourcompany.com
```
 
---
 
### `config/security.ini`
 
Controls user classification, group naming conventions, and exclusions.
 
```ini
[classification]
service_patterns       = svc_*, etl_*, batch_*, app_*
admin_patterns         = *_dba, admin_*, dbc, tdadmin
dormant_days_threshold = 180
skip_dormant           = true
 
[groups]
group_name_prefix =
group_name_suffix = _uc
 
[exclusions]
excluded_users = dbc, tdadmin, sysadmin
excluded_roles = ALL, PUBLIC
```
 
---
 
### `config/alerts.ini`
 
Configures Mailgun email alerts.
 
```ini
[mailgun]
api_key        = your-mailgun-api-key
sending_domain = mg.yourcompany.com
from_address   = pipeline@mg.yourcompany.com
to_address     = your-team@yourcompany.com
```
 
---
 
### `metadata/user_email_mapping.csv`
 
A CSV file used to manually override the auto-generated username-to-email mappings. This file is produced by the `export_users_for_mapping.exe` step and should be reviewed before running the full pipeline.
 
```csv
teradata_username,email_address
jsmith,john.smith@yourcompany.com
svc_etl,etl-service@yourcompany.com
```
 
---
 
## User Classification Logic
 
Every extracted Teradata user is automatically classified into one of four categories:
 
```
Teradata User
      │
      ▼
Matches admin pattern? (e.g. *_dba, admin_*, dbc)
├── Yes ──► admin
└── No
      │
      ▼
Matches service pattern? (e.g. svc_*, etl_*, batch_*)
├── Yes ──► service_account
└── No
      │
      ▼
Last login > dormant_days_threshold days ago?
├── Yes ──► dormant
│         ├── skip_dormant = true  ──► SKIP (not provisioned)
│         └── skip_dormant = false ──► provisioned as inactive
└── No  ──► human
```
 
---
 
## Privilege Translation
 
Teradata access rights are automatically translated to Unity Catalog `GRANT` statements:
 
| Teradata Right | Unity Catalog Privilege | Notes |
|---|---|---|
| `R` — SELECT | `SELECT` | Read access |
| `I` — INSERT | `MODIFY` | Write access |
| `U` — UPDATE | `MODIFY` | Write access |
| `D` — DELETE | `MODIFY` | Write access |
| `E` — EXECUTE | `EXECUTE` | Stored procedure / function |
| `A` — ALL | `ALL PRIVILEGES` | Full access |
| `CREATETABLE` | `CREATE TABLE` | DDL permission |
| DDL-only (`CT`, `CR`, `CV`, `CP`) | — | Excluded from migration by default |
 
---
 
## Pipeline Flow
 
```
  securitypipeline.exe
          │
          ▼
  ┌─────────────────────┐
  │  Load Config         │
  └──────────┬──────────┘
             │
             ▼
       Config valid?
       ├── No  ──► EXIT 2: Config Error
       └── Yes
             │
             ▼
  ┌──────────────────────┐
  │  Run Preflight Checks │
  └──────────┬───────────┘
             │
             ▼
       Preflight OK?
       ├── No  ──► EXIT 3: Preflight Failed
       └── Yes
             │
             ▼
  ┌──────────────────────────────────────────┐
  │  EXTRACT                                  │
  │  Query DBC.UsersV, DBC.RoleInfo,          │
  │  DBC.RoleMembersV, DBC.AllRightsV         │
  │  → Write security_registry.json           │
  └──────────┬───────────────────────────────┘
             │
             ▼
       Extract OK?
       ├── No  ──► EXIT 5: Extract Failed
       └── Yes
             │
             ▼
  ┌──────────────────────────────────────────┐
  │  CLASSIFY & MAP                           │
  │  Label users: human / service /           │
  │  admin / dormant                          │
  │  Map roles → groups                       │
  │  Translate privileges to UC format        │
  └──────────┬───────────────────────────────┘
             │
             ▼
       Classification OK?
       ├── No  ──► EXIT 6: Classification Failed
       └── Yes
             │
             ▼
  ┌──────────────────────────────────────────┐
  │  PROVISION USERS & GROUPS                 │
  │  SCIM POST /Users                         │
  │  SCIM POST /Groups                        │
  │  SCIM PATCH group memberships             │
  └──────────┬───────────────────────────────┘
             │
             ▼
       Provisioning OK?
       ├── No  ──► EXIT 7: Provisioning Failed
       └── Yes
             │
             ▼
  ┌──────────────────────────────────────────┐
  │  APPLY UC PRIVILEGES                      │
  │  GRANT SELECT / MODIFY /                  │
  │  EXECUTE / ALL PRIVILEGES                 │
  └──────────┬───────────────────────────────┘
             │
             ▼
       Grants OK?
       ├── No  ──► EXIT 8: Privilege Grant Failed
       └── Yes
             │
             ▼
  ┌──────────────────────────────────────────┐
  │  RECONCILE                                │
  │  Verify users · groups · memberships      │
  │  Privilege spot-check                     │
  │  → Write recon JSON report                │
  └──────────┬───────────────────────────────┘
             │
             ▼
       Recon PASS?
       ├── No  ──► EXIT 9: Reconciliation FAIL
       └── Yes
             │
             ▼
  ┌──────────────────────┐
  │  Send Alert Email     │
  └──────────┬───────────┘
             │
             ▼
        EXIT 0: Migration Complete ✓
```
 
---
 
## Exit Codes
 
| Code | Meaning |
|---|---|
| `0` | Migration complete — reconciliation PASS |
| `2` | Configuration or argument error — check your INI files |
| `3` | Preflight failed — connectivity or permission issue |
| `4` | Teradata connection failed |
| `5` | Security extract failed |
| `6` | Classification failed |
| `7` | User/group provisioning failed |
| `8` | UC privilege grant failed |
| `9` | Reconciliation FAIL — action required, review recon report |
| `10` | Unexpected error |
 
---
 
## Artifacts & Outputs
 
After each run, the following outputs are generated automatically:
 
```
artifacts/
├── pipeline_control.csv          ← Phase-level audit trail
├── logs/                         ← One log file per run
└── output/
    ├── recon/                    ← Reconciliation JSON report per run
    └── schema/                   ← security_registry.json
```
 
### Reconciliation Report Sample
 
```json
{
  "run_id": "sec-run-001",
  "timestamp": "2026-04-28T10:15:00Z",
  "overall_status": "PASS",
  "users": {
    "expected_count": 120,
    "found_count":    120,
    "missing_count":  0,
    "status": "PASS"
  },
  "groups": {
    "expected_count": 15,
    "found_count":    15,
    "missing_count":  0,
    "status": "PASS"
  },
  "group_memberships": {
    "expected_count": 340,
    "found_count":    340,
    "missing_count":  0,
    "status": "PASS"
  },
  "privileges": {
    "sample_size": 20,
    "verified":    20,
    "status":      "PASS"
  }
}
```
 
| Recon Status | Meaning |
|---|---|
| `PASS` | All checks passed — migration complete |
| `WARN` | Minor issues detected — review recommended |
| `FAIL` | Users, groups, or memberships missing — action required |
 
---
 
## Permissions Requirements
 
**Databricks Account (SCIM API):**
- Account Admin role — required to create users and groups via `POST /api/2.0/accounts/{id}/scim/v2/Users`
- The account-level token must have `account:admin` scope
**Databricks Workspace (SQL GRANT):**
- `GRANT` privilege on the target catalogs, schemas, and tables
- Workspace Admin for cross-user grant operations
**Teradata:**
- `SELECT` on `DBC.UsersV`, `DBC.RoleInfo`, `DBC.RoleMembersV`, `DBC.AllRightsV`
- Read-only access — no writes to Teradata at any stage
**Mailgun (optional):**
- A verified sending domain and API key with `send` permissions
---
 
## Library Dependencies
 
| Library | Purpose | License |
|---|---|---|
| `teradatasql` | Teradata Python connector | Teradata |
| `databricks-sdk` | Databricks account SCIM & workspace API | Apache 2.0 |
| `databricks-sql-connector` | Databricks SQL Warehouse client | Apache 2.0 |
| `requests` | HTTP client for SCIM API calls | Apache 2.0 |
 
---
 
&copy; 2026 Databricks, Inc. All rights reserved. The source in this accelerator is provided subject to the [Databricks License](https://databricks.com/db-license-source). All included or referenced third-party libraries are subject to the licenses listed above.
