---
name: windows-sql-server
description: "Manage SQL Server instances on Windows — uninstall, install, service/registry cleanup, and elevated execution patterns for when Hermes lacks admin privileges."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [windows, sql-server, admin, devops]
    related_skills: []
---

# Windows SQL Server Management

## Overview

Manage SQL Server instances on Windows when Hermes Agent does not have administrator privileges. The core pattern is to create `.bat` scripts that the user executes via right-click → "Run as Administrator".

## When to Use

- Uninstalling named SQL Server instances (e.g., SQLEXPRESS, SQLEXPRESS01)
- Installing a new SQL Server instance (default or named)
- Cleaning up leftover services, directories, or registry entries from failed uninstalls
- Any SQL Server `setup.exe` operation that requires elevation

## Workflow Pattern

### 1. Discovery — Find the setup bootstrapper

```bash
find '/c/Program Files/Microsoft SQL Server/' -name 'setup.exe' -path '*/Setup Bootstrap/*'
```

Common paths:
- SQL 2025: `C:\Program Files\Microsoft SQL Server\170\Setup Bootstrap\SQL2025\setup.exe`
- SQL 2017: `C:\Program Files\Microsoft SQL Server\140\Setup Bootstrap\SQL2017\setup.exe`

### 2. Diagnose current state

```bash
# Check Windows services (won't show LocalDB — it's not a service)
sc query state= all | grep -i 'mssql'

# Check LocalDB instances (separate system, no admin needed)
sqllocaldb info

# Check registry instances (may include orphans with no service)
powershell -Command "Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL' -ErrorAction SilentlyContinue"

# Check directories
ls -d '/c/Program Files/Microsoft SQL Server/MSSQL'*
```

### 3. Create the .bat script

Always include `chcp 65001 >nul` as the first line to fix encoding on Spanish Windows.

Template: `references/cleanup-install-template.bat`

### 4. Tell the user to execute it

> **Right-click** on the `.bat` file → **"Run as Administrator"** → accept UAC.

Never attempt to run elevated commands directly from Hermes terminal — `Start-Process -Verb RunAs` opens a UAC prompt per command, which is unusable for multi-step operations.

## SQL Server setup.exe Commands

### Uninstall a named instance

```batch
setup.exe /ACTION=Uninstall /INSTANCENAME=SQLEXPRESS /Q /IACCEPTSQLSERVERLICENSETERMS
```

### Install default instance (MSSQLSERVER)

SQL Server 2022+ (v16+) and 2025 (v17) use `/SAPWD` (not `/SAPASSWORD`). `/SQLSVCACCOUNT` and `/SQLSVCPASSWORD` are now required.

```batch
setup.exe /ACTION=Install ^
  /INSTANCENAME=MSSQLSERVER ^
  /FEATURES=SQLEngine ^
  /SQLSVCACCOUNT="NT Service\MSSQLSERVER" ^
  /SQLSVCPASSWORD="" ^
  /SQLSYSADMINACCOUNTS="BUILTIN\Administradores" ^
  /SECURITYMODE=SQL ^
  /SAPWD="<password>" ^
  /TCPENABLED=1 ^
  /IACCEPTSQLSERVERLICENSETERMS ^
  /Q
```

### Install a named instance (e.g., SQLEXPRESS01)

```batch
setup.exe /ACTION=Install ^
  /INSTANCENAME=SQLEXPRESS01 ^
  /FEATURES=SQLEngine ^
  /SQLSVCACCOUNT="NT Service\MSSQL$SQLEXPRESS01" ^
  /SQLSVCPASSWORD="" ^
  /SQLSYSADMINACCOUNTS="BUILTIN\Administradores" ^
  /SECURITYMODE=SQL ^
  /SAPWD="<password>" ^
  /TCPENABLED=1 ^
  /IACCEPTSQLSERVERLICENSETERMS ^
  /Q
```

## Service Cleanup

After `setup.exe /ACTION=Uninstall` sometimes services and directories persist. Force-clean with:

```batch
sc stop "MSSQL$SQLEXPRESS"
sc delete "MSSQL$SQLEXPRESS"
rmdir /s /q "C:\Program Files\Microsoft SQL Server\MSSQL17.SQLEXPRESS"
reg delete "HKLM\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL17.SQLEXPRESS" /f
```

Also look for related services: `SQLAgent$INSTANCE`, `SQLBrowser`, `SQLWriter`.

## LocalDB Management

LocalDB is a lightweight SQL Server Express edition that runs on-demand (not as a Windows service). It can coexist with full SQL Server instances and survives even when named instance services are deleted.

### Diagnosis

```bash
# List all LocalDB instances
sqllocaldb info

# Detail on a specific instance
sqllocaldb info MSSQLLocalDB

# Check version (matches SQL Server version, e.g. 17.x = SQL 2025)
sqllocaldb versions
```

