# Backup Strategy

> **In one line —** copies of your data that survive things replication cannot survive; and a backup nobody has restored is a hypothesis, not a backup.

| | |
|---|---|
| **Category** | Concept / Practice |
| **Architectural Layer** | Data |
| **Related notes** | [Disaster Recovery](Disaster%20Recovery.md) · [High Availability](High%20Availability.md) · [RDS](../12%20-%20Cloud%20Architecture/RDS.md) · [S3](../12%20-%20Cloud%20Architecture/S3.md) · [Cloud Security](../12%20-%20Cloud%20Architecture/Cloud%20Security.md) · [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) |

---

## 1. Short Definition

*What is it?*

A backup is an **independent copy of data at a point in time**, stored so that losing the original — or having it deliberately destroyed — does not lose the copy. The strategy is the set of decisions about what, how often, how long, and how you get it back.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Things that redundancy does NOT protect against
    a migration that drops a column           → replicated instantly
    DELETE FROM orders with no WHERE clause    → replicated instantly
    ransomware encrypting your files           → replicated instantly
    a bug corrupting records over three weeks  → replicated, and old copies gone
    an angry or compromised administrator      → they can delete replicas too
    ↓
Replication faithfully copies destruction. That is its job.
```

> [!IMPORTANT]
> **Replication is for availability; backups are for recovery. Confusing the two is the most consequential mistake in this entire topic.** A Multi-AZ database with three read replicas has excellent availability and, by itself, no protection whatsoever against `DROP TABLE`. The two mechanisms defend against different categories of failure, and you need both.

---

## 3. Architecture Position

```text
   PRODUCTION DATA
        │
        ├──► replicas          same account, same region   → AVAILABILITY
        │
        ├──► snapshots         same account                → quick recovery
        │
        ├──► BACKUP COPIES     SEPARATE ACCOUNT            → survives compromise
        │       │              versioned, Object Lock
        │       │
        │       └──► another region                        → survives a region loss
        │
        └──► ARCHIVE           Glacier / cold storage       → compliance, long tail
```

> [!TIP]
> **The separate-account line is the one that matters most, and it is often missing.** Snapshots in the same account, deletable by the same credentials that manage production, do not protect against a compromised credential or a bad script. Copying them to an account with different access, where nobody has delete permission, converts them from convenient into trustworthy.

---

## 4. The two numbers that define everything

```text
RPO — Recovery Point Objective
    "how much data can we afford to lose?"
    → determines BACKUP FREQUENCY
    daily backup   → up to 24 h of data lost
    continuous WAL → seconds

RTO — Recovery Time Objective
    "how long can we be down?"
    → determines RESTORE MECHANISM
    restore a 4 TB dump   → hours
    promote a standby      → seconds
```

> [!IMPORTANT]
> **Write down RPO and RTO per dataset, and make the business agree to them.** They are not technical preferences — they are commercial decisions about acceptable loss. Without them, backup design is guesswork, and the numbers only get discovered during an incident, in the worst possible conversation. Different datasets deserve different answers: the orders table and the analytics warehouse are not equally precious.

---

## 5. The 3-2-1 rule, updated for the cloud

```text
CLASSIC 3-2-1
    3 copies of the data
    2 different media
    1 offsite

CLOUD 3-2-1-1-0
    3 copies
    2 storage types or classes
    1 in a different REGION or ACCOUNT
    1 IMMUTABLE (Object Lock / WORM)     ← the anti-ransomware clause
    0 errors on a VERIFIED restore       ← the clause everyone skips
```

> [!CAUTION]
> **The "0" is the whole point, and it is the part that is almost never done.** Backups fail silently: a job that has been erroring for four months, a snapshot of an empty volume, a dump missing a schema, an encrypted archive whose key is in the system you just lost. None of that is visible until you attempt a restore. **Automate a periodic restore into a scratch environment and check a row count.** If that sounds like a lot of work, consider that it is the only evidence your backups exist.

---

## 6. Backup types

```text
FULL            everything, every time
                simple, fast to restore, expensive to store

INCREMENTAL     only what changed since the last backup
                cheap; restore needs the full PLUS EVERY increment
                → one missing link breaks the chain

DIFFERENTIAL    everything changed since the last FULL
                middle ground; restore needs full + one differential

CONTINUOUS / PITR   transaction log shipped constantly
                → restore to any second; the best RPO available
                → this is what RDS automated backups give you
