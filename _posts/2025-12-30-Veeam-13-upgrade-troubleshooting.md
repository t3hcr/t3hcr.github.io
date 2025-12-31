*Author's note: This post is being made to help others and is the result of my interactions with AI and should be considered as such. ALWAYS backup and test before implementing any fixes. If you're unsure or uncomfortable making changes (or fixing them) contact Veeam support!*

## Overview

Upgrading **Veeam Backup & Replication** from **v12 to v13** is usually painless — until the installer hard-stops with this message:

> **[Error] Outdated Veeam Agents**  
> Please upgrade or remove the following Veeam agents from the product configuration:  
> **OLD-AGENT-HOST (Veeam Agent for Windows 3.0.2.1170)**

The problem?  
`OLD-AGENT-HOST` **no longer existed anywhere** in the environment.

- No backups  
- No protection groups  
- No inventory objects  
- Nothing visible in the UI  

And yet the v13 installer refused to proceed.

This post walks through **why this happens**, **why the usual cleanup methods fail**, and **how to fix it correctly** when your Veeam configuration database is running on **PostgreSQL**.

---

## What’s Actually Going On

Veeam v13 introduced **stricter prerequisite checks** than v12. One of those checks validates **agent inventory state**, not just backup jobs or objects.

That inventory data lives in Veeam’s **Enterprise Plug-in (EP)** tables and can persist even after:

- a machine is decommissioned  
- the agent is uninstalled  
- the backup object is deleted  

Veeam v12 tolerates this inconsistency.  
Veeam v13 does **not**.

There is no “ignore” or “continue anyway” option. The stale record must be removed.

---

## Why the UI and PowerShell Show Nothing

Every obvious place was checked:

- **Home → Backups → Agent Backups**
- **Inventory → Physical Infrastructure**
- **Protection Groups**

Nothing referenced `OLD-AGENT-HOST`.

PowerShell wasn’t helpful either — even when the Veeam cmdlets are available, these EP inventory records often don’t surface in a way that allows removal.

At this point, the **configuration database is the only source of truth**.

---

## Safety First

Before touching the database:

- Take a **Veeam Configuration Backup**
- Preferably stop write-heavy services (at minimum **Veeam Backup Service**)
- Use transactions so you can safely roll back if needed

```sql
BEGIN;
-- changes
COMMIT;
-- or ROLLBACK;
```

---

## Step 1 — Identify the Blocking Agent Version

The installer helpfully tells you the agent version it’s unhappy with.

Query the EP agent inventory:

```sql
SELECT
    host_id,
    bobject_id,
    agent_version,
    is_upgrade_required,
    agent_installation_status
FROM "backup.model.epagents"
WHERE agent_version = '3.0.2.1170'
   OR agent_version ILIKE '3.0.2.%';
```

Example Result: 

```
host_id	agent_version
<EP_HOST_UUID>	3.0.2.1170
```

This confirms the installer is blocking on a legacy Agent 3.x record.

---

## Step 2 — Map the Agent Record to a Host

The epagents table does not store hostnames directly.
Host details are stored in:

backup.model.ephosts

Inspect the table (optional but recommended):

```sql
\d "backup.model.ephosts"
```

Now retrieve the EP host record:

```sql
SELECT
    id,
    connection_point,
    display_name,
    os_version,
    host_state,
    host_aux_data
FROM "backup.model.ephosts"
WHERE id = '<EP_HOST_UUID>'::uuid;
```

### What You’ll See

If this is the ghost entry, fields like these will jump out immediately:
- connection_point: OLD-AGENT-HOST
- display_name: OLD-AGENT-HOST

At this point, you’ve proven exactly what’s blocking the upgrade.

---

## Why Deleting the EP Host Is the Correct Fix

This is the critical relationship:

```pgsql
backup.model.epagents.host_id
  → references backup.model.ephosts(id)
  → ON DELETE CASCADE
```

Deleting the EP host automatically removes:
- the EP agent inventory record
- EP agent membership rows
- EP agent backup statistics
- related EP metadata

This is the clean, supported way to resolve the condition.

---

## Step 3 — Delete the Orphaned EP Host

Delete the EP host inside a transaction:

```sql
BEGIN;

DELETE FROM "backup.model.ephosts"
WHERE id = '<EP_HOST_UUID>'::uuid;

COMMIT;
```

Verify Cleanup

```sql
SELECT *
FROM "backup.model.epagents"
WHERE host_id = '<EP_HOST_UUID>'::uuid;
```

Expected result:

0 rows

## Step 4 — Clear Cached Installer State

After database changes, don’t reuse an already-running installer window.

On the Veeam server, restart the following services:
- Veeam Backup Service
- Veeam Installer Service
- Veeam Broker Service (if present)

Then launch the v13 installer fresh.

The “Outdated Veeam Agents” check should now be gone.

## Common Gotchas
PostgreSQL case sensitivity
- Unquoted identifiers fold to lowercase
- Quoted identifiers are case-sensitive

If a table is named bobjects, "BObjects" will not work.

The hostname may not be stored as name

You may need to correlate using:
- connection_point
- display_name
- agent_version
- host_id

# Deleting bobjects is not enough

The v13 installer checks EP agent inventory, not just backup objects.

## Lessons Learned

1. Veeam v13 validates agent inventory, not just backups
2. Orphaned EP records can block upgrades silently
3. Removing the EP host is the correct cleanup method
4. Restart services to avoid cached prerequisite results

## Final Advice

- Always take a configuration backup before DB work
- Delete parent EP host records, not child agent rows
- Trust what the installer tells you — it’s reading real data

This is one of those cases where the UI shows nothing, PowerShell can’t help, and the database is the only place the truth lives.