### Connection strings

| Instance type | Connection string |
|---|---|
| Default LocalDB | `(localdb)\MSSQLLocalDB` |
| Named LocalDB | `(localdb)\NombreInstancia` |
| Auto-named (VS) | `DESKTOP-XXX\LOCALDB#XXXXXXXX` (no usar — ver pitfalls) |

### Commands (no admin required)

```bash
sqllocaldb start MSSQLLocalDB     # Start instance (auto-starts on connect anyway)
sqllocaldb stop MSSQLLocalDB      # Stop instance
sqllocaldb create MiInstancia     # Create a named LocalDB instance
sqllocaldb delete MiInstancia     # Delete it
```

### When to offer LocalDB

- The user's named instance services (SQLEXPRESS, SQLEXPRESS01) are gone but they need a working SQL Server **now**.
- As a quick fallback: `(localdb)\MSSQLLocalDB` works with zero setup.
- Creating a named LocalDB (`sqllocaldb create`) gives a clean, short name without reinstalling.

## Pitfalls

1. **`/SAPASSWORD` not recognized (SQL Server 2022+)**: SQL Server 2022 (v16) and 2025 (v17) deprecated and removed `/SAPASSWORD`. Use `/SAPWD` instead. Error: `The setting 'SAPASSWORD' specified is not recognized.` → Fix: replace with `/SAPWD`.
2. **`/SQLSVCACCOUNT` and `/SQLSVCPASSWORD` are now required**: Starting with SQL Server 2022, these parameters are mandatory even for virtual accounts. Use `"NT Service\MSSQLSERVER"` (default instance) or `"NT Service\MSSQL$<INSTANCE>"` (named) with `/SQLSVCPASSWORD=""`.
3. **Destructive cleanup — ASK FIRST**: The force-cleanup pattern (`sc delete`, `rmdir /s /q`, `reg delete`) permanently deletes working instances. Always confirm with the user before running a script that nukes existing SQL Server instances. If the user just wants to use their existing instance, don't suggest cleanup at all.
4. **`reg delete … 2>nul` may silently fail**: On permission-restricted systems, `reg delete` can fail silently when stderr is redirected to nul, leaving orphaned registry entries even though the script reports `[OK]`. Verify with `reg query` after cleanup.
5. **"No features were uninstalled" error**: This is normal if the instance is already partially removed. Follow up with manual service/registry/directory cleanup.
6. **Encoding garbled output**: Always start `.bat` files with `chcp 65001 >nul`. Without this, Spanish characters render as garbled text (e.g., `⚠` → `âš `).
7. **PowerShell variable escaping in batch**: When calling PowerShell from `.bat` inside setup arguments, double the `$` signs (e.g., `MSSQL`$SQLEXPRESS`).
8. **Spanish Windows group name**: Use `BUILTIN\\Administradores` on Spanish Windows, not `BUILTIN\\Administrators`.
9. **UAC per-command with Start-Process**: Do NOT use `Start-Process -Verb RunAs` for multi-step operations from Hermes terminal. Each call spawns its own UAC dialog. Create a single `.bat` instead.
10. **Setup.exe exit codes**: A non-zero exit code from `/ACTION=Uninstall` does not necessarily mean failure — it may mean the instance was already partially removed. Always verify with `sc query` afterward.
11. **Orphaned registry entries**: Instances can appear in `HKLM\SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL` but have no corresponding Windows service (`sc query` returns ERROR 1060). This happens after incomplete uninstalls or manual service deletion. The registry entry alone does NOT mean the instance works. Always cross-check with `sc query`.
12. **Auto-named LocalDB instances from Visual Studio**: Visual Studio updates can create LocalDB instances with names like `DESKTOP-71B8376\LOCALDB#CA4E9BFC`. These are automatic, non-renamable, and confuse users. When the user reports a "weird name", check `sqllocaldb info` first. Guide them to `(localdb)\MSSQLLocalDB` or create a clean named instance with `sqllocaldb create`.
13. **LocalDB fallback when named instances are dead**: When `sc query` shows ERROR 1060 for all MSSQL$ services, DO NOT jump straight to reinstalling. Check `sqllocaldb info` — LocalDB often survives and gives the user an immediate working SQL Server at `(localdb)\MSSQLLocalDB`. Offer this first before suggesting a full setup.exe reinstall.

## Verification Checklist

- [ ] `sc query MSSQLSERVER` shows RUNNING
- [ ] Can connect with `sqlcmd -S localhost -U sa -P <password>`
- [ ] No leftover services: `sc query state= all | grep -i mssql` shows only expected instances
- [ ] No leftover registry entries under `HKLM\SOFTWARE\Microsoft\Microsoft SQL Server`