SNAPSHOT        a point-in-time copy of a volume
                fast, cheap, and NOT application-consistent by default
```

> [!CAUTION]
> **A volume snapshot of a running database is a crash-consistent copy, not a clean one.** It captures the disk as if the machine had lost power — usually recoverable, occasionally not, and never guaranteed. Use the database's own mechanism (`pg_basebackup`, RDS snapshots, `mysqldump` with proper flags) or quiesce writes first. This distinction has ended companies that thought they had backups.

---

## 7. Retention: grandfather-father-son

```text
hourly     keep 24        → recover from something that happened this morning
daily      keep 30        → the common case
weekly     keep 12        → three months back
monthly    keep 12        → a year
yearly     keep 7         → compliance
```

```text
WHY LONG RETENTION MATTERS
    slow corruption is discovered WEEKS later
    a 7-day window means the last known-good copy is already gone
```

> [!TIP]
> **Match retention to how long a problem can hide, not to how long you expect to need it.** Deletions are noticed in minutes; a subtle bug corrupting one field on some records may take a month to surface. Thirty days of dailies is a sensible floor for anything transactional, and it costs very little once old backups age into cold storage.

---

## 8. What people forget to back up

```text
BACKED UP                    OFTEN FORGOTTEN
the main database             object storage buckets (versioning ≠ backup)
                              secrets and encryption keys  ⚠
                              database schema and migrations
                              infrastructure definitions (Terraform state)
                              CI/CD configuration and pipeline secrets
                              DNS records
                              certificates
                              queue contents in flight
                              managed service configuration
                              THE RESTORE DOCUMENTATION ITSELF
