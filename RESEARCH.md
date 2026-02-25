# Backup & Disaster Recovery Research Notes

## Dental Practice — Research compiled 2026-02-25

> This document contains all raw research findings gathered during the planning
> phase. Use this alongside `DISASTER-RECOVERY-PLAN.md` when filling in details
> and implementing changes.

---

## Table of Contents

1. [HIPAA Backup & Retention Requirements](#1-hipaa-backup--retention-requirements)
2. [Open Dental Backup Best Practices](#2-open-dental-backup-best-practices)
3. [CareStream CS Imaging 8 Backup Notes](#3-carestream-cs-imaging-8-backup-notes)
4. [Vatech Ez3D-i / EzDent-i Backup Notes](#4-vatech-ez3d-i--ezdent-i-backup-notes)
5. [Synology DS920+ & HIPAA Compliance](#5-synology-ds920--hipaa-compliance)
6. [Dental Practice DR Planning — Industry Guidance](#6-dental-practice-dr-planning--industry-guidance)
7. [Source Links](#7-source-links)

---

## 1. HIPAA Backup & Retention Requirements

### 1.1 Core Retention Period: 6 Years

The HIPAA Security Rule mandates that all PHI-related compliance documentation must be retained for a minimum of **six years** from the date on which a policy or procedure was last in force, a risk assessment was last used to make a security decision, or an authorization to disclose PHI was signed by a patient. Reference: **45 CFR § 164.316(b)(2)(i)**.

This applies to all Covered Entities, Business Associates (BAs), and Subcontractors.

### 1.2 Documents Subject to 6-Year Retention

- Patient authorization forms for disclosure of PHI
- Workforce training records
- Business Associate Agreements (BAAs)
- Incident logs and security event logs
- Complaints and investigation records
- Audit logs and system activity logs
- Disaster recovery and contingency planning documents
- IT security system reviews (access controls, password policies, automatic log-off, audit controls)
- Risk assessments

### 1.3 Medical Records vs. HIPAA Documentation (Critical Distinction)

HIPAA's retention rule is often confused with medical record-keeping requirements, but they are **distinct**:

- **HIPAA** does NOT prescribe how long medical records must be kept
- **State law** determines medical record retention periods — these vary widely (5 years in some states to 10+ years, or until a minor reaches adulthood)
- **HIPAA** mandates that all **compliance-related documentation** be retained for 6 years, regardless of state medical record rules

**Action item:** Look up your specific state's medical record and dental record retention requirements.

### 1.4 Backup Frequency

HIPAA does not state a specific frequency (e.g., "daily"). The key is to determine your **Recovery Point Objective (RPO)** — how much data you can afford to lose:

- A busy hospital EHR may need near-continuous backup
- A small dental office's server might be fine with nightly backup
- **For most dental practices, daily backup is the minimum standard**
- For Open Dental (scheduling, billing, treatment plans), hourly incremental backups during business hours are recommended given high transaction volume

### 1.5 Data Disposal Requirements

When the minimum retention period has been reached:

- HIPAA requires all forms of PHI to be destroyed or disposed of securely
- Electronic media options: clearing (overwrite with non-sensitive data), purging (degaussing), or physical destruction (disintegration, pulverization, melting, incinerating, shredding)
- The Privacy and Security Rules do not require a particular disposal method

### 1.6 HIPAA Security Rule — Contingency Plan Requirements

Under **45 CFR § 164.308(a)(7)**, covered entities must implement:

| Requirement | Status | CFR Reference |
|-------------|--------|---------------|
| Data Backup Plan | **Required** | §164.308(a)(7)(ii)(A) |
| Disaster Recovery Plan | **Required** | §164.308(a)(7)(ii)(B) |
| Emergency Mode Operation Plan | **Required** | §164.308(a)(7)(ii)(C) |
| Testing & Revision Procedures | **Addressable** | §164.308(a)(7)(ii)(D) |
| Application & Data Criticality Analysis | **Addressable** | §164.308(a)(7)(ii)(E) |

"Addressable" does NOT mean optional — it means you must implement it or document why an equivalent alternative is used.

### 1.7 Dental-Specific HIPAA Enforcement

- In 2022, three dental practices reached settlements totaling **$142,500** for noncompliance with patients' access rights, disclosing PHI on social media, and impermissibly using PHI for marketing
- In 2024, Gums Dental Care (Silver Springs, MD) was fined **$70,000** for a right of access violation (the 50th HIPAA covered entity found guilty of such a violation)
- Penalties for failure to retain documentation: **$137 to $68,928 per violation**, depending on severity

### 1.8 Physical Safeguard Requirements for Dental Offices

The HIPAA physical safeguards concern:
- Security of computer systems and the environment in which they are situated
- Establishing a facility plan and a contingency plan for emergencies
- Implementing validation procedures to restrict physical access to ePHI

This applies to BOTH the on-site server AND the off-site Synology NAS at the owner's home.

---

## 2. Open Dental Backup Best Practices

### 2.1 What Must Be Backed Up

Two key components:

1. **The MySQL/MariaDB Database:** Typically located at `C:\mysql\data\opendental\` (may differ per installation)
2. **The Images Folder (A-Z Folder):** `OpenDentImages\` — contains scanned documents, imported images, etc.

### 2.2 Database Engine

- Open Dental recommends **MariaDB 10.5** as the database engine
- Many offices are still on **MySQL 5.5** — this is the minimum version for InnoDB conversion
- **Strong recommendation:** Upgrade to MariaDB 10.5 before converting storage engine to InnoDB
- InnoDB is the recommended storage engine (better crash recovery, row-level locking, ACID compliance)

### 2.3 Backup Methods for InnoDB Databases

**Manual (cold) backups should only be done on MyISAM databases, NOT InnoDB.** For InnoDB:

- **SQL Dump Backup (Hot Method):** Creates a `.sql` file, can be compressed to `.zip`. Can be done while Open Dental is in use but may cause slowness. Use `mysqldump --single-transaction`.
- **Binary Logs (Hot Method):** Requires specialized IT. Can be used with `mariabackup` or dump backups for up-to-the-minute point-in-time recovery.
- **Key warning:** The InnoDB database type does NOT function with most file-level "online backup" or "hot copy" tools. You MUST use logical dumps or `mariabackup`.

### 2.4 Built-In Backup Tool

- Open Dental has a built-in Backup Tool accessible from the Setup menu
- **Must be run from the server** that hosts the MySQL database — running from a workstation will error
- **Do NOT restore a backup over a live production database** — data loss can occur and is irreversible
- **Do NOT replace tables within a database with tables from another database** — foreign key issues will occur

### 2.5 MySQL Security

- **Never expose MySQL port (3306) to the internet** — do not open this port on routers
- **Do not give database credentials to third-party vendors** — read-only access is acceptable, but many vendors ask for full control which is dangerous
- Create a dedicated backup user with minimal privileges: SELECT, LOCK TABLES, SHOW VIEW, EVENT, TRIGGER

### 2.6 Verifying Backups

- Restore to a workstation that is **NOT connected to the network** (e.g., a laptop)
- Ensure Open Dental and MySQL/MariaDB versions on the test machine **match the versions that were backed up**
- The only useful backup is a **tested, verified backup** — "if you're only copying your data but not testing your backup, you don't really have a backup at all"

### 2.7 Online/Cloud Backup Considerations

**Advantages:**
- Automated with no user action after setup
- Off-site protection against fire, flood, burglary

**Disadvantages:**
- Initial backup can be very slow — up to a week for large image sets
- Subsequent backups are incremental (faster)
- InnoDB databases do NOT work with most online backup tools
- Must be encrypted and regularly tested

### 2.8 Third-Party Tools

- **SyncBack Pro** handles MySQL backups well; supports multiple destinations including Amazon S3 (which offers a BAA for HIPAA compliance)
- Whatever tool is used, ensure it supports application-consistent snapshots or uses `mysqldump`

---

## 3. CareStream CS Imaging 8 Backup Notes

### 3.1 Database

- CareStream CS Imaging 8 uses an **embedded Microsoft SQL Server** instance
- The SQL Server instance name can be found in the **CS Imaging Server Configuration tool** → Service tab
- The instance name is often something like `.\ADOREL_CS` or `.\CARESTREAM` but varies by installation

### 3.2 Built-In Backup

- CareStream has a **built-in database backup path** configurable in the CS Imaging Server Configuration tool:
  - Service tab → "Directory for database backup"
- Verify this is active, pointing to your backup directory, and running on schedule
- If a backup fails, CS Imaging 8 will display an error message to end users — **do not ignore this**

### 3.3 Image Repository

- To locate image storage: Open CS Imaging Server → Configure → **General Setting tab** → browse "Image Repository" path
- This is typically a large folder and must be within the VM-level backup scope
- Image growth depends on patient volume and imaging frequency

### 3.4 Backup Method

- Use `sqlcmd` to perform native SQL Server backups: `BACKUP DATABASE [CSImaging] TO DISK='path' WITH FORMAT, COMPRESSION`
- The `WITH COMPRESSION` flag significantly reduces backup file size
- SQL Server native backup is application-consistent and can be done while CareStream is running

### 3.5 UPS Recommendation

- CareStream recommends a UPS with automatic shutdown software
- An unclean shutdown during imaging acquisition can corrupt the database and/or image files

---

## 4. Vatech Ez3D-i / EzDent-i Backup Notes

### 4.1 Certified IT Professional Requirement

**Vatech America states that ONLY certified IT professionals should set up, manage, and monitor daily backups.** Vatech support will only assist certified IT professionals due to HIPAA requirements. They will tell you *what* to back up but will NOT recommend specific backup software or methodology.

### 4.2 What to Back Up

1. **EzServer database:** Default path is `C:\Program files (x86)\Vatech\Common\FM`
   - Verify against your installation — the actual path is in `Config_base.ini` (look for `fm_top_dir=`)
   - Files may be on a different drive depending on what was selected during install
2. **Capture programs:** `C:\VCaptureSW\`
   - These are the programs that interface with the imaging hardware
   - Separate from the database
3. **DICOM image store:** Location varies; check EzDent-i settings

### 4.3 Locating the EzServer

- If EzDent-i points to IP `127.0.0.1`, the database is on the same PC as the application
- Any other IP means it's on a separate server
- In your case, everything runs on the single Windows Server 2022 VM

### 4.4 Data Size Considerations

- CBCT data is **very large** — each 3D scan can be **200-500 MB**
- Total data volume grows significantly with patient volume
- Ensure backup storage and network bandwidth can accommodate the full dataset
- Factor this into replication time estimates for off-site backup

### 4.5 Full PC Clone Recommendation

- Vatech recommends a **full PC clone/image** of the Capture PC for ease of restoration in the event of hardware failure
- This aligns with the VM-level backup approach already in place via Synology ABB

### 4.6 Pre-Update Requirement

- **Before any software updates:** Vatech requires a working backup be in place and verified before updating EzServer, EzDent-i, or Ez3D-i
- This is a hard requirement from Vatech support — they may refuse to assist with update issues if backups weren't verified beforehand

---

## 5. Synology DS920+ & HIPAA Compliance

### 5.1 Synology's HIPAA Positioning

Synology markets its solutions as suitable for storing ePHI. They provide documentation on how to design a HIPAA-compliant infrastructure using their products. A real-world example: **Alabama Cancer Care** (10 physical servers, 30 VMs) uses Synology Active Backup for Business for HIPAA-compliant backup.

### 5.2 Encryption Capabilities

- **AES-256 shared folder encryption** — ePHI can be stored in folders encrypted by a separate set of keys, protecting data even if admin accounts are compromised
- **Active Backup for Business 2.2.0+** supports compression and encryption at the backup destination
- **Hyper Backup** supports client-side encryption for off-site replication

### 5.3 Business Associate Agreements (BAAs)

**Critical finding:** Synology **only offers BAAs for their C2 cloud services:**

- C2 Object Storage
- C2 Storage
- C2 Backup
- C2 Transfer
- C2 Password

The **on-premise DS920+ does NOT require a BAA** since the practice self-manages it. However, if you add Synology C2 cloud as a third backup tier, you **MUST** execute a BAA with Synology. Contact C2 support to request one.

### 5.4 Auditing & Access Controls

- Synology provides detailed logs — up to **40 types of actions** can be monitored
- Tracks changes to files, folder changes, and service activities
- Can perform audits to pinpoint suspicious events
- This satisfies HIPAA audit control requirements (§164.312(b))

### 5.5 Data Integrity

- The DS920+ supports **Btrfs file system** with built-in checksums
- Btrfs provides data scrubbing (scheduled integrity checks)
- Immutable snapshots can protect against ransomware

### 5.6 Third-Party Certifications

- Synology C2 US data centers: **ISO 27001** and **SOC 2 Type II** certified
- Europe and APAC data centers: ISO 27001 certified

### 5.7 DS920+ Limitations

- It is a **consumer/prosumer 4-bay NAS** — limited CPU and RAM compared to rackmount units
- Can run Synology Virtual Machine Manager (VMM) for emergency VM hosting, but performance will be degraded
- If the DS920+ hardware fails, you lose local rapid-restore capability
- Consider it a **single point of failure** for the backup chain

### 5.8 Recommended Security Hardening for Both NAS Devices

- Enable AES-256 encryption on all shared folders containing ePHI
- Disable the default "admin" account; create a new admin with a strong unique password
- Enable 2-Factor Authentication (MFA)
- Enable auto-block after failed login attempts
- Enable the firewall — allow only necessary ports
- Keep DSM updated to the latest version
- For the off-site NAS: lock it in a secured room, put it behind a firewall with no inbound access except Synology VPN, and connect it to a UPS

---

## 6. Dental Practice DR Planning — Industry Guidance

### 6.1 HIPAA Requires a Written DR Plan

The HIPAA Security Rule requires covered entities to have a **written disaster recovery plan** that outlines how to restore critical patient information and resume normal operations. The plan must address scenarios ranging from a single server failure to total site loss.

### 6.2 Statistics

- More than **40% of small businesses** are forced to close permanently following a disaster
- Of those that manage to reopen, **25% fail within a year**
- Having a DR plan is not optional — it is a **legal requirement** for HIPAA-covered entities

### 6.3 RTO/RPO Guidance for Dental Practices

| Solution Type | Typical RTO | Notes |
|---------------|-------------|-------|
| BDR (Backup & Disaster Recovery) appliance with instant VM | **< 60 minutes** | Backs up entire server; provides on-site VM boot in emergency |
| Cloud backup with local NAS restore | **2-4 hours** | Depends on data volume and restore speed |
| Off-site only restore (no local backup available) | **8-24 hours** | Depends on hardware procurement + data transfer over WAN |
| Full cloud restore to new hardware | **12-48 hours** | Depends on internet speed and data volume |

### 6.4 The 3-2-1-1 Rule (Healthcare Standard)

- **3** copies of data (production + 2 backups)
- **2** different storage media types
- **1** copy off-site
- **1** copy immutable or air-gapped (ransomware protection)

Your current setup achieves 3-2-1 but is **missing the immutable/air-gapped copy** (the "1" for ransomware protection).

### 6.5 Common Pitfalls in Dental Practice Backups

- Local hard drive backups that never leave the office
- Cloud backups that aren't actively monitored (and often aren't working)
- No functional backup at all
- Backup exists but has never been tested with a full restore
- Only one person knows how to restore (single point of knowledge failure)
- No application-level database backups — only VM snapshots (risk of corrupt database on restore)

### 6.6 ADA Standards

- **ADA TR 1021-2023:** Reviews options for data backup to prevent data loss and corruption, maintain data integrity, and restore/maintain access to data. Discusses contingency plans for emergency recovery and data authentication.
- The **ADA Emergency Planning Guide** helps practices create their own Emergency Action Plan

### 6.7 Key Recommendation: Tiered System Classification

| Tier | Description | Examples |
|------|-------------|---------|
| Tier 0 (Foundational) | Identity, DNS, networking, backup systems | ESXi host, Synology NAS, network switches, firewall |
| Tier 1 (Mission Critical) | Revenue systems, patient-facing apps, core databases | Open Dental, CareStream, Vatech |
| Tier 2 (Business Important) | Internal productivity and reporting | Email, office productivity apps |
| Tier 3 (Deferred) | Non-critical systems | Training materials, archived marketing |

### 6.8 Emergency Mode Operations

When full systems can't be restored within the RTO:

- **Patient scheduling:** Keep a printed current-week schedule (print every Friday)
- **Records access:** If Synology VMM instant-restore is available, boot a read-only VM on the NAS
- **Imaging:** Defer non-urgent; refer urgent cases to a partner practice or imaging center
- **Billing:** Queue on paper; submit when systems restored
- **Prescriptions:** Use state PDMP website + manual prescriptions

---

## 7. Source Links

### HIPAA Requirements
- [HIPAA Retention Requirements — 2026 Update (HIPAA Journal)](https://www.hipaajournal.com/hipaa-retention-requirements/)
- [HIPAA Record Retention Requirements (HIPAA Guide)](https://www.hipaaguide.net/hipaa-record-retention-requirements/)
- [HIPAA Data Retention Requirements: 2026 Guide (Sprinto)](https://sprinto.com/blog/hipaa-data-retention-requirements/)
- [HIPAA Rules for Dentists — Updated for 2026 (HIPAA Journal)](https://www.hipaajournal.com/hipaa-rules-for-dentists/)
- [HIPAA Data Backup: Top Guide to Compliance and Recovery (HIPAA Vault)](https://www.hipaavault.com/cyber-data/hipaa-data-backup/)
- [HIPAA Data Retention & Backup Requirements (Kiteworks)](https://www.kiteworks.com/hipaa-compliance/hipaa-compliant-data-retention/)
- [HIPAA Rules on Data Backup and Disaster Recovery (HIPAA Guard)](https://hipaa-guard.com/hipaa-rules-on-data-back-up-and-disaster-recovery-plan/)

### Open Dental
- [Open Dental — Backups (Official Manual)](https://opendental.com/manual/backups.html)
- [Open Dental — Backup Tool](https://www.opendental.com/manual/backuptool.html)
- [Open Dental — InnoDB](https://www.opendental.com/site/mysqlinnodb.html)
- [Open Dental — Manual Backups](https://www.opendental.com/manual/backupsmanual.html)
- [Open Dental — MySQL Security](https://opendental.com/manual/securitymysql.html)
- [Open Dental — Online Backups](https://opendental.com/manual/backupsonline.html)
- [How to Backup Your Open Dental Database (Divergent Dental)](https://divergentdental.com/how-to-backup-your-open-dental-database/)
- [Open Dental Blog — Disaster Recovery Data Center](https://opendental.blog/disaster-recovery-data-center/)
- [Open Dental Blog — How to Update Your Disaster Recovery Plan](https://opendental.blog/disaster-recovery-plan/)

### Synology & HIPAA
- [Synology HIPAA Compliance (Official)](https://www.synology.com/en-us/security/hipaa)
- [Synology C2 HIPAA Compliance & BAA](https://c2.synology.com/en-us/resource/hipaa)
- [HIPAA-Compliant Cloud Backup for Healthcare (Synology C2 Blog)](https://medium.com/synologyc2/hipaa-compliant-cloud-backup-for-healthcare-professionals-659feb6c6fc9)
- [Alabama Cancer Care Case Study (Synology)](https://www.synology.com/en-us/company/case_study/Alabama_Cancer_Care)
- [Active Backup for Business Technical Specifications](https://www.synology.com/en-global/dsm/7.3/software_spec/abb)

### Dental IT & Disaster Recovery
- [Effective Dental Disaster Recovery (Zenith Dental IT)](https://zenithdentalit.com/effective-dental-disaster-recovery-planning/)
- [Emergency Planning and Disaster Recovery in Dental (ADA)](https://www.ada.org/resources/practice/practice-management/emergency-planning-and-disaster-recovery-planning-in-the-dental-office)
- [Disaster Preparedness for Dental Practices (Medix Dental IT)](https://medixdental.com/the-critical-importance-of-disaster-preparedness-for-dental-and-dental-specialty-practices/)
- [Dental Data Backup and Disaster Recovery (Medix Dental IT)](https://medixdental.com/dental-data-backup-and-disaster-recovery/)
- [Understanding Disaster Recovery Plans (Darkhorse Tech)](https://www.darkhorsetech.com/blogs/understanding-disaster-recovery-plans)
- [Best-in-Class Backup & Recovery for Dental Offices (Pact-One)](https://www.pact-one.com/2025/10/best-in-class-backup-recovery-strategies-for-dental-offices/)
- [Disaster Recovery Planning for Healthcare (Healthy IT)](https://www.myhealthyit.com/disaster-recovery-healthcare/)
- [ADA TR 1021-2023: Dental Practice Data Integrity (ANSI Blog)](https://blog.ansi.org/ansi/ada-tr-1021-2023-dental-practice-data-integrity/)
- [HIPAA Data Recovery / Backup (DDS Rescue)](https://ddsrescue.com/hipaa-data-recovery/)
- [Dental Backup Solutions (NovaBACKUP)](https://www.novabackup.com/solutions/dental-backup)

### General DR Planning
- [IT Disaster Recovery Plan Template (Accrets)](https://www.accrets.com/backupanddr/it-disaster-recovery-plan-template/)
- [Disaster Recovery Plan Template (PDQ)](https://www.pdq.com/blog/disaster-recovery-plan-template-with-free-downloads/)
- [RPO and RTO Explained (Druva)](https://www.druva.com/blog/understanding-rpo-and-rto)

---

*Research compiled during Claude Code session 2026-02-25. Use this alongside DISASTER-RECOVERY-PLAN.md for implementation.*
