# RDS

> **In one line —** the same PostgreSQL or MySQL you already know, with the operational work you are bad at handled by people who do it full-time.

| | |
|---|---|
| **Full name** | Amazon Relational Database Service |
| **Category** | Managed Relational Database |
| **Architectural Layer** | Data |
| **Related notes** | [AWS Architecture](AWS%20Architecture.md) · [Database Fundamentals](../08%20-%20Databases%20and%20Data/Database%20Fundamentals.md) · [PostgreSQL](../08%20-%20Databases%20and%20Data/PostgreSQL.md) · [High Availability](../14%20-%20Scalability%20and%20Reliability/High%20Availability.md) · [Backup Strategy](../14%20-%20Scalability%20and%20Reliability/Backup%20Strategy.md) · [VPC](VPC.md) |

---

## 1. Short Definition

*What is it?*

RDS runs a **standard relational database engine** — PostgreSQL, MySQL, MariaDB, SQL Server, Oracle — on infrastructure AWS operates. Your application connects with the same driver and the same SQL. What changes is who handles backups, failover, patching and replication.

---

## 2. Problem

*What engineering problem does it solve?*

```text
Running PostgreSQL yourself means owning:
    installation and tuning
    automated backups, AND verified restores
    point-in-time recovery
    a standby, and automatic promotion when the primary dies
    read replicas
    minor and major version upgrades
    storage growth before the disk fills
    ↓
All of it difficult. All of it invisible until it fails.
All of it catastrophic when done badly.
```

> [!IMPORTANT]
> **This is the best-value trade in the whole cloud, and the reason is asymmetry.** Provisioning a server is easy; correctly restoring a database to a point in time under pressure at 3 a.m. is not. RDS costs roughly double the raw instance price, and in exchange removes the single failure mode most likely to end a company. Very few teams who paid it regret it. Many teams who did not, do.

---

## 3. Architecture Position

```text
        ┌──── VPC ─────────────────────────────────┐
        │                                           │
        │  PRIVATE SUBNET AZ-a    PRIVATE SUBNET AZ-b│
        │   ┌───────────────┐      ┌──────────────┐ │
        │   │ RDS PRIMARY   │─sync─│  STANDBY     │ │
        │   │ (writes+reads)│      │ (invisible,  │ │
        │   └───────┬───────┘      │  idle)       │ │
        │           │              └──────────────┘ │
        │           │ async                          │
        │   ┌───────▼───────┐                        │
        │   │ READ REPLICA  │  optional, for reads   │
        │   └───────────────┘                        │
        └───────────────────────────────────────────┘
                    ▲
        Application connects to ONE DNS endpoint
        Failover swaps what that name points to
```

**RDS lives inside your VPC, in private subnets, reached only by a security group rule.** It should never have a public address.

---

## 4. Multi-AZ is not a read replica

```text
MULTI-AZ STANDBY                    READ REPLICA
synchronous replication             asynchronous replication
same AZ pair, one region            same or another region
CANNOT be read from                 CAN be read from
exists only for failover            exists only for capacity
automatic promotion, ~60-120 s      manual promotion, data loss possible
this is AVAILABILITY                this is SCALING
```

> [!CAUTION]
> **These two things are constantly confused, and the confusion is expensive in both directions.** Adding read replicas does not make you survive an AZ failure — nothing promotes them automatically and they lag. Adding Multi-AZ does not add read capacity — the standby serves no queries at all. If you need both properties, you pay for both. Multi-AZ first; replicas only when reads actually saturate the primary.

---

## 5. Backups and point-in-time recovery

```text
AUTOMATED BACKUPS
    daily snapshot + continuous transaction logs
    → restore to ANY SECOND within the retention window
    → default retention is 7 days; set it to 14-35

MANUAL SNAPSHOTS
    kept until you delete them; survive instance deletion

RESTORE CREATES A NEW INSTANCE
    it does NOT roll back the existing one
    → you restore beside it, verify, then repoint the application
```

