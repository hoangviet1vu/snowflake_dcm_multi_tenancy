# AGENTS.md

Instructions for AI coding agents working in this repository. Read `README.md` first for the
design. This file covers the rules for changing it safely.

## What this repo is

The **platform layer** of a Snowflake multi-tenancy PoC. One Snowflake DCM project, with two targets
(`DEV`, `PROD`) in a single account, provisions tenant environments: databases, warehouses, roles,
tags and boundary grants. Tenants manage their own objects in a separate tenant DCM project, which
is **out of scope** here.

DCM Projects are in Preview. When unsure about syntax or behaviour, check the official docs rather
than guessing:
https://docs.snowflake.com/user-guide/dcm-projects/dcm-projects-files

## Layout and where changes go

| Change | Location |
|---|---|
| What every tenant gets (template) | `sources/macros/tenant.sql` |
| Shared platform objects (tags, governance schema) | `sources/definitions/_platform/` |
| Tenant roles (shared by all envs) | `sources/definitions/tenants/<tenant>/<tenant>.sql` |
| One tenant environment (DB, warehouse, tags) | `sources/definitions/tenants/<tenant>/<env>/<env>.sql` |
| Test tenant for the DEV target | `sources/definitions/tenants/_sandbox/` |
| Targets and environment variables | `manifest.yml` |
| One-time or imperative SQL (bootstrap, cleanup) | `scripts/`, never under `sources/` |

## Hard rules

1. **Definition files contain only `DEFINE`, `GRANT` and `ATTACH` statements.** No `CREATE`,
   `ALTER`, `DROP`, `USE`, `INSERT`, `CALL` or `SET`. Imperative SQL goes in `scripts/`.
2. **Macro files contain only macros.** Macros in `sources/macros/` are global and auto-imported.
   Do **not** use Jinja `import`, `include` or `extends`; DCM rejects them. Name file-local macros
   with a leading underscore.
3. **Never list tenants in `manifest.yml`.** A tenant is registered by its folder. The manifest
   holds only per-target environment values (`platform_env`, `suffix`, `owner_role`,
   `default_wh_size`).
4. **Every tenant file is wrapped in a target guard:**
   - Real tenants: `{% if platform_env == 'PROD' %} ... {% endif %}`
   - Sandbox: `{% if platform_env == 'DEV' %} ... {% endif %}`

   Without the guard, both targets define the same objects in the same account and conflict.
5. **Define each object, grant and tag attachment only once** across the whole project. Duplicates
   fail the run. Shared objects belong in `_platform/`, not in a tenant file.
6. **Use fully qualified names.** Account-level: `T_ACME_ADMIN`. Database-level:
   `T_ACME_DEV_DB.DCM`. Schema-level: `db.schema.object`.
7. **Platform-level objects must include `{{suffix}}`** (for example `GOVERNANCE{{suffix}}`) so DEV
   and PROD never define the same object.
8. **Never delete a tenant folder, rename a tenant, or remove a `DEFINE`** unless the task
   explicitly asks for offboarding or removal. DCM drops removed objects at the next deploy,
   including databases with data. Renaming counts as drop plus create.
9. **No secrets in templating variables or definitions.** Rendered SQL is not redacted.
10. **Do not touch objects owned by the bootstrap** (`PLATFORM_DB`, `PLATFORM_DB.DCM`, the
    `PLATFORM_DCM_*` roles, the DCM project objects). DCM must not manage its own home.
11. **Stay out of the tenant layer.** Do not define schemas other than `<db>.DCM`, or tables,
    stages or streams inside tenant databases. Those belong to the tenant's own DCM project.
    Defining them here would make two projects fight over the same objects.
12. **Do not run `snow dcm deploy`** unless the user explicitly asks. Planning is always allowed.

## Naming conventions

| Object | Pattern | Example |
|---|---|---|
| Tenant roles (shared across envs) | `T_<TENANT>_ADMIN`, `T_<TENANT>_READER` | `T_ACME_ADMIN` |
| Tenant database | `T_<TENANT>_<ENV>_DB` | `T_ACME_STAGING_DB` |
| Tenant DCM schema | `T_<TENANT>_<ENV>_DB.DCM` | `T_ACME_PROD_DB.DCM` |
| Tenant warehouse | `T_<TENANT>_<ENV>_WH` | `T_ACME_DEV_WH` |
| Tags | `PLATFORM_DB.GOVERNANCE{{suffix}}.TENANT_ID / TENANT_ENV` | |

- `<TENANT>`: uppercase, `A-Z0-9_` only, no environment words.
- `<ENV>`: one of `DEV`, `STAGING`, `PROD`.

## Common tasks

### Add a tenant

1. Copy `sources/definitions/tenants/acme/acme.sql` to `tenants/<tenant>/<tenant>.sql` (roles only).
2. For each environment, copy `tenants/acme/<env>/<env>.sql` to `tenants/<tenant>/<env>/<env>.sql`.
3. Replace the tenant name and adjust `wh_size` per environment.
4. Keep the `{% if platform_env == 'PROD' %}` guard in every file.
4. Do not edit `manifest.yml`.

### Change what every tenant gets

1. Edit `sources/macros/tenant.sql` only.
2. Remember that the change applies to **all** tenants on the next PROD deploy. Prefer additive
   changes (new grants, new tags). Call out any change that alters or drops existing objects.
3. Check the `DEFINE WAREHOUSE` limits before changing it. For example, `INITIALLY_SUSPENDED` can't
   be changed after creation.

## Verification

Before saying a change is done, render and plan it. Use `DEV` for template changes and `PROD` for
tenant changes:

```bash
snow dcm plan -c dcm_poc --target DEV  --save-output
snow dcm plan -c dcm_poc --target PROD --save-output
```

Then check:

- [ ] The plan succeeds with no rendering or validation errors.
- [ ] The rendered SQL in `out/` looks as intended (each run recreates `out/`).
- [ ] The plan contains only the expected operations. List every `DROP` and `ALTER` in your summary.
- [ ] There is no `DROP DATABASE`, unless the task is offboarding.
- [ ] A new tenant produces only `CREATE` operations for `T_<TENANT>_*` objects.

If you cannot run the CLI (no connection or credentials), say so explicitly, and describe the
operations you expect the plan to contain.

## Commit and PR conventions

- One logical change per PR. Onboarding, offboarding and template changes go in separate PRs.
- Commit prefixes: `tenant:` (onboard or modify a tenant), `offboard:`, `template:`, `platform:`,
  `docs:`, `ci:`.
- PR description: what changed, the target(s) planned, and a summary of the plan
  (counts of CREATE / ALTER / DROP plus any DROP lines verbatim).
