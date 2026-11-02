# Snowflake DCM Multi-Tenancy (PoC)

Proof of concept for provisioning **tenant environments** in a single Snowflake account using
[Snowflake DCM Projects](https://docs.snowflake.com/user-guide/dcm-projects/dcm-projects-overview)
as the root of resource management.

> DCM Projects are in **Preview**. Syntax and privilege names may change. Verify against the
> official docs before relying on anything here in production.

## Concept

```
┌──────────────────────────── One Snowflake account ────────────────────────────┐
│                                                                               │
│  PLATFORM_DB  (bootstrap, not managed by DCM)                                 │
│  └── DCM                                                                      │
│      ├── PLATFORM_DEV    ← platform DCM project, DEV target  (SANDBOX tenant) │
│      └── PLATFORM_PROD   ← platform DCM project, PROD target (real tenants)   │
│                                                                               │
│  Provisioned by PLATFORM_PROD for each tenant (e.g. ACME):                    │
│    Roles (shared by all envs):  T_ACME_ADMIN, T_ACME_READER                   │
│    Per env (DEV / STAGING / PROD):                                            │
│      T_ACME_<ENV>_DB          database                                        │
│      T_ACME_<ENV>_DB.DCM      schema for the tenant's own DCM project         │
│      T_ACME_<ENV>_WH          warehouse                                       │
│      TENANT_ID / TENANT_ENV   tag values attached to DB and WH                │
│                                                                               │
│  Tenant layer (owned by T_ACME_ADMIN, out of scope of this repo):             │
│      T_ACME_<ENV>_DB.DCM.APP  ← tenant DCM project for tables, stages, streams│
└───────────────────────────────────────────────────────────────────────────────┘
```

Two layers, split by ownership:

| Layer | Owner role | Manages |
|---|---|---|
| Platform (this repo) | `PLATFORM_DCM_DEV` / `PLATFORM_DCM_PROD` | Tenant databases, warehouses, roles, tags, boundary grants |
| Tenant | `T_<TENANT>_ADMIN` | Schemas and objects inside their own databases |

### Key design decisions

- **One manifest, two targets.** `DEV` deploys only the `SANDBOX` tenant so template changes can
  be tested. `PROD` deploys the real tenants.
- **The folder is the registration.** Each tenant has its own folder under
  `sources/definitions/tenants/`, with the tenant roles at the top and one subfolder per
  environment. The manifest never lists tenants, so it stays small as the number of tenants grows.
- **Macros hold the template.** `tenant_roles()` and `tenant_env()` in `sources/macros/` define
  what a tenant gets. Tenant files only call them with tenant-specific arguments.

## Repository layout

```
.
├── manifest.yml                    # 2 targets (DEV, PROD) + templating configs
├── sources/
│   ├── macros/
│   │   └── tenant.sql              # tenant_roles(), tenant_env()
│   └── definitions/
│       ├── _platform/
│       │   └── governance.sql      # GOVERNANCE schema + TENANT_ID / TENANT_ENV tags
│       └── tenants/
│           ├── _sandbox/               # DEV target only
│           │   ├── sandbox.sql         # tenant roles
│           │   └── dev/dev.sql         # DB, warehouse, tags for the DEV env
│           ├── acme/                   # PROD target
│           │   ├── acme.sql            # tenant roles (T_ACME_ADMIN, T_ACME_READER)
│           │   ├── dev/dev.sql         # DB, warehouse, tags for the DEV env
│           │   ├── staging/staging.sql # ... STAGING env
│           │   └── prod/prod.sql       # ... PROD env
│           └── globex/                 # PROD target
│               ├── globex.sql
│               └── prod/prod.sql
├── scripts/
│   ├── 00_bootstrap.sql            # one-time setup, run by ACCOUNTADMIN
│   └── 99_cleanup.sql              # tear down the PoC
├── AGENTS.md                       # rules for AI coding agents
└── out/                            # local PLAN output (git-ignored)
```

Each tenant folder holds one SQL file for the tenant's **roles** (shared by all environments) and
one subfolder per **environment** (`dev`, `staging`, `prod`). An environment file defines that
environment's database, warehouse and tag attachments. Adding or removing an environment is a
matter of adding or removing its subfolder.

Everything under `sources/` is uploaded and rendered by DCM. Anything that must **not** be run by
DCM, such as bootstrap SQL, lives in `scripts/`.

## Prerequisites

- A Snowflake account with DCM Projects available, and `ACCOUNTADMIN` for the one-time bootstrap
- A recent [Snowflake CLI](https://docs.snowflake.com/developer-guide/snowflake-cli/index) with the
  `snow dcm` commands
- A CLI connection in `~/.snowflake/config.toml`, for example:

```toml
[connections.dcm_poc]
account = "MYORG-MYACCOUNT"
user = "YOUR_USER"
authenticator = "externalbrowser"
role = "PLATFORM_DCM_DEV"
```

## Getting started

### 1. Bootstrap (once, as ACCOUNTADMIN)

`scripts/00_bootstrap.sql` creates the objects DCM cannot manage for itself: the database and schema
that hold the project objects, the owner roles, and the project objects.

```sql
USE ROLE ACCOUNTADMIN;

-- Home for the DCM project objects (not managed by DCM)
CREATE DATABASE IF NOT EXISTS PLATFORM_DB;
CREATE SCHEMA   IF NOT EXISTS PLATFORM_DB.DCM;

-- Project owner roles
CREATE ROLE IF NOT EXISTS PLATFORM_DCM_DEV;
CREATE ROLE IF NOT EXISTS PLATFORM_DCM_PROD;
GRANT ROLE PLATFORM_DCM_DEV  TO ROLE SYSADMIN;
GRANT ROLE PLATFORM_DCM_PROD TO ROLE SYSADMIN;
GRANT ROLE PLATFORM_DCM_DEV  TO USER YOUR_USER;
GRANT ROLE PLATFORM_DCM_PROD TO USER YOUR_USER;

-- Account-level privileges needed to provision tenant objects
GRANT CREATE DATABASE, CREATE WAREHOUSE, CREATE ROLE, APPLY TAG
  ON ACCOUNT TO ROLE PLATFORM_DCM_DEV;
GRANT CREATE DATABASE, CREATE WAREHOUSE, CREATE ROLE, APPLY TAG
  ON ACCOUNT TO ROLE PLATFORM_DCM_PROD;

-- Access to the project home
GRANT USAGE ON DATABASE PLATFORM_DB TO ROLE PLATFORM_DCM_DEV;
GRANT USAGE ON DATABASE PLATFORM_DB TO ROLE PLATFORM_DCM_PROD;
GRANT CREATE SCHEMA ON DATABASE PLATFORM_DB TO ROLE PLATFORM_DCM_DEV;   -- GOVERNANCE_PDEV
GRANT CREATE SCHEMA ON DATABASE PLATFORM_DB TO ROLE PLATFORM_DCM_PROD;  -- GOVERNANCE
GRANT USAGE, CREATE DCM PROJECT ON SCHEMA PLATFORM_DB.DCM TO ROLE PLATFORM_DCM_DEV;
GRANT USAGE, CREATE DCM PROJECT ON SCHEMA PLATFORM_DB.DCM TO ROLE PLATFORM_DCM_PROD;

-- Project objects (one per target)
USE ROLE PLATFORM_DCM_DEV;
CREATE DCM PROJECT IF NOT EXISTS PLATFORM_DB.DCM.PLATFORM_DEV;
USE ROLE PLATFORM_DCM_PROD;
CREATE DCM PROJECT IF NOT EXISTS PLATFORM_DB.DCM.PLATFORM_PROD;
```

```bash
snow sql -c dcm_poc -f scripts/00_bootstrap.sql
```

### 2. Review the manifest

```yaml
manifest_version: 2
type: DCM_PROJECT
default_target: DEV

targets:
  DEV:
    account_identifier: MYORG-MYACCOUNT
    project_name: PLATFORM_DB.DCM.PLATFORM_DEV
    project_owner: PLATFORM_DCM_DEV
    templating_config: DEV
  PROD:
    account_identifier: MYORG-MYACCOUNT
    project_name: PLATFORM_DB.DCM.PLATFORM_PROD
    project_owner: PLATFORM_DCM_PROD
    templating_config: PROD

templating:
  defaults:
    default_wh_size: "XSMALL"
  configurations:
    DEV:
      platform_env: "DEV"
      suffix: "_PDEV"        # keeps DEV platform objects apart from PROD in the same account
      owner_role: "PLATFORM_DCM_DEV"
    PROD:
      platform_env: "PROD"
      suffix: ""
      owner_role: "PLATFORM_DCM_PROD"
```

### 3. Plan and deploy to DEV (SANDBOX tenant)

```bash
snow dcm plan   -c dcm_poc --target DEV --save-output   # inspect out/ before deploying
snow dcm deploy -c dcm_poc --target DEV --alias poc-v1
```

Check the result:

```sql
SHOW DATABASES  LIKE 'T_SANDBOX_%';
SHOW WAREHOUSES LIKE 'T_SANDBOX_%';
SHOW ROLES      LIKE 'T_SANDBOX_%';
SELECT * FROM TABLE(PLATFORM_DB.INFORMATION_SCHEMA.TAG_REFERENCES('T_SANDBOX_DEV_DB', 'DATABASE'));
```

### 4. Plan and deploy to PROD (real tenants)

```bash
snow dcm plan   -c dcm_poc --target PROD --save-output
snow dcm deploy -c dcm_poc --target PROD --alias poc-v1
```

## Template

### Macros (`sources/macros/tenant.sql`)

Macros in `sources/macros/` are global: every definition file can call them without an import.

```sql
{% macro tenant_roles(t) %}
DEFINE ROLE T_{{t}}_ADMIN;
DEFINE ROLE T_{{t}}_READER;
GRANT ROLE T_{{t}}_READER TO ROLE T_{{t}}_ADMIN;
GRANT ROLE T_{{t}}_ADMIN  TO ROLE {{owner_role}};   -- platform keeps control of tenant roles
{% endmacro %}

{% macro tenant_env(t, env, wh_size=default_wh_size) %}
DEFINE DATABASE  T_{{t}}_{{env}}_DB;
DEFINE SCHEMA    T_{{t}}_{{env}}_DB.DCM;
DEFINE WAREHOUSE T_{{t}}_{{env}}_WH
  WITH WAREHOUSE_SIZE = '{{wh_size}}' AUTO_SUSPEND = 60 INITIALLY_SUSPENDED = TRUE;

GRANT USAGE ON WAREHOUSE T_{{t}}_{{env}}_WH TO ROLE T_{{t}}_ADMIN;
GRANT USAGE, CREATE SCHEMA ON DATABASE T_{{t}}_{{env}}_DB TO ROLE T_{{t}}_ADMIN;
GRANT USAGE, CREATE DCM PROJECT ON SCHEMA T_{{t}}_{{env}}_DB.DCM TO ROLE T_{{t}}_ADMIN;

ATTACH TAG PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ID  = '{{t}}'   TO DATABASE  T_{{t}}_{{env}}_DB;
ATTACH TAG PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ENV = '{{env}}' TO DATABASE  T_{{t}}_{{env}}_DB;
ATTACH TAG PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ID  = '{{t}}'   TO WAREHOUSE T_{{t}}_{{env}}_WH;
ATTACH TAG PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ENV = '{{env}}' TO WAREHOUSE T_{{t}}_{{env}}_WH;
{% endmacro %}
```

### Platform objects (`sources/definitions/_platform/governance.sql`)

```sql
DEFINE SCHEMA PLATFORM_DB.GOVERNANCE{{suffix}};
DEFINE TAG    PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ID;
DEFINE TAG    PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ENV;
```

### Tenant roles (`sources/definitions/tenants/acme/acme.sql`)

```sql
{% if platform_env == 'PROD' %}
{{ tenant_roles('ACME') }}
{% endif %}
```

### Tenant environments (`sources/definitions/tenants/acme/<env>/<env>.sql`)

Each environment file carries the same target guard and calls `tenant_env` for its own environment.
For example `acme/prod/prod.sql`:

```sql
{% if platform_env == 'PROD' %}
{{ tenant_env('ACME', 'PROD', wh_size='MEDIUM') }}
{% endif %}
```

`acme/dev/dev.sql` and `acme/staging/staging.sql` are the same with `'DEV'` / `'STAGING'` and their
own `wh_size`. The sandbox files use the same macros guarded by `{% if platform_env == 'DEV' %}`.

## Operations

### Onboard a tenant

1. Create `sources/definitions/tenants/<tenant>/<tenant>.sql` for the roles, and one
   `<tenant>/<env>/<env>.sql` per environment. Copy the `acme` files and change the name and sizes.
2. Open a PR. Run `snow dcm plan --target PROD` and confirm the plan contains **only CREATE**
   operations for `T_<TENANT>_*` objects.
3. Merge, then deploy to PROD.

No change to `manifest.yml` is needed.

### Change the template

1. Edit the macros in `sources/macros/tenant.sql`.
2. Deploy to `DEV` and verify on the `SANDBOX` tenant.
3. Run `plan --target PROD` and review the change for **every** tenant before deploying.

### Offboard a tenant

Deleting a tenant folder **drops all of its databases, warehouses and roles** at the next deploy.
Treat it as a deliberate, approved operation:

1. Confirm the tenant's data is exported or no longer needed.
2. Remove the folder in a dedicated PR labelled `offboarding`.
3. Review the plan's `DROP` list, then deploy.

### Safety rules

- Never deploy without reading the plan first. DCM drops anything removed from the definitions.
- CI should fail any plan that contains `DROP DATABASE` unless the PR is labelled `offboarding`.
- Project size limit: about 10,000 entities per project. One tenant is roughly 50 entities, which
  gives about 200 tenants per platform project. Split the project before reaching that.

## Cleanup

`scripts/99_cleanup.sql`: drop the tenant objects by deploying with empty tenant folders, or by
hand, then:

```sql
USE ROLE ACCOUNTADMIN;
DROP DCM PROJECT IF EXISTS PLATFORM_DB.DCM.PLATFORM_DEV;
DROP DCM PROJECT IF EXISTS PLATFORM_DB.DCM.PLATFORM_PROD;
DROP DATABASE IF EXISTS PLATFORM_DB;
DROP ROLE IF EXISTS PLATFORM_DCM_DEV;
DROP ROLE IF EXISTS PLATFORM_DCM_PROD;
```

## References

- [Snowflake DCM Projects](https://docs.snowflake.com/user-guide/dcm-projects/dcm-projects-overview)
- [DCM Projects files and templates](https://docs.snowflake.com/user-guide/dcm-projects/dcm-projects-files)
- [Supported object types](https://docs.snowflake.com/user-guide/dcm-projects/dcm-projects-supported-entities)
- [Enterprise use cases](https://docs.snowflake.com/en/user-guide/dcm-projects/dcm-projects-enterprise)
- [Managing DCM projects with Snowflake CLI](https://docs.snowflake.com/en/developer-guide/snowflake-cli/data-pipelines/dcm-projects)