> [!IMPORTANT]
> **A backup you have never restored is a hypothesis, not a backup.** RDS makes taking backups automatic and taking *restores* still entirely your responsibility to rehearse. Do it once, deliberately, with a timer running — because the number you actually need to know is not "do we have backups" but "how long does recovery take". See [Backup Strategy](../14%20-%20Scalability%20and%20Reliability/Backup%20Strategy.md) and [Disaster Recovery](../14%20-%20Scalability%20and%20Reliability/Disaster%20Recovery.md).

> [!CAUTION]
> **Deleting an RDS instance can delete its automated backups with it.** Take a final snapshot, and keep production snapshots copied to a separate account. Backups an intruder or a bad script can delete alongside the database are not a recovery plan.

---

## 6. RDS versus Aurora

```text
RDS (standard)
    real PostgreSQL/MySQL on EBS
    predictable, portable, cheaper at small scale
    failover 60-120 s

AURORA
    AWS's reimplementation of the storage layer, wire-compatible
    storage replicated 6 ways across 3 AZs, grows automatically
    failover typically under 30 s, up to 15 read replicas
    faster, more expensive, and more locked-in

AURORA SERVERLESS v2
    scales capacity in fine increments, including down
    good for spiky or unpredictable load; a premium for steady load
```

> [!TIP]
> **Start with standard RDS PostgreSQL and move to Aurora when a specific number forces you.** Aurora is genuinely better technology, but it is more expensive, and the migration into it is easy while the migration out is not. The properties that justify it are faster failover, many read replicas, and storage that grows without intervention — if none of those are limiting you today, standard RDS is the cheaper, more portable choice.

---

## 7. Connection management

```text
Every PostgreSQL connection = a backend process ≈ 5-10 MB
    ↓
db.t4g.medium → ~170 max connections
    ↓
30 application containers × a pool of 20 = 600 connections
    ↓
"FATAL: too many connections" — and no amount of instance size fixes it properly
```

```text
Fix, in order:
    1. bound your application's pool size deliberately
    2. add PgBouncer or RDS Proxy in transaction pooling mode
    3. only then consider a larger instance
```

> [!CAUTION]
> **Connection exhaustion is the most common way a healthy RDS instance appears to be down.** Serverless and containerised applications make it worse, because each new instance opens its own pool with no global view. RDS Proxy also holds connections during a failover, which shortens the outage your users see — it earns its cost twice.

---

## 8. Real World Example

- **Almost every SaaS application's primary datastore** — a Multi-AZ PostgreSQL instance is the boring, correct centre of most architectures.
- **Migrating a legacy application** — the same SQL Server, without a DBA on staff.
- **Analytics read replicas** — heavy reporting queries kept off the primary.
- **Multi-tenant platforms** — one large instance with schema or row-level isolation.
- **Aurora Serverless for internal tools** that are idle overnight.

---

## 9. Communication and Dependencies

- **[VPC](VPC.md)**, a DB subnet group across two AZs, and a security group referencing your application's group
- **A parameter group** — engine settings live here, not in a config file you can edit
- **Secrets Manager** for credentials, with rotation
- **[IAM](IAM.md)** — optionally for authentication, always for control-plane actions
- **CloudWatch and Performance Insights** for visibility; see [Cloud Monitoring](Cloud%20Monitoring.md)
- **A connection pooler**, at any real concurrency

---

## 10. When To Use / When NOT To Use

> [!TIP]
> Use RDS for essentially every relational workload on AWS. The exception is not "we can run it cheaper ourselves" — it is a specific technical requirement RDS cannot meet.

> [!CAUTION]
> - **Not if you need an extension or engine version RDS does not offer** — that is the honest reason to self-manage on EC2
> - **Not for superuser access** — you do not get it, and some tooling expects it
> - **Not for enormous analytical scans** — that is Redshift, Athena or a warehouse
> - **Not for key-value access at extreme scale** — DynamoDB fits better
> - **Not single-AZ in production**, no matter how tempting the saving looks
> - **Not publicly accessible**, ever

---

## 11. Advantages and Disadvantages

