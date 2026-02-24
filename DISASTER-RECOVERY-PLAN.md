# Backup & Disaster Recovery Plan

## Dental Practice — Confidential

| Field               | Value                          |
|---------------------|--------------------------------|
| **Document Owner**  | [Practice Owner Name]          |
| **Version**         | 1.1                            |
| **Created**         | 2026-02-24                     |
| **Last Reviewed**   | 2026-02-24                     |
| **Next Review Due** | 2026-08-24 (6-month cycle)     |
| **Classification**  | CONFIDENTIAL — Contains PHI references |

> **Purpose:** This document exists so that *any* competent IT professional can
> walk into this practice cold and restore full operations after a disaster. It
> must never live solely in one person's head.

---

## Table of Contents

1. [Current Environment Inventory](#1-current-environment-inventory)
2. [Current Backup Architecture](#2-current-backup-architecture)
3. [Audit Findings & Gap Analysis](#3-audit-findings--gap-analysis)
4. [HIPAA Compliance Requirements](#4-hipaa-compliance-requirements)
5. [Recommended Target Architecture](#5-recommended-target-architecture)
6. [Recovery Objectives](#6-recovery-objectives)
7. [Disaster Recovery Procedures](#7-disaster-recovery-procedures)
8. [Backup Verification & Testing Schedule](#8-backup-verification--testing-schedule)
9. [Contact List & Vendor Information](#9-contact-list--vendor-information)
10. [Appendices](#10-appendices)

---

## 1. Current Environment Inventory

### 1.1 Physical Hardware

| Component          | Details                                       |
|--------------------|-----------------------------------------------|
| **Physical Server** | [Make/Model — FILL IN]                       |
| **Hypervisor**     | VMware vSphere (ESXi) — [version — FILL IN]  |
| **Virtual Machine** | 1x Windows Server 2022 VM                    |
| **UPS**            | [Make/Model — FILL IN, or "NONE"]             |
| **Network Switch** | [Make/Model — FILL IN]                        |
| **Firewall/Router**| [Make/Model — FILL IN]                        |
| **Workstations**   | [Count and OS — FILL IN]                      |

### 1.2 Software Running on the VM

| Application                   | Purpose                        | Data Location on VM                     |
|-------------------------------|--------------------------------|-----------------------------------------|
| **Open Dental**               | Practice management / EHR      | MySQL/MariaDB database (default: `C:\mysql\data\opendental\`) + `OpenDentImages\` (A-Z folder) |
| **CareStream CS Imaging 8**   | 2D dental imaging              | CS Imaging database (embedded SQL Server) + image files |
| **Vatech Ez3D-i / EzDent-i**  | 3D CBCT imaging                | EzServer database (`C:\Program files (x86)\Vatech\Common\FM`) + DICOM image store + capture programs (`C:\VCaptureSW\`) |

### 1.3 Data Classification

All patient data on this server is **electronic Protected Health Information (ePHI)** under HIPAA. This includes:

- Patient demographics, insurance, treatment plans (Open Dental)
- 2D radiographs, intraoral photos, panoramic images (CareStream)
- 3D CBCT volumes (Vatech) — these are large DICOM datasets, often hundreds of MB per scan
- Financial records, clinical notes, prescriptions

### 1.4 Estimated Data Volumes

| Dataset                        | Estimated Size  | Growth Rate              |
|--------------------------------|-----------------|--------------------------|
| Open Dental MySQL database     | [FILL IN] GB    | ~[FILL IN] GB/year       |
| Open Dental Images (A-Z folder)| [FILL IN] GB    | ~[FILL IN] GB/year       |
| CareStream image store         | [FILL IN] GB    | ~[FILL IN] GB/year       |
| Vatech CBCT data               | [FILL IN] GB    | ~[FILL IN] GB/year (large — each CBCT scan can be 200-500 MB) |
| Windows Server OS + apps       | ~60-80 GB       | Minimal                  |
| **Total VM disk**              | [FILL IN] TB    |                          |

> **Action required:** Fill in the above values. Use Windows Disk Management or
> `dir /s` on each data folder to get current sizes. These numbers are critical
> for sizing backup storage and estimating replication times.

---

## 2. Current Backup Architecture

### 2.1 What Is Currently in Place

```
┌─────────────────┐        ┌───────────────────────┐        ┌───────────────────────┐
│  Physical Server │        │  Synology DS920+      │        │  Synology (Off-site)  │
│  (VMware ESXi)   │───────▶│  (On-site NAS)        │───────▶│  (Owner's Home)       │
│                  │  ABB   │  Active Backup for    │ Hyper  │  Replication target   │
│  1x Win 2022 VM  │  Agent │  Business (Server)    │ Backup │                       │
└─────────────────┘        └───────────────────────┘        └───────────────────────┘
```

| Layer             | Technology                              | Schedule      | Retention  |
|-------------------|-----------------------------------------|---------------|------------|
| **On-site backup**| Synology Active Backup for Business     | [FILL IN]     | [FILL IN]  |
| **Off-site copy** | Synology Hyper Backup or Snapshot Replication to home NAS | [FILL IN] | [FILL IN] |

### 2.2 What Is Being Backed Up

- [x] Full VM image (via Active Backup for Business — server-level backup)
- [ ] Application-level database dumps (Open Dental MySQL)
- [ ] Application-level database dumps (CareStream embedded SQL Server)
- [ ] Vatech EzServer database export
- [ ] vSphere/ESXi configuration export
- [ ] Network equipment configuration
- [ ] Backup of backup configuration itself

---

## 3. Audit Findings & Gap Analysis

### CRITICAL FINDINGS

| # | Finding | Risk Level | Description |
|---|---------|------------|-------------|
| 1 | **No application-level database backups** | **CRITICAL** | Active Backup for Business captures the VM disk image, but does not perform application-consistent database dumps. If the MySQL database (Open Dental) or SQL Server database (CareStream) is mid-transaction during a snapshot, the restored database may be corrupt or missing recent data. A VM-level snapshot is not a substitute for a proper database backup. |
| 2 | **Single point of failure — one physical server** | **CRITICAL** | If the physical server hardware fails (motherboard, RAID controller, etc.), the practice is down until replacement hardware is procured and the VM is restored. There is no warm standby or failover. |
| 3 | **Unknown backup testing status** | **CRITICAL** | There is no documented evidence that backups have been tested by performing a full restore. An untested backup is not a backup — it is a hope. |
| 4 | **Disaster recovery plan exists only in owner's head** | **CRITICAL** | If the practice owner is unavailable during a disaster, nobody else knows how to restore operations. |
| 5 | **Off-site NAS at owner's home — physical security and HIPAA concerns** | **HIGH** | The off-site Synology at a private residence must meet HIPAA physical safeguard requirements. Is it in a locked room? Is the data encrypted at rest? Who else has physical access to the home? Is there a BAA in place if a third party manages it? |

### HIGH-PRIORITY FINDINGS

| # | Finding | Risk Level | Description |
|---|---------|------------|-------------|
| 6 | **No encryption verification** | **HIGH** | It is unconfirmed whether (a) the on-site Synology shared folders are AES-256 encrypted, (b) the replication to the off-site NAS uses encrypted transfer (TLS), and (c) the off-site NAS shared folders are encrypted at rest. HIPAA requires encryption of ePHI at rest and in transit. |
| 7 | **No ransomware protection (immutable backups)** | **HIGH** | If ransomware compromises the server, it may also reach the on-site Synology via network shares or credentials. There are no immutable snapshots or air-gapped backups documented. |
| 8 | **ESXi host not backed up** | **HIGH** | If the ESXi host configuration is lost, the VM backup alone is not enough — you need to reinstall and reconfigure ESXi before you can restore the VM. ESXi host configuration should be exported regularly. |
| 9 | **No UPS shutdown integration confirmed** | **MEDIUM** | CareStream recommends a UPS with automatic shutdown software. An unclean shutdown during imaging acquisition can corrupt data. |
| 10 | **Retention policy undocumented** | **MEDIUM** | HIPAA requires 6-year retention of compliance documentation. State law may require 7-10+ years for medical records. Current retention settings are unknown. |
| 11 | **No backup monitoring/alerting** | **MEDIUM** | If a backup job fails silently, it could go unnoticed for days or weeks. There should be email or push notifications for every backup job status. |
| 12 | **Synology DS920+ hardware is also a single point of failure** | **MEDIUM** | If the on-site NAS dies, you lose your local rapid-restore capability and are dependent on the off-site NAS which will be much slower to restore from. |

---

## 4. HIPAA Compliance Requirements

The following HIPAA Security Rule requirements apply directly to this backup and DR plan. Reference: **45 CFR § 164.308(a)(7) — Contingency Plan**.

### 4.1 Required Implementations

| HIPAA Requirement | CFR Reference | Current Status | Action Needed |
|-------------------|---------------|----------------|---------------|
| Data Backup Plan | §164.308(a)(7)(ii)(A) | PARTIAL — VM backup exists but no application-level backups | Add application-level database backups |
| Disaster Recovery Plan | §164.308(a)(7)(ii)(B) | MISSING — not documented | This document |
| Emergency Mode Operation Plan | §164.308(a)(7)(ii)(C) | MISSING | Section 7.5 below |
| Testing & Revision Procedures | §164.308(a)(7)(ii)(D) | MISSING | Section 8 below |
| Application & Data Criticality Analysis | §164.308(a)(7)(ii)(E) | MISSING | Section 6 below |
| Encryption of ePHI at rest | §164.312(a)(2)(iv) | UNVERIFIED | Verify Synology encryption settings |
| Encryption of ePHI in transit | §164.312(e)(2)(ii) | UNVERIFIED | Verify replication uses TLS |
| Access controls | §164.312(a)(1) | UNVERIFIED | Verify NAS user accounts and permissions |
| Audit controls | §164.312(b) | UNVERIFIED | Enable and review Synology access logs |
| Integrity controls | §164.312(c)(1) | PARTIAL — Btrfs checksums on Synology | Verify Btrfs is in use; enable data scrubbing |

### 4.2 Business Associate Agreements (BAAs)

Any third party with access to ePHI must have a BAA in place. Review and document:

| Vendor/Service | BAA in Place? | Date Signed | Notes |
|----------------|---------------|-------------|-------|
| Synology (if using C2 cloud) | [FILL IN] | [FILL IN] | **Synology only offers BAAs for their C2 cloud services** (C2 Object Storage, C2 Backup, C2 Transfer, C2 Password). The on-premise DS920+ itself does not require a BAA since you self-manage it, but if you add C2 cloud as a third backup tier, you MUST execute a BAA with Synology. Contact C2 support to request one. |
| Off-site backup host (home) | N/A — self-managed | N/A | Owner is the covered entity |
| IT support provider | [FILL IN] | [FILL IN] | Required for any IT vendor with ePHI access |
| Open Dental (if using their cloud services) | [FILL IN] | [FILL IN] | |
| CareStream | [FILL IN] | [FILL IN] | |
| Vatech | [FILL IN] | [FILL IN] | |

### 4.3 Data Retention Requirements

| Data Type | Minimum Retention | Authority |
|-----------|-------------------|-----------|
| HIPAA compliance documentation (policies, risk assessments, BAAs, audit logs) | **6 years** | 45 CFR §164.316(b)(2)(i) |
| Patient medical/dental records | **[FILL IN — check your state law]** | State law (commonly 7-10 years; longer for minors) |
| Financial records | **[FILL IN]** | State/federal tax law |

---

## 5. Recommended Target Architecture

### 5.1 The 3-2-1-1 Backup Rule

The industry standard for healthcare is the **3-2-1-1 rule**:

- **3** copies of your data (production + 2 backups)
- **2** different storage media types
- **1** copy off-site
- **1** copy that is immutable or air-gapped (ransomware protection)

### 5.2 Proposed Architecture

```
┌──────────────────────┐
│  PHYSICAL SERVER     │
│  VMware ESXi         │
│  ┌────────────────┐  │
│  │ Windows Server │  │
│  │ 2022 VM        │  │
│  │                │  │
│  │ Nightly DB     │  │
│  │ dump scripts ──│──│──┐   (1) Application-consistent DB dumps
│  └────────────────┘  │  │       run INSIDE the VM nightly
└──────────┬───────────┘  │
           │              │
     (2) VM-level         │
     image backup         │
     via ABB              │
           │              │
           ▼              ▼
┌──────────────────────────────┐
│  SYNOLOGY DS920+ (ON-SITE)   │
│                              │
│  • Active Backup for Business│
│    (VM image backups)        │
│  • Database dump target share│
│  • Btrfs immutable snapshots │
│  • AES-256 shared folder     │
│    encryption ENABLED        │
│  • Email alerts configured   │
│  • Snapshot replication ON   │
│                              │
└──────────┬───────────────────┘
           │
     (3) Encrypted replication
     via Hyper Backup (TLS)
     or Snapshot Replication
           │
           ▼
┌──────────────────────────────┐
│  SYNOLOGY (OFF-SITE — HOME)  │
│                              │
│  • Hyper Backup target       │
│  • AES-256 encryption at rest│
│  • In a locked, secured room │
│  • UPS-protected             │
│  • Firewall — no inbound     │
│    except Synology VPN       │
│                              │
└──────────────────────────────┘

     (4) OPTIONAL but recommended:
         Synology C2 cloud backup
         (with BAA) for true
         geographic separation
         and immutability
```

### 5.3 Recommended Changes Summary

| Priority | Change | Why |
|----------|--------|-----|
| **P0 — Do immediately** | Add nightly application-level database dump scripts inside the VM (see Appendix A) | Ensures application-consistent, restorable database backups for Open Dental (MySQL), CareStream (SQL Server), and Vatech (EzServer) |
| **P0 — Do immediately** | Verify and enable AES-256 encryption on BOTH Synology NAS shared folders | HIPAA encryption at rest |
| **P0 — Do immediately** | Verify replication to off-site NAS uses encrypted transport (TLS/VPN) | HIPAA encryption in transit |
| **P0 — Do immediately** | Perform a full test restore of the VM to a test environment | Validate that your backups actually work |
| **P1 — Do this month** | Enable Synology Btrfs immutable snapshots with a retention lock | Ransomware protection — snapshots cannot be deleted or modified |
| **P1 — Do this month** | Configure email/push alerts for all backup jobs (success AND failure) | Catch silent failures |
| **P1 — Do this month** | Export and back up the ESXi host configuration weekly | Enables faster bare-metal rebuild |
| **P1 — Do this month** | Secure the off-site NAS (locked room, UPS, firewall rules, strong passwords) | HIPAA physical safeguards |
| **P2 — Do this quarter** | Consider Synology C2 Backup (with BAA) as a third backup target | True geographic diversity + immutable cloud copy |
| **P2 — Do this quarter** | Document and verify UPS at server + automatic shutdown software | Prevent data corruption from power loss |
| **P2 — Do this quarter** | Review and set retention policies on all backup jobs per state law | Compliance |
| **P3 — Evaluate** | Evaluate a spare/refurbished server or a Synology VMM instant-restore plan as a warm standby | Reduce RTO from hours/days to minutes |

---

## 6. Recovery Objectives

### 6.1 Application & Data Criticality Analysis

| System | Criticality | Impact if Down | RPO Target | RTO Target |
|--------|-------------|----------------|------------|------------|
| **Open Dental** (practice management, scheduling, billing) | **CRITICAL** | Cannot see patients, no access to records, no billing | **1 hour** (hourly incremental backup) | **2 hours** |
| **CareStream CS Imaging 8** (2D imaging) | **CRITICAL** | Cannot take or view X-rays | **24 hours** (nightly backup) | **4 hours** |
| **Vatech Ez3D-i** (3D CBCT) | **HIGH** | Cannot take or view 3D scans; can refer patients out for urgent cases | **24 hours** (nightly backup) | **4 hours** |
| **Windows Server 2022** (OS) | **HIGH** | All applications are unavailable | **24 hours** | **2 hours** |
| **ESXi hypervisor** | **HIGH** | VM cannot run | N/A (config export) | **1-2 hours** (reinstall + import config) |
| **Network infrastructure** | **HIGH** | Nothing works | N/A (config backup) | **30 min** |

### 6.2 RPO / RTO Definitions

- **RPO (Recovery Point Objective):** The maximum amount of data you can afford to lose, measured in time. An RPO of 1 hour means you accept losing up to 1 hour of data entry.
- **RTO (Recovery Time Objective):** The maximum time from disaster declaration to systems being operational again. An RTO of 2 hours means the practice must be functional within 2 hours.

### 6.3 Realistic RTO Scenarios

| Scenario | Restore From | Estimated RTO | Data Loss (RPO) |
|----------|-------------|---------------|-----------------|
| VM corruption, server hardware OK | On-site Synology (ABB restore) | **30-60 min** | Last backup interval |
| Server hardware failure, on-site NAS OK | New/spare server + ESXi install + ABB restore from NAS | **4-8 hours** (depends on hardware procurement) | Last backup interval |
| Office disaster (fire, flood, theft) | Off-site Synology at home → new hardware | **8-24 hours** (depends on hardware procurement + data transfer) | Last replication interval |
| Ransomware — server + NAS compromised | Off-site Synology (if unaffected) or immutable snapshots | **4-12 hours** | Last immutable snapshot |
| Ransomware — everything compromised | Cloud backup (if implemented) or last known-good off-site copy | **12-48 hours** | Last cloud backup |

---

## 7. Disaster Recovery Procedures

> **IMPORTANT:** These are step-by-step runbooks. A person unfamiliar with this
> practice's IT should be able to follow them.

### 7.1 Scenario A — VM Failure (Server Hardware Is Fine)

**Symptoms:** Windows Server VM is unresponsive, BSOD, or corrupt. ESXi host is still running.

**Steps:**

1. **Assess** — Log into the ESXi web console (`https://[ESXi-IP-ADDRESS]`).
   - Credentials: stored in [FILL IN — password manager / sealed envelope location]
   - Check if the VM is powered off, hung, or showing errors.

2. **Attempt restart** — Right-click the VM → Power → Reset. Wait 5 minutes.

3. **If restart fails, restore from backup:**
   a. Log into the Synology DS920+ web interface (`https://[NAS-IP-ADDRESS]:5001`).
      - Credentials: stored in [FILL IN]
   b. Open **Active Backup for Business** → **Virtual Machine** tab.
   c. Select the most recent successful backup of the Windows Server 2022 VM.
   d. Click **Restore** → **Restore to VMware vSphere**.
   e. Select the ESXi host as the restore target.
   f. Choose **"Restore to original location"** and check **"Overwrite existing VM"**.
   g. Start the restore. Monitor progress.
   h. Once complete, power on the VM from the ESXi console.

4. **Verify applications:**
   a. Log into Windows Server (credentials: [FILL IN]).
   b. Open **Open Dental** — verify the database loads and patient data is current.
   c. Open **CareStream CS Imaging 8** — verify images display.
   d. Open **Vatech EzDent-i / Ez3D-i** — verify 3D scans are accessible.
   e. Check the date/time of the most recent patient data to confirm RPO.

5. **If application databases are inconsistent:**
   a. Restore the most recent application-level database dump (see Scenario E below).

6. **Document** — Log the incident: date, time, cause (if known), restore time, data loss (if any).

### 7.2 Scenario B — Physical Server Hardware Failure

**Symptoms:** ESXi host is unreachable. Server will not power on or is displaying hardware errors.

**Steps:**

1. **Assess hardware failure:**
   - Check power, network cables, UPS status.
   - Check for beep codes, LED indicators, or error messages on the server console.
   - If a component (e.g., RAM, disk, PSU) is identifiable as failed, source a replacement.

2. **If server is unrecoverable, procure replacement hardware:**
   - Contact vendor: [FILL IN — vendor name, phone, account number]
   - Minimum specs for replacement: [FILL IN — CPU, RAM, storage requirements to run ESXi + this VM]
   - **Alternatives for faster recovery:**
     - Ask if vendor has same-day or next-business-day delivery/swap option.
     - Consider renting/purchasing a temporary server from a local IT reseller.
     - **Instant Restore option (if configured):** Synology Virtual Machine Manager (VMM) on the DS920+ can boot the VM directly from the backup as a temporary measure. **WARNING:** The DS920+ has limited CPU/RAM; this will be slow but can provide emergency read-only access to patient records.

3. **Install ESXi on new hardware:**
   a. Download the ESXi ISO from VMware (or use a USB installer if one was prepared — stored at [FILL IN]).
   b. Install ESXi onto the new server.
   c. Restore ESXi configuration from the saved backup (see Appendix B for ESXi config export/import).
   d. If no ESXi config backup exists, manually configure: networking (IP: [FILL IN], VLAN: [FILL IN], DNS: [FILL IN]), datastore, and NTP.

4. **Restore the VM from on-site Synology:**
   - Follow Step 3 from Scenario A above, selecting the new ESXi host as the target.

5. **Verify applications** — same as Scenario A, Step 4.

6. **Update this document** with the new server hardware details.

### 7.3 Scenario C — Office Disaster (Fire, Flood, Theft, Total Loss)

**Symptoms:** The office, server, and on-site Synology are all destroyed or inaccessible.

**Steps:**

1. **Ensure personal safety first.** Do not enter a damaged building.

2. **Notify:**
   - Insurance company: [FILL IN]
   - Patients (if extended closure): [FILL IN — phone tree / answering service]
   - HIPAA Breach Assessment: If ePHI may have been exposed (e.g., theft of server/NAS), initiate the HIPAA Breach Notification process (see Section 7.6).

3. **Secure a temporary location with network access.**

4. **Procure replacement server hardware** (same as Scenario B, Step 2).

5. **Retrieve the off-site Synology from home:**
   - Location: [FILL IN — exact address and location within home, e.g., "locked closet in home office"]
   - Transport it to the temporary or rebuilt office location.

6. **Install ESXi on new hardware** (Scenario B, Step 3).

7. **Restore the VM from the off-site Synology:**
   a. Connect the off-site Synology to the new server's network.
   b. Install Active Backup for Business on the off-site Synology (if not already present).
   c. Use ABB or Hyper Backup to restore the VM image to the new ESXi host.
   d. **Note:** If the off-site copy was made via Hyper Backup (not ABB), you may need to extract the VM disk files and manually import them into ESXi. See Appendix C.

8. **Verify applications** — same as Scenario A, Step 4.

9. **Re-establish backups** — reconfigure the backup chain once new permanent hardware is in place.

### 7.4 Scenario D — Ransomware Attack

**Symptoms:** Files encrypted, ransom note displayed, applications inaccessible.

**Steps:**

1. **IMMEDIATELY disconnect the server from the network.** Pull the Ethernet cable. Do NOT shut down the server (forensic evidence may be needed).

2. **Disconnect the on-site Synology from the network.** This is to prevent lateral spread.

3. **Do NOT pay the ransom.** There is no guarantee of data recovery, and payment funds criminal activity.

4. **Assess the scope:**
   - Is only the VM affected, or has the Synology also been compromised?
   - Check the off-site Synology (remotely if possible) — is it still accessible and clean?

5. **If on-site Synology is clean:**
   a. Wipe and reinstall ESXi on the server.
   b. Restore the VM from the most recent clean backup on the Synology.
   c. Before connecting the restored VM to the network, ensure it is fully patched and the attack vector is closed.

6. **If on-site Synology is compromised but immutable snapshots exist:**
   a. Restore from the immutable Btrfs snapshot (these cannot be modified by ransomware).
   b. Follow step 5 above.

7. **If both on-site server and NAS are compromised:**
   a. Use the off-site Synology at home (Scenario C, Steps 5-8).

8. **Post-incident:**
   - Conduct a HIPAA Breach Risk Assessment (Section 7.6).
   - Change ALL passwords (ESXi, Windows, Synology, all application passwords, network equipment).
   - Review firewall rules, disable RDP if exposed to the internet, enable MFA everywhere possible.
   - Report to: FBI IC3 (ic3.gov), your cyber insurance carrier ([FILL IN]), and HHS if breach threshold is met.
   - Document everything for HIPAA compliance records (retain 6 years).

### 7.5 Emergency Mode Operations

If full systems cannot be restored within the RTO, the practice must be able to operate in a degraded mode:

| Function | Emergency Alternative |
|----------|-----------------------|
| Patient scheduling | Paper schedule book (keep a current-week printout each Friday) |
| Patient records access | If Synology VMM instant-restore is available, boot a read-only VM on the NAS for record lookups |
| Taking X-rays | Defer non-urgent imaging; refer urgent cases to [FILL IN — nearby imaging center or partner practice] |
| Billing & insurance claims | Queue on paper; submit when systems restored |
| Prescriptions | Use state PDMP website + manual prescriptions |

> **Action:** Print a current-week patient schedule every Friday and store it in [FILL IN — fire-safe location]. This ensures you know who to call if the office must close.

### 7.6 HIPAA Breach Notification Procedure

If ePHI may have been accessed, acquired, used, or disclosed in an unauthorized manner:

1. **Conduct a Breach Risk Assessment** within 24 hours. Evaluate:
   - Nature and extent of the PHI involved
   - The unauthorized person who used the PHI or to whom it was disclosed
   - Whether the PHI was actually acquired or viewed
   - The extent to which the risk to the PHI has been mitigated

2. **If a breach is confirmed (more than low probability of compromise):**
   - **Individuals:** Notify affected patients without unreasonable delay, no later than 60 days after discovery.
   - **HHS:** If breach affects fewer than 500 individuals, report to HHS OCR annually. If 500+ individuals, report within 60 days.
   - **Media:** If 500+ individuals in a single state/jurisdiction are affected, notify prominent media outlets.

3. **Document everything** — date discovered, assessment results, notifications sent, corrective actions taken. Retain for 6 years.

---

## 8. Backup Verification & Testing Schedule

> **An untested backup is not a backup.**

### 8.1 Routine Verification (Daily)

- [ ] Check Synology Active Backup for Business dashboard — verify last backup completed successfully.
- [ ] Check email alerts — confirm no failure notifications.
- [ ] Verify database dump scripts ran (check log file and dump file timestamps inside the VM).

**Responsible party:** [FILL IN — name or role]

### 8.2 Monthly Testing

- [ ] Perform a **file-level restore test**: Pick a random patient record from the latest backup and restore it to a test location. Verify it opens correctly in the application.
- [ ] Verify **off-site replication** is current: Log into the off-site Synology dashboard and confirm the last replication timestamp.
- [ ] Review Synology storage utilization — project when you will run out of space.
- [ ] Review and rotate backup admin passwords if required.

**Responsible party:** [FILL IN]
**Log results in:** [FILL IN — spreadsheet, binder, or ticketing system]

### 8.3 Quarterly Testing

- [ ] Perform a **full VM restore test** to an isolated environment (e.g., a test ESXi host or Synology VMM). Boot the VM and verify all three applications (Open Dental, CareStream, Vatech) function and data is present.
- [ ] Time the restore and document the actual RTO achieved.
- [ ] Perform a **database-only restore test**: Restore the latest MySQL dump to a test instance. Restore the CareStream SQL Server backup to a test instance. Verify data integrity.
- [ ] Review this entire DR document for accuracy and update any changes (hardware, software versions, contacts, credentials).

**Responsible party:** [FILL IN]
**Log results in:** [FILL IN]

### 8.4 Annual Testing

- [ ] Perform a **full disaster simulation**: Pretend the server is gone. Using only this document and the off-site backup, restore the entire environment on test hardware.
- [ ] Involve a third party (e.g., your IT provider) who was NOT the original implementer, to validate that this document is usable by someone unfamiliar with the environment.
- [ ] Update the HIPAA Risk Assessment to reflect any changes in backup or DR posture.
- [ ] Review data retention compliance with state law requirements.

### 8.5 Testing Log

| Date | Test Type | Performed By | Result | Actual RTO | Data Loss (RPO) | Notes / Issues Found |
|------|-----------|-------------|--------|------------|-----------------|----------------------|
| [FILL IN] | | | | | | |

---

## 9. Contact List & Vendor Information

### 9.1 Internal Contacts

| Role | Name | Phone | Email | Notes |
|------|------|-------|-------|-------|
| Practice Owner | [FILL IN] | [FILL IN] | [FILL IN] | Has off-site NAS at home |
| Office Manager | [FILL IN] | [FILL IN] | [FILL IN] | |
| IT Support / MSP | [FILL IN] | [FILL IN] | [FILL IN] | BAA on file? [YES/NO] |

### 9.2 Vendor Contacts

| Vendor | Product | Support Phone | Account/Contract # | SLA | Notes |
|--------|---------|---------------|-------------------|-----|-------|
| VMware / Broadcom | vSphere ESXi | [FILL IN] | [FILL IN] | [FILL IN] | |
| Microsoft | Windows Server 2022 | [FILL IN] | [FILL IN] | | |
| Open Dental | Practice management | (503) 363-5432 | [FILL IN] | | |
| CareStream Dental | CS Imaging 8 | 1-800-944-6365 | [FILL IN] | | |
| Vatech America | Ez3D-i / EzDent-i | 1-888-396-6288 | [FILL IN] | | |
| Synology | DS920+ / NAS | [FILL IN] | [FILL IN] | | |
| Server Hardware Vendor | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Next-day replacement? |
| Internet Service Provider | [FILL IN] | [FILL IN] | [FILL IN] | | |
| Cyber Insurance | [FILL IN] | [FILL IN] | [FILL IN] | | |

### 9.3 Credential Storage

> **CRITICAL:** Credentials must be stored securely but must be accessible in a disaster.

Recommended approach: Use a password manager (e.g., Bitwarden, 1Password) with the master password stored in a **sealed, signed envelope** in a fireproof safe at the practice AND a second copy at the off-site location.

| System | Username | Password Location |
|--------|----------|-------------------|
| ESXi web console | root | [Password manager entry name] |
| Windows Server | Administrator | [Password manager entry name] |
| Synology DS920+ (on-site) | admin | [Password manager entry name] |
| Synology (off-site) | admin | [Password manager entry name] |
| Open Dental | [FILL IN] | [Password manager entry name] |
| CareStream | [FILL IN] | [Password manager entry name] |
| Vatech | [FILL IN] | [Password manager entry name] |
| Firewall/Router | [FILL IN] | [Password manager entry name] |

---

## 10. Appendices

### Appendix A — Application-Level Database Backup Scripts

These scripts should be created inside the Windows Server 2022 VM and scheduled via Windows Task Scheduler to run nightly (e.g., 11:00 PM after the office closes).

#### A.1 Open Dental (MySQL/MariaDB) Database Dump

```batch
@echo off
REM === Open Dental MySQL Database Backup ===
REM Schedule: Nightly at 23:00 via Task Scheduler
REM Run from: The server hosting the MySQL/MariaDB instance

SET TIMESTAMP=%DATE:~10,4%%DATE:~4,2%%DATE:~7,2%_%TIME:~0,2%%TIME:~3,2%
SET TIMESTAMP=%TIMESTAMP: =0%
SET BACKUP_DIR=D:\DatabaseBackups\OpenDental
SET MYSQL_BIN="C:\Program Files\MariaDB 10.5\bin\mysqldump.exe"
SET MYSQL_USER=root
SET MYSQL_PASS=[FILL IN - USE A DEDICATED BACKUP USER WITH READ-ONLY ACCESS]
SET DATABASE=opendental

REM Create backup directory if it doesn't exist
IF NOT EXIST %BACKUP_DIR% MKDIR %BACKUP_DIR%

REM Perform the dump
%MYSQL_BIN% --user=%MYSQL_USER% --password=%MYSQL_PASS% --single-transaction --routines --triggers --databases %DATABASE% > "%BACKUP_DIR%\opendental_%TIMESTAMP%.sql"

IF %ERRORLEVEL% EQU 0 (
    echo SUCCESS: Open Dental backup completed at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
) ELSE (
    echo FAILURE: Open Dental backup FAILED at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
)

REM Clean up backups older than 14 days (local retention; Synology handles long-term)
forfiles /p "%BACKUP_DIR%" /s /m *.sql /d -14 /c "cmd /c del @path" 2>nul

exit /b
```

> **Important notes:**
> - Open Dental recommends **MariaDB 10.5** as the database engine. If you are still on MySQL 5.5, plan an upgrade. MySQL 5.5 is the minimum version required for InnoDB conversion, but MariaDB 10.5 is strongly recommended.
> - If the Open Dental database uses **InnoDB** (recommended), the `--single-transaction` flag ensures a consistent snapshot without locking tables. The InnoDB storage engine does NOT work with most file-level "hot copy" or online backup tools — you must use logical dumps (`mysqldump`) or MariaDB's `mariabackup` utility.
> - If the database uses **MyISAM**, you must use `--lock-all-tables` instead. Refer to the [Open Dental InnoDB documentation](https://www.opendental.com/site/mysqlinnodb.html) for your storage engine.
> - Create a **dedicated MySQL user** with read-only (SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER) privileges for backups. Do not use root. See [Open Dental MySQL Security](https://opendental.com/manual/securitymysql.html).
> - Do NOT expose the MySQL port (default 3306) to the internet. Do NOT give database credentials to third-party vendors with full control — read-only access is acceptable.
> - The `D:\DatabaseBackups\` directory should be included in the Synology backup scope OR mapped to a Synology share.
> - **Verification:** To verify a backup is good, restore it to a workstation that is NOT connected to the network. Ensure Open Dental and MySQL/MariaDB versions match the versions that were backed up.

#### A.2 CareStream CS Imaging 8 (Embedded SQL Server) Database Backup

```batch
@echo off
REM === CareStream CS Imaging Database Backup ===
REM CareStream uses an embedded Microsoft SQL Server instance.
REM Check the CS Imaging Server Configuration tool for the database instance name.

SET TIMESTAMP=%DATE:~10,4%%DATE:~4,2%%DATE:~7,2%_%TIME:~0,2%%TIME:~3,2%
SET TIMESTAMP=%TIMESTAMP: =0%
SET BACKUP_DIR=D:\DatabaseBackups\CareStream

IF NOT EXIST %BACKUP_DIR% MKDIR %BACKUP_DIR%

REM Use sqlcmd to perform the backup. Adjust the instance name as needed.
REM The instance name can be found in CS Imaging Server Configuration tool.
sqlcmd -S ".\ABOREL_CS" -Q "BACKUP DATABASE [CSImaging] TO DISK='%BACKUP_DIR%\csimaging_%TIMESTAMP%.bak' WITH FORMAT, COMPRESSION, STATS=10"

IF %ERRORLEVEL% EQU 0 (
    echo SUCCESS: CareStream backup completed at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
) ELSE (
    echo FAILURE: CareStream backup FAILED at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
)

REM Clean up old backups
forfiles /p "%BACKUP_DIR%" /s /m *.bak /d -14 /c "cmd /c del @path" 2>nul

exit /b
```

> **Notes:**
> - The SQL Server instance name (`.\ADOREL_CS` above) is a placeholder — check your CS Imaging Server Configuration tool's Service tab for the actual instance name.
> - CareStream has a **built-in database backup path** configurable in the CS Imaging Server Configuration tool (Service tab → "Directory for database backup"). Verify this is active, pointing to your backup directory, and running on schedule. If a backup fails, CS Imaging 8 will display an error message to end users — do not ignore this.
> - To locate your data paths: Open CS Imaging Server → Configure → General Setting tab → browse "Image Repository" path. This is your image store that must also be backed up.
> - Also back up the **image repository** (image files). This is typically a large folder; ensure it is within the VM-level backup scope.

#### A.3 Vatech EzServer Database Backup

```batch
@echo off
REM === Vatech EzServer Database Backup ===
REM Consult Vatech's EzDent-i DB & Fileserver backup documentation
REM from the Vatech Customer Learning Center for exact paths.

SET TIMESTAMP=%DATE:~10,4%%DATE:~4,2%%DATE:~7,2%_%TIME:~0,2%%TIME:~3,2%
SET TIMESTAMP=%TIMESTAMP: =0%
SET BACKUP_DIR=D:\DatabaseBackups\Vatech

IF NOT EXIST %BACKUP_DIR% MKDIR %BACKUP_DIR%

REM Copy the EzServer data directory. Adjust the source path per your installation.
REM Default path: C:\Program files (x86)\Vatech\Common\FM
REM The data path is configured in Config_base.ini (see fm_top_dir= setting).
REM Also back up capture programs at C:\VCaptureSW\
SET VATECH_DATA="C:\Program files (x86)\Vatech\Common\FM"

robocopy %VATECH_DATA% "%BACKUP_DIR%\ezserver_%TIMESTAMP%" /MIR /R:3 /W:5 /NP /LOG:"%BACKUP_DIR%\robocopy_log_%TIMESTAMP%.txt"

IF %ERRORLEVEL% LEQ 3 (
    echo SUCCESS: Vatech backup completed at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
) ELSE (
    echo FAILURE: Vatech backup FAILED at %DATE% %TIME% (robocopy exit code: %ERRORLEVEL%) >> "%BACKUP_DIR%\backup_log.txt"
)

exit /b
```

> **Notes:**
> - **Vatech America states that ONLY certified IT professionals should set up, manage, and monitor daily backups.** Vatech support will only assist certified IT professionals due to HIPAA requirements. They will tell you *what* to back up, but will not recommend specific backup software or methodology.
> - Default database path: `C:\Program files (x86)\Vatech\Common\FM` — but verify against your installation. Files may be on a different drive depending on what was selected during initial install.
> - Also back up **capture programs** at `C:\VCaptureSW\`.
> - Vatech recommends a **full PC clone/image** of the Capture PC for ease of restoration in the event of hardware failure.
> - To locate the EzServer: if EzDent-i points to IP `127.0.0.1`, the database is on the same PC. Any other IP means it's on a separate server.
> - The EzServer data path may differ — check `Config_base.ini` for the `fm_top_dir=` value.
> - CBCT data is very large (200-500 MB per scan). Ensure backup storage can accommodate the full dataset.
> - **Before any software updates:** Vatech requires a working backup be in place and verified before updating EzServer, EzDent-i, or Ez3D-i.

#### A.3b Vatech Capture Programs Backup

```batch
@echo off
REM === Vatech Capture Programs Backup ===
REM Backs up the capture software and installation files.
REM This is separate from the database - these are the programs that
REM interface with the imaging hardware.

SET TIMESTAMP=%DATE:~10,4%%DATE:~4,2%%DATE:~7,2%_%TIME:~0,2%%TIME:~3,2%
SET TIMESTAMP=%TIMESTAMP: =0%
SET BACKUP_DIR=D:\DatabaseBackups\VatechCapture

IF NOT EXIST %BACKUP_DIR% MKDIR %BACKUP_DIR%

REM Copy the capture programs directory
robocopy "C:\VCaptureSW" "%BACKUP_DIR%\VCaptureSW_%TIMESTAMP%" /MIR /R:3 /W:5 /NP /LOG:"%BACKUP_DIR%\capture_robocopy_log_%TIMESTAMP%.txt"

IF %ERRORLEVEL% LEQ 3 (
    echo SUCCESS: Vatech capture programs backup completed at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
) ELSE (
    echo FAILURE: Vatech capture programs backup FAILED at %DATE% %TIME% >> "%BACKUP_DIR%\backup_log.txt"
)

exit /b
```

#### A.4 Master Backup Orchestrator

Create a master script that calls all three, and schedule it as a single Task Scheduler job:

```batch
@echo off
REM === Master Nightly Database Backup ===
REM Schedule: Nightly at 23:00 via Windows Task Scheduler
REM Run as: A service account with appropriate database permissions

echo ============================================ >> D:\DatabaseBackups\master_log.txt
echo Master backup started at %DATE% %TIME% >> D:\DatabaseBackups\master_log.txt
echo ============================================ >> D:\DatabaseBackups\master_log.txt

call D:\DatabaseBackups\Scripts\backup_opendental.bat
call D:\DatabaseBackups\Scripts\backup_carestream.bat
call D:\DatabaseBackups\Scripts\backup_vatech.bat
call D:\DatabaseBackups\Scripts\backup_vatech_capture.bat

echo Master backup finished at %DATE% %TIME% >> D:\DatabaseBackups\master_log.txt
echo ============================================ >> D:\DatabaseBackups\master_log.txt
```

### Appendix B — ESXi Host Configuration Backup

Run this from a machine that can reach the ESXi host via SSH (or schedule it on a workstation).

```bash
# Backup ESXi configuration (run via SSH to the ESXi host or from vCLI)
# This creates a config bundle that can restore networking, storage, and settings.

vim-cmd hostsvc/firmware/sync_config
vim-cmd hostsvc/firmware/backup_config

# The backup file is downloadable at:
# http://[ESXi-IP]/downloads/configBundle-[hostname].tgz
# Save this file to the Synology NAS backup share weekly.
```

Alternatively, use PowerCLI from a Windows workstation:

```powershell
# PowerCLI ESXi Config Backup
Connect-VIServer -Server [ESXi-IP] -User root -Password [password]
Get-VMHostFirmware -VMHost [ESXi-IP] -BackupConfiguration -DestinationPath "\\[NAS-IP]\backups\esxi-config\"
Disconnect-VIServer -Confirm:$false
```

### Appendix C — Restoring a VM from Hyper Backup (When ABB Is Not Available on Off-site NAS)

If the off-site Synology only has a Hyper Backup copy (not an Active Backup for Business copy), the restoration process is different:

1. On the off-site Synology, open **Hyper Backup**.
2. Browse the backup and locate the VM disk files (`.vmdk` files).
3. Restore/extract the VM files to a local share on the Synology.
4. Transfer the `.vmdk` files to the new ESXi host datastore (via SCP, SFTP, or the ESXi datastore browser).
5. Create a new VM in ESXi with matching specs (CPU, RAM, disk controller type) and attach the existing `.vmdk` as the boot disk.
6. Power on and verify.

> This process is slower than an ABB direct restore. If possible, install Active
> Backup for Business on the off-site NAS as well and configure it as a
> replication target for ABB rather than relying solely on Hyper Backup.

### Appendix D — Synology Encryption Verification Checklist

Run through this checklist on **both** Synology NAS devices:

- [ ] **Shared folder encryption:** Go to Control Panel → Shared Folder → select each folder → Edit → Encryption tab. Verify AES-256 encryption is enabled. If not, you must create a new encrypted shared folder and migrate data.
- [ ] **Hyper Backup encryption:** Open the Hyper Backup task → Settings → verify "Enable client-side encryption" is checked.
- [ ] **Network transport encryption:** Verify Synology-to-Synology replication uses an encrypted tunnel. If using Hyper Backup over the internet, ensure it connects via HTTPS or Synology QuickConnect (which uses TLS). For maximum security, configure a site-to-site VPN between office and home networks.
- [ ] **Firewall:** Control Panel → Security → Firewall. Enable the firewall and allow only necessary ports.
- [ ] **Disable default admin:** Create a new admin account with a strong, unique password. Disable the built-in "admin" account.
- [ ] **Enable MFA:** Control Panel → Security → Account → 2-Factor Authentication.
- [ ] **Enable auto-block:** Control Panel → Security → Protection → enable auto-block after failed login attempts.
- [ ] **Update DSM:** Keep DiskStation Manager updated to the latest version for security patches.

### Appendix E — Recommended Backup Schedule

| Time | Action | Tool |
|------|--------|------|
| Every 1 hour (business hours) | Incremental VM snapshot | Synology ABB |
| 11:00 PM nightly | Application database dumps (Open Dental, CareStream, Vatech) | Windows Task Scheduler scripts (Appendix A) |
| 12:00 AM nightly | Full/incremental VM backup | Synology ABB |
| 2:00 AM nightly | Replicate on-site NAS to off-site NAS | Synology Hyper Backup / Snapshot Replication |
| Weekly (Sunday AM) | ESXi host configuration export | PowerCLI script (Appendix B) |
| Weekly (Sunday AM) | Btrfs scrub on both NAS devices | Synology Storage Manager schedule |
| Monthly | File-level restore test | Manual (see Section 8.2) |
| Quarterly | Full VM restore test | Manual (see Section 8.3) |
| Annually | Full disaster simulation | Manual (see Section 8.4) |

### Appendix F — Quick-Reference Wallet Card

Print this, laminate it, and keep copies in the practice owner's wallet and the fire-safe:

```
╔══════════════════════════════════════════════════════╗
║          DISASTER RECOVERY QUICK REFERENCE           ║
╠══════════════════════════════════════════════════════╣
║                                                      ║
║  Full DR Plan Location: [FILL IN - e.g., practice    ║
║  fire-safe, off-site NAS, password manager notes]    ║
║                                                      ║
║  ESXi Host IP:    [FILL IN]                          ║
║  Synology (site): [FILL IN]                          ║
║  Synology (home): [FILL IN]                          ║
║                                                      ║
║  IT Support:      [FILL IN - name & phone]           ║
║  Password Mgr:    [FILL IN - app name]               ║
║  Master Password: In sealed envelope at [FILL IN]    ║
║                                                      ║
║  Step 1: Call IT support                             ║
║  Step 2: Refer to full DR plan                       ║
║  Step 3: If breach suspected, see Section 7.6        ║
║                                                      ║
╚══════════════════════════════════════════════════════╝
```

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-24 | [FILL IN] | Initial creation |
| 1.1 | 2026-02-24 | [FILL IN] | Updated with vendor-specific backup paths (Vatech official FM path, CareStream Server Config tool references, Open Dental MariaDB 10.5 recommendation). Added Vatech capture programs backup script. Clarified Synology BAA availability (C2 cloud only). Added Open Dental MySQL security notes and backup verification guidance. |

---

## Signatures

This plan has been reviewed and approved:

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Practice Owner | | | |
| IT Administrator/MSP | | | |
| HIPAA Privacy Officer | | | |

---

*This document contains references to systems storing Protected Health Information (PHI).
Store this document securely. Do not share outside of authorized personnel.
Review and update every 6 months or after any significant infrastructure change.*