```

> [!CAUTION]
> **If your backups are encrypted with a key that lives only in the system you lost, you have no backups.** This is a genuine and recurring failure: the KMS key, the vault, or the passphrase stored in the same account that was destroyed or compromised. Keys need their own recovery path, held separately, and someone must know how to use it. The same applies to the runbook — a restore procedure stored only in a wiki that is also down is not available when required.

---

## 9. Real World Example

- **RDS automated backups** — daily snapshot plus continuous logs, giving point-in-time recovery to any second in the window; retention raised from the 7-day default; see [RDS](../12%20-%20Cloud%20Architecture/RDS.md).
- **S3 versioning plus a lifecycle rule** — protects against overwrite and deletion; see [S3](../12%20-%20Cloud%20Architecture/S3.md).
- **Cross-account snapshot copies with Object Lock** — the ransomware answer.
- **A nightly logical dump** alongside snapshots, because it can be restored into a different engine version and inspected.
- **Terraform state versioned in S3**, so a corrupted state file is recoverable; see [Terraform](../13%20-%20DevOps%20and%20Delivery/Terraform.md).
- **A monthly automated restore test** that reports a row count into a monitoring dashboard.

---

## 10. Communication and Dependencies

- **Object storage** with versioning and Object Lock; see [S3](../12%20-%20Cloud%20Architecture/S3.md)
- **A separate account or subscription**, with different credentials
- **Key management** whose recovery does not depend on the system being restored
- **Monitoring on backup jobs** — success, size and age, all alarmed
- **A restore runbook** stored somewhere reachable during an outage
- **Enough network and I/O capacity** to restore within your RTO
- **Lifecycle policies**, or storage cost grows without limit

---

## 11. When To Use / When NOT To Use

> [!TIP]
> Every system with data that cannot be trivially regenerated needs backups, including ones with excellent replication. Managed automated backups plus a cross-account copy plus a tested restore covers most needs and is a day of work.

> [!CAUTION]
> - **Do not treat replicas or Multi-AZ as backups** — they replicate destruction
> - **Do not treat S3 versioning alone as a backup** — an account compromise can still remove versions unless Object Lock is set
> - **Do not keep backups only in the account they protect**
> - **Do not back up data you can regenerate** — caches, derived indexes, thumbnails
> - **Do not back up secrets into a less secure place**; a backup of sensitive data is sensitive data
> - **Do not rely on a snapshot of a running database** without checking consistency
> - **Do not skip the restore test**, whatever the schedule pressure

---

## 12. Advantages and Disadvantages

**Advantages**
- The only defence against deletion, corruption and ransomware
- Point-in-time recovery turns a catastrophe into an inconvenience
- Enables realistic staging environments from production-shaped data
- Cheap — cold storage costs very little relative to what it protects
- Required for most compliance regimes
- Managed services make the mechanics nearly free

**Disadvantages**
- **Restores are slow** — a large database takes hours, and that is your RTO
- Storage cost grows if retention is unmanaged
- Backups of sensitive data expand the surface that must be secured
- Testing restores takes real, recurring effort
- Application-consistent backups of distributed systems are genuinely hard
- Easy to believe you have them when you do not

---

## 13. Performance Impact

| Aspect | Impact |
|---|---|
| **Backup window** | I/O load during the copy; negligible on Multi-AZ, noticeable on a single node |
| **Logical dumps** | Long-running transactions and heavy read load on large datasets |
| **Snapshots** | Nearly instant to take; the first one is a full copy underneath |
| **Restore time** | Dominated by data volume and provisioned throughput — this *is* your RTO |
| **Glacier retrieval** | Minutes to 12 hours; unsuitable for anything with a short RTO |
| **Continuous log shipping** | Small, steady overhead; the best RPO for the cost |

> [!CAUTION]
> **Archive storage classes and short RTOs are incompatible, and this catches people out.** Deep Archive can take twelve hours to retrieve. If a dataset must be back within an hour, its recent backups must be in a warm class — archive only what you can afford to wait for. Test the retrieval time, not just the storage cost.

---

## 14. Security Considerations

> [!CAUTION]
> **Modern ransomware deliberately targets backups first, because attackers know that intact backups mean no payment.** The counter is immutability: Object Lock in compliance mode, in a separate account, where **no credential — including your own root account — can delete a copy before its retention expires.** If an administrator can delete your backups, so can whoever compromises that administrator.

- **Immutable copies** — Object Lock or WORM, in a separate account
- **Encrypt backups**, and keep the key recoverable independently of the protected system
- **Least privilege** — the backup writer should not have delete permission
- **A backup of sensitive data is sensitive data** — same access controls, same audit
- **Never make a snapshot public**; shared or public database snapshots have leaked real data
- **Test restores in an isolated environment**, not into production
- **Alarm on backup failure and on unexpected deletion attempts** — the latter is an intrusion signal
- **Consider what personal data retention means legally** — an old backup may conflict with a deletion request

---

## 15. Mental Model

> [!NOTE]
> **Replicas are photocopies made continuously; backups are a copy in a locked safe at a different address.**
>
> If someone shreds the original, every photocopier dutifully shreds its copy too — that is what replication does. The safe is different: it is offsite, only one person can put things in, nobody can take things out before a set date, and crucially **someone has actually opened it once to confirm the papers inside are the right ones and legible.** An unopened safe is faith, not insurance.

---

## 16. Mini Architecture Diagram

```text
   ┌──────── PRODUCTION ACCOUNT ─────────────────────────────┐
   │                                                          │
   │   DATABASE PRIMARY ──sync──► STANDBY   (availability)    │
   │        │                                                  │
   │        ├─► continuous WAL ──► automated backups (35 d)    │
   │        │                       → point-in-time recovery   │
   │        ├─► nightly logical dump → S3 (versioned)          │
   │        │                                                  │
   │   S3 DATA BUCKET (versioning ON, lifecycle set)           │
   └────────┬─────────────────────────────────────────────────┘
            │ automated copy — write-only, no delete permission
   ┌────────▼──── BACKUP ACCOUNT (different credentials) ──────┐
   │   snapshots + dumps                                       │
   │   OBJECT LOCK, compliance mode  ← nobody can delete       │
   │   30 daily · 12 weekly · 12 monthly · 7 yearly            │
   │        │                                                   │
   │        ├─► cross-region copy      (region loss)            │
   │        └─► Glacier after 90 days  (cost, long tail)        │
   └────────┬──────────────────────────────────────────────────┘
            │
   ┌────────▼──────────────────────────────────────────────────┐
   │  MONTHLY AUTOMATED RESTORE TEST                            │
   │  restore → run checks → report row count → ALARM if failed │
   │  ← this is the only thing that proves any of the above      │
   └───────────────────────────────────────────────────────────┘

   Encryption keys: held in the backup account, recoverable independently
   Runbook: stored where it is readable while production is down
```

---

## 17. Complete Request Flow

```text
─────────────── the routine ───────────────
Continuous WAL shipping gives an RPO of seconds
Nightly snapshot; a logical dump to S3 with versioning
Copies pushed to the backup account by a role with PutObject but not Delete
Object Lock applies a 30-day retention nobody can shorten
Lifecycle moves copies to Glacier at 90 days
A monthly job restores the latest backup, checks row counts, reports success
    ↓