**Advantages**
- Automated backups with point-in-time recovery
- Automatic failover to a synchronous standby
- Patching and version upgrades handled
- Read replicas created with one API call
- Encryption, monitoring and Performance Insights built in
- Standard engines — your SQL, drivers and ORM are unchanged
- Storage autoscaling, so a full disk is no longer an incident

**Disadvantages**
- **Roughly twice the cost** of the raw instance
- No superuser, no OS access, no arbitrary extensions
- Failover still takes 60–120 seconds on standard RDS
- Major version upgrades mean a maintenance window you must plan
- Configuration only through parameter groups
- Scaling writes still means a bigger instance — sharding is your problem
- Cross-region disaster recovery is extra work and extra money

---

## 12. Performance Impact

| Aspect | Impact |
|---|---|
| **Multi-AZ writes** | Synchronous replication adds 1–2 ms per commit |
| **Read replica lag** | Milliseconds normally, seconds or worse under write bursts |
| **Failover** | 60–120 s on RDS, typically under 30 s on Aurora |
| **Storage** | gp3 by default; provisioned IOPS for write-heavy workloads |
| **Connections** | Memory-bound and a hard ceiling — pool deliberately |
| **Backups** | Minimal impact on Multi-AZ; a brief I/O hit on single-AZ |

> [!TIP]
> **Enable Performance Insights and turn on slow query logging on day one.** Almost every RDS performance problem is a missing index or an unbounded query, and Performance Insights shows you which statement is consuming the database in about thirty seconds. Reaching for a larger instance before looking there is how teams pay double for a problem an index would have solved.

---

## 13. Security Considerations

> [!CAUTION]
> **`Publicly accessible: yes` is a checkbox in the creation wizard, and it is the beginning of a great many breaches.** A database with a public address is exposed to continuous credential-stuffing from the internet. RDS belongs in a private subnet with a security group that permits traffic only from your application's security group — no CIDR ranges, no exceptions for "the office IP".

- **Private subnets only**, security group to security group
- **Encryption at rest enabled at creation** — it cannot be added later without a restore
- **Force TLS** in transit (`rds.force_ssl`), and verify the certificate in your driver
- **Credentials in Secrets Manager with rotation**, or IAM database authentication
- **Deletion protection on** for production
- **Snapshots are as sensitive as the database** — never make one public, and encrypt cross-account copies
- **Audit logging** to CloudWatch for regulated data
- **Least privilege inside the database too** — your application should not connect as the owner

---

## 14. Mental Model

> [!NOTE]
> **RDS is a managed apartment rather than a house you own.**
>
> It is the same living space, and your furniture fits unchanged. When the boiler fails, someone else is contractually obliged to fix it, and there is a spare flat next door you move into automatically if yours floods. You cannot knock down a wall or install your own plumbing — that is the price. Anyone who insists on owning the house should be able to name the wall they need to move.

---

## 15. Mini Architecture Diagram

```text
                Application tier (private subnets, both AZs)
                            │
                    ┌───────▼────────┐
                    │  RDS Proxy     │  pools and holds connections
                    └───────┬────────┘
                            │  one DNS endpoint
        ┌───────────────────┼─────────────────────────┐
        │  AZ-a             │             AZ-b        │
        │  ┌────────────────▼──────┐   ┌───────────┐  │
        │  │ PRIMARY               │──►│ STANDBY   │  │
        │  │ PostgreSQL 16         │sync│ (no reads)│  │
        │  │ encrypted gp3 storage │   └───────────┘  │
        │  └──────┬────────────────┘                  │
        │         │ async                              │
        │  ┌──────▼────────┐                           │
        │  │ READ REPLICA  │  reporting queries only   │
        │  └───────────────┘                           │
        └──────────────────────────────────────────────┘
                │
        Automated backups + transaction logs → S3
        Snapshots copied to a separate account
```

---

## 16. Complete Request Flow

