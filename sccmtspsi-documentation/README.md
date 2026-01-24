
# SCCMTSPSI — SCCM Task Sequence Deployment Orchestrator

SCCMTSPSI is a WinPE-based Task Sequence Pre‑Start Interface that orchestrates Microsoft Endpoint Configuration Manager (SCCM/MECM) operating system deployments. It automates device naming, app/collection/AD group selection, BitLocker unlock, USMT migration, disk setup, and more—reducing custom scripting while improving SOE build consistency and success. 

> **Docs source**: Converted from the page “Documentation for SCCM task sequence deployment orchestrator – SCCMTSPSI”.

---

## Table of Contents
- [Features](#features)
- [Concepts: Realms](#concepts-realms)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
  - [Active Directory & SCCM objects](#active-directory--sccm-objects)
  - [Configuration directory layout](#configuration-directory-layout)
- [Configuration (sccmtspsi.config)](#configuration-sccmtspsiconfig)
- [WinPE Boot Image Setup](#winpe-boot-image-setup)
- [Task Sequence Integration](#task-sequence-integration)
  - [sccmtspsi.exe (boot image command)](#sccmtspsiexe-boot-image-command)
  - [sccmtspsi-tasksequence.exe switches](#sccmtspsi-tasksequenceexe-switches)
- [Operator Workflow (WinPE UI)](#operator-workflow-winpe-ui)
- [User State Migration (USMT)](#user-state-migration-usmt)
- [Disk & Partitioning](#disk--partitioning)
- [Key Task Sequence Variables](#key-task-sequence-variables)
- [Notifications](#notifications)
- [Troubleshooting & Error Handling](#troubleshooting--error-handling)
- [Security & Permissions](#security--permissions)
- [Credits & Resources](#credits--resources)

---

## Features
- UI-driven orchestration for SCCM OSD: select Task Sequence, OS image/package, Office suites, SCCM apps, collections, and AD groups per **Realm**. 
- Automates hostname rules (case, prefix/suffix), conflict checks (DNS/SCCM), and optional overrides.
- Integrated BitLocker unlock (AD, MBAM, key/password) and decryption wait options.
- Built-in USMT capture/restore (Hardlink/USB/Network) with encryption, rules, and domain/user move support.
- Disk partitioning for UEFI/Legacy with optional data volumes and recovery partition.
- 18 **Extension Attributes** (custom variables) to branch Task Sequence logic.
- Content availability checks, operator email notifications, final build reports, and robust logging.

## Concepts: Realms
A **Realm** is a 3-character scope (e.g., `r01`, `xyz`) that isolates configuration and visibility: each Realm has its own security group, broker account, deployment/limiting collections, admin categories, AD parent group, staging OU, config directory, and WinPE instance(s).

## Prerequisites
- Microsoft Endpoint Configuration Manager (SCCM/MECM) with PXE/Media support.
- Active Directory with **Active Directory Web Services** (Windows Server 2008 R2+).
- Windows ADK + WinPE Addon (Deployment Tools, USMT, WinPE).
- .NET Framework 4.5+ (for the Configuration Tool).
- SMTP server (for notifications) and optional MBAM server (for BitLocker recovery).

## Installation & Setup
1. **Download components** and obtain the `free.token` license file.
2. **Create a customized WinPE** image using the SCCMTSPSI WinPE customizer (ADK required).
3. **Create your Realm** (3 chars) and the required AD/SCCM objects (see below).
4. **Create `sccmtspsi.config`** with the Configuration Tool (enter the Realm secret key).
5. **Place files** in the configuration directory structure and set NTFS/Share permissions.
6. **Import the customized WinPE** into SCCM and assign it to your Task Sequences.

### Active Directory & SCCM objects
Create these per **Realm** (`XXX` = Realm name):
- **AD global security group**: `sccmtspsi-users-XXX` – controls access and grants read permission to the config share.
- **AD broker account**: `sccmtspsi-broker-XXX` – service identity for Realm operations; add it to `sccmtspsi-users-XXX`.
- **SCCM deployment collection**: `sccmtspsi-deployments-XXX` – target for Realm task sequence deployments (enable incremental updates).
- **SCCM administrative categories**: `sccmtspsi-applications-XXX`, `sccmtspsi-officesuites-XXX` – tag apps/Office suites for the Realm.
- **SCCM limiting collection**: `sccmtspsi-collections-XXX` – parent limiting collection for Realm collections; include the deployment collection and a top-level collection.
- **AD parent group**: `sccmtspsi-groups-XXX` – parent for Realm AD groups.
- **AD staging OU** – computer staging (DistinguishedName path, e.g., `ldap://OU=Staging,...`).

### Configuration directory layout
Create a share (e.g., `\\SCCM-MP\sccmtspsi`) and inside it a folder per Realm with strict ACLs:
```
sccmtspsi\
  └─ r01\
      ├─ token\        (place license: free.token)
      ├─ usmt\         (USMT XMLs: MigUser/MigDocs/MigApps/Config.xml)
      ├─ patch\        (optional customizations)
      └─ sccmtspsi.config
```
- Share & NTFS: **Read** for `sccmtspsi-users-XXX`; Realm subfolders break inheritance; **Read** for broker & admins only.

## Configuration (sccmtspsi.config)
Key settings managed by the Configuration Tool:
- **Realm secret key** (must match `sccmtspsi-broker-XXX` password).
- **Allowed WinPE instances** (`-instancename` values) to control USB/Full media usage.
- **Network access account** (for log copy & network USMT) and **Notification account** (SMTP auth).
- **Hostname rules** (case, prefix, suffix), **auto-identification** and **conflict checks** (DNS/SCCM) with `True/False/Truebutallow!`.
- **MBAM/SMTP** server details; **logs** & **profiles** locations; **hardware model allow‑list**.

## WinPE Boot Image Setup
- Use the WinPE customizer (ADK/WinPE Addon) to generate AMD64/x86 images and import into SCCM.
- Assign the sccmtspsi WinPE to at least one Task Sequence (PXE/Media). For PXE, deploy a TS (can be empty) to **ALL UNKNOWN COMPUTERS** with **Available** + **Only media and PXE (hidden)**.
- Configure the boot image **command line** to run `sccmtspsi.exe` with **all** of these parameters on one line:

```bash
sccmtspsi.exe \
  -realmname r01 \
  -instancename yourinstancename \
  -logindomain yourdomain.com \
  -sccmserver sccmserver.yourdomain.com \
  -adservername yourdc.yourdomain.com \
  -adconnectiontype adws \
  -sccmsitecode S01 \
  -timeserver timeserver.yourdomain.com \
  -timezone "New Zealand Standard Time" \
  -dnsserverip 10.0.0.10
```

## Task Sequence Integration
After installing `sccmtspsicomponent.msi`, copy **the correct‑arch** `sccmtspsi-tasksequence.exe` into a new SCCM package **and** into the **USMT package root**. Then add the following steps:

### sccmtspsi.exe (boot image command)
Runs **inside WinPE** to present the operator UI and capture selections/variables.

### sccmtspsi-tasksequence.exe switches
Add this executable at key points:
- **First step**: `sccmtspsi-tasksequence.exe -start` – initializes, disables BitLocker if needed, restarts to WinPE, opens the UI.
- **USMT restore**: `sccmtspsi-tasksequence.exe -usmt` – restores captured user state.
- **Move to AD OU**: `sccmtspsi-tasksequence.exe -movetoadou "OU=Staging,OU=Devices,DC=example,DC=local"`
- **End of sequence**: `sccmtspsi-tasksequence.exe -end` – sends build documentation & completion report; also finalizes drive letters.
- **(Optional)** run **without switches** before/after restarts to gather/copy logs.

## Operator Workflow (WinPE UI)
- **Sign‑in**: Only members of `sccmtspsi-users-XXX` may log in (domain time sync required).
- **Network**: Enable/disable NICs, set IP/gateway/DNS, choose connection type.
- **Hostname**: Enforce case/prefix/suffix; auto‑detect; allow `!` override if permitted.
- **BitLocker**: Unlock via recovery file, AD, MBAM, or password; optional “Turn off BitLocker”.
- **Task Sequence**: List TS deployed to `sccmtspsi-deployments-XXX`; select **OS image/package**.
- **Office & Apps**: Choose Office (`sccmtspsi-officesuites-XXX`) and SCCM apps (`sccmtspsi-applications-XXX`), or load **profiles** (`.sccmapps`).
- **Collections & AD groups**: Choose collections (limited by `sccmtspsi-collections-XXX`) and AD groups (under `sccmtspsi-groups-XXX`); profiles supported (`.sccmcols`).
- **USMT**: Pick **Hardlink/USB/Network**, set rules, encryption, and optional domain/user moves.
- **Disk**: Configure UEFI/Legacy partitioning, OS label, up to 2 data volumes.
- **Actions**: Choose end‑state actions (Install Windows, capture/restore combos, Decommission, Turn off BitLocker, etc.).
- **Primary users**: Pull from SCCM and assign (UDA).
- **Extension Attributes (1–18)**: Set custom variables to route TS logic; load **presets** or from `.extatts` (18 lines).

## User State Migration (USMT)
- **Types**: Hardlink (in‑place), USB (by label), Network (share); optional compression.
- **Encryption**: AES/AES_128/AES_192/AES_256 or 3DES/3DES_112 with 8–20 char key.
- **Rules**: Use `MigUser.xml`, `MigDocs.xml`, `MigApps.xml` (plus custom) stored under `usmt`.
- **Config.xml**: Optional (generated via `/genconfig`) to exclude OS components or tune behavior.
- **Advanced**: Ignore non‑fatal return codes; handle EFS (Abort/Skip/Decryptcopy/Copyraw/Hard‑link); support domain/user moves; skip corrupt profiles.
- **Paths/Perms**: Configure network log/profiles locations; grant write to broker + network account.

## Disk & Partitioning
- Automatic **UEFI** (EFI + MSR) or **Legacy** (MBR/NTFS) layouts; optional **Recovery** (set 0 MB to skip).
- Create up to **2 data volumes** on the selected disk; set **OS volume label**.
- Final drive letters are applied by `sccmtspsi-tasksequence.exe -end`.

## Key Task Sequence Variables
> _SCCMTSPSI sets many variables; below are commonly used examples._

**Selection & IDs**
- `SMSTSPSISELECTEDTASKSEQUENCENAME`, `SCCMTSPSISELECTEDTASKSEQUENCEPACKAGEID`, `SMSTSPreferredAdvertID` (TS selection)
- `SCCMTSPSISELECTEDOPERATINGSYSTEMNAME`, `SCCMTSPSISELECTEDOPERATINGSYSTEMPACKAGEID` (OS)
- `SCCMTSPSISELECTEDOFFICESUITENAME`, `SCCMTSPSISELECTEDOFFICESUITEPACKAGEID` (Office)
- `SCCMTSPSISET{1|2|3}APP01…99` and flags like `SCCMTSPSI-APP-PACKAGEID-<ID>-ISSET` (Applications)
- `SCCMTSPSICOLNAME1…999`, `SCCMTSPSICOLID1…999` (Collections)

**USMT**
- `SCCMTSPSIUSMTXMLRULEFILES`, `SCCMTSPSIUSMTINCLUDEDANDEXCLUDEDUSERS`, `SCCMTSPSIUSMTENCRYPTIONALGORITHM`, `SCCMTSPSIUSMTEFSHANDLE`
- `SCCMTSPSIUSMTCOMPRESSUSBDATASTORE`, `SCCMTSPSIUSMTCOMPRESSNETWORKDATASTORE`, `SCCMTSPSIUSMTNETWORKDATASTORE`, `OSDStateStorePath`, `OSDMigrateConfigFiles`, `OSDMigrateAdditionalRestoreOptions`

**System/Infra**
- `SCCMTSPSISCCMSERVER`, `SCCMTSPSISCCMSITECODE`, `SMSTSMP`
- `SCCMTSPSIWINPEARCHITECTURE`, `SCCMTSPSIPROCESSORARCHITECTURE`, `SCCMTSPSIPCSYSTEMTYPE`, `SCCMTSPSISYSTEMMODEL`, `SCCMTSPSISYSTEMMANUFACTURER`
- `SCCMTSPSIREALMNAME`, `SCCMTSPSILOGINDOMAIN`
- `SCCMTSPSISMTPSERVER`, `SCCMTSPSISMTPSERVERPORT`, `SCCMTSPSISMTPSERVERSSLENABLED`

**Device/OS**
- `SCCMTSPSIASSETNAME` / `OSDComputerName` (hostname)
- `SCCMTSPSIOPERATINGSYSTEMVOLUMELABEL`, `SCCMTSPSIOPERATINGSYSTEMDRIVE`
- `SCCMTSPSIDATADRIVE1/2`, `SCCMTSPSIDATADRIVEVOLUMELABEL1/2`

**Routing/Actions & UDA**
- `SMSTSPSIACTIONS` (capture/restore/Install/Decommission/Turn off BitLocker)
- `SCCMTSPSIACTIONS` (AD/SCCM keep/remove combinations)
- `SMSTSUdaUsers` (primary users)
- `SCCMTSPSIEXTENSIONATTRIBUTE1…18` (custom routing variables)

## Notifications
Configure **Login**, **Failure**, **Success**, and **Final Build Report** emails. Success notices include operator, architecture, runtime, system, Realm, domain, selected items, partitions, and the full set of variables.

## Troubleshooting & Error Handling
- **Content checks** ensure required TS/OS/App content exists on the client DPs before proceeding.
- **Collection/AD group add** errors can be continued or set to fail the build (permissions/limiting collection issues are common causes).
- **BitLocker decryption wait** modes: `True`, `Ask`, or `False`.
- **Approved hardware**: optionally restrict builds to an allow‑list of device models (WMI `Win32_ComputerSystem.Model`).
- Use `sccmtspsi-tasksequence.exe` (no switches) anywhere to gather/copy logs.

## Security & Permissions
- Keep **`sccmtspsi-users-XXX`** flat (no nested/foreign principals) for performance.
- Realm **broker account** needs scoped rights to add/move computer objects in AD, manage SCCM collection membership, read token/usmt/patch, write to logs, and MBAM recovery access.
- Realm **secret key** protects config access; update when the broker password is rotated.

## Credits & Resources
- **Official docs**: <https://sccmtspsi.com/sccmtspsi-documentation/>
- **Components download**: <https://sccmtspsi.com/product/sccmtspsi-components/>
- **Video tutorial**: YouTube (2020‑04‑09)

---

> This README was generated by converting the vendor documentation into a GitHub‑friendly format. Review and tailor environment‑specific names, paths, and examples before publishing.