─────────────── human error ───────────────
14:32 — a migration drops a column on the orders table
    ↓
Replicated to the standby in milliseconds. Redundancy is no help.
    ↓
Point-in-time restore to 14:31, into a NEW instance beside production
    ↓
Verified, table extracted, application repointed
    ↓
Data loss: none. Downtime: 40 minutes.
    ↓
─────────────── slow corruption ───────────────
A bug has been writing a wrong currency value for three weeks
    ↓
Only 7-day retention → the last clean copy is gone
    ↓
WITH 30-day retention → restore a 3-week-old copy into a scratch environment,
compare, and repair the affected rows
    ↓
LESSON: retention length is defined by how long a problem can HIDE
    ↓
─────────────── ransomware ───────────────
An attacker gains admin credentials in the production account
    ↓
Encrypts data, deletes snapshots, empties buckets
    ↓
The backup account uses different credentials and Object Lock
    ↓
The attacker's access cannot delete a single immutable copy
    ↓
Rebuild infrastructure from Terraform, restore from the backup account
    ↓
The company survives, specifically because of the separate account and the lock
    ↓
─────────────── the version where it goes wrong ───────────────
Same attack. Backups were snapshots in the same account, deletable.
    ↓
Nothing to restore from.
```

---

## 18. Key Takeaway

> [!IMPORTANT]
> Replication is not a backup — keep immutable copies in a separate account, retain them long enough for slow corruption to surface, and run an automated restore test, because a backup nobody has restored has not been shown to exist.

---

## 19. Common Mistakes

- **Believing replicas or Multi-AZ are backups**
- **Never testing a restore**, so nobody knows the RTO or whether the data is usable
- **Backups in the same account**, deletable by the same credentials
- **No immutability**, leaving ransomware free to delete them
- **Default 7-day retention** for data whose corruption might take a month to notice
- **Encryption keys stored inside the system being protected**
- **Volume snapshots of running databases** assumed to be application-consistent
- **Incremental chains with no verification**, where one missing link breaks everything
- **No alarm on backup failure**, so a job errors silently for months
- **Backing up the database but not the schema, secrets, DNS or infrastructure**
- **Archive storage for data with a one-hour RTO**
- **A restore runbook stored only in a system that is also down**
- **RPO and RTO never agreed with the business**, so expectations surface during the incident

---

## 20. Open Source Technologies

- **pgBackRest**, **WAL-G**, **Barman** — serious PostgreSQL backup with PITR and verification
- **Percona XtraBackup** — consistent hot backups for MySQL
- **restic**, **BorgBackup**, **Kopia** — deduplicated, encrypted, verifiable file backups
- **Velero** — Kubernetes cluster and persistent volume backups
- **rclone** — move backups between storage providers
- **litestream** — continuous replication for SQLite to object storage
- **MinIO** — S3-compatible target with Object Lock support
- **Prometheus** + **Alertmanager** — alarm on backup age, size and job failure
- **AWS Backup**, **Object Lock** — the managed cross-account, immutable path

---

## 21. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 22. Workbook Exercise

- [ ] Restore your most important database into a scratch environment. Time it. That number is your real RTO.
- [ ] Check whether any credential that manages production can also delete your backups.
- [ ] Write down RPO and RTO per dataset, and get someone from the business to agree.
- [ ] Confirm your backup encryption key is recoverable without the production account.
- [ ] Check the age of your oldest usable backup against how long a subtle bug could hide.
- [ ] Verify there is an alarm on backup job failure, not just on success.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
production data → snapshots + continuous logs → separate account (immutable, versioned)
                                              → cross-region → archive
                                              → automated restore test
```

## 2. Request Flow

```text
Input       data that could be deleted, corrupted or encrypted by someone else
    ↓
Processing  independent copies, immutably retained, in a separate blast radius
    ↓
Output      a restore that has been proven to work before it was needed
```

## 3. Real-World Usage

The organisations that survive destructive incidents share one unglamorous trait: immutable backups in a separate account, retained for weeks, and a restore someone had already performed. The organisations that do not usually had backups too — in the same account, unverified, and encrypted with a key that went down with everything else.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Independent, point-in-time copies of data in a separate blast radius |
| **Why does it exist?** | Because replication copies destruction as faithfully as it copies data |
| **Where does it belong?** | Outside the account, region and credentials that hold the original |
| **When should I use it?** | For any data you cannot regenerate — and test the restore, always |