```text
Application opens a pooled connection via RDS Proxy over TLS
    ↓
Credentials fetched from Secrets Manager, not from an environment variable
    ↓
Write executed on the primary in AZ-a
    ↓
Committed only after the synchronous standby acknowledges  (+1-2 ms)
    ↓
Transaction log continuously shipped to S3 for point-in-time recovery
    ↓
Reporting query routed to the read replica, seconds behind — acceptable there
    ↓
─────────────── failure ───────────────
AZ-a fails
    ↓
RDS detects it and promotes the standby in AZ-b
    ↓
The endpoint's DNS record is repointed  (60-120 s total)
    ↓
RDS Proxy held client connections, so the visible outage is shorter
    ↓
A new standby is built in the background
    ↓
─────────────── human error ───────────────
A bad migration deletes a table at 14:32
    ↓
Restore to 14:31 — into a NEW instance, beside the live one
    ↓
Verify the data, extract the table or repoint the application
```

---

## 17. Key Takeaway

> [!IMPORTANT]
> Multi-AZ RDS in a private subnet, with encryption on, backups retained for weeks, a connection pooler in front and one rehearsed restore behind you — that is a production database, and it costs less than the incident it prevents.

---

## 18. Common Mistakes

- **Single-AZ in production** to save money, then losing hours to an AZ event
- **`Publicly accessible: yes`**, exposing the database to the internet
- **Confusing read replicas with Multi-AZ** and believing you have failover
- **Never rehearsing a restore**, so nobody knows the recovery time
- **Default 7-day backup retention** for data that matters
- **No connection pooler**, then "too many connections" under load
- **Scaling the instance up** instead of reading Performance Insights and adding an index
- **Forgetting encryption at creation**, which later requires a full restore
- **No deletion protection**, and no final snapshot
- **Automated backups deleted with the instance**, with no copy elsewhere
- **The application connecting as the database owner**

---

## 19. Open Source Technologies

- **PostgreSQL**, **MySQL**, **MariaDB** — the actual engines
- **PgBouncer**, **pgpool-II** — connection pooling you control
- **pgBackRest**, **WAL-G** — the backup tooling you would need if self-managing
- **pg_stat_statements** — enabled via a parameter group; the first place to look
- **Flyway**, **Liquibase**, **Alembic** — versioned schema migrations
- **pgloader**, **AWS DMS** — migrating in from elsewhere
- **Patroni** — what high availability looks like when you build it yourself, and a good argument for RDS

---

## 20. Personal Notes

*Fill this in yourself after finishing the lesson.*

- **Things I learned:**
- **Questions I still have:**
- **To research later:**

---

## 21. Workbook Exercise

- [ ] Confirm your production database is Multi-AZ, encrypted, private, and has deletion protection on.
- [ ] Restore a snapshot to a new instance and time it. Write the number down — that is your RTO.
- [ ] Count the maximum connections your application tier can open, and compare it to the instance limit.
- [ ] Open Performance Insights and find your single most expensive query.

---

# Standard Lesson Summary

## 1. Mini Architecture Map

```text
App → pooler → RDS primary (AZ-a) ──sync──► standby (AZ-b)
                     └──async──► read replica    └──► backups + logs to S3
```

## 2. Request Flow

```text
Input       ordinary SQL over a pooled TLS connection to one endpoint
    ↓
Processing  committed on the primary and its synchronous standby, logged continuously
    ↓
Output      the same results your engine always gave — plus recoverability and failover
```

## 3. Real-World Usage

The most common production database on AWS is an unremarkable Multi-AZ PostgreSQL instance, and that is the point. Teams that self-manage successfully do it for a named reason — an extension, a version, a licence. Teams that self-manage without one usually discover, during an incident, exactly which parts of RDS they had been getting for free.

## 4. Final Summary

| Question | Answer |
|---|---|
| **What is it?** | Standard relational engines with AWS operating the hard parts |
| **Why does it exist?** | Because backups, failover and recovery are difficult and unforgiving |
| **Where does it belong?** | Private subnets across two AZs, behind a connection pooler |
| **When should I use it?** | Nearly always — unless a specific technical requirement rules it out |
