# Architecture Decision Record: Aurora PostgreSQL RDS Optimization & Audit Logs Cleanup

## Context

- Aurora PostgreSQL RDS cluster (>24TB data), audit_logs table (~22TB), four additional large tables (~48% of total storage).
- Challenges: Storage cost, performance bottlenecks, inefficient vacuuming, high IOPS cost, legacy Intel-based instances, lack of partitioning, risk of transaction ID wraparound.
- Goals: Safely delete audit_logs older than 365 days (~21TB), archive to S3 Deep Glacier, validate deletions via restored snapshot, drop four large tables, optimize storage and performance, improve maintainability, and align with AWS/PostgreSQL best practices.

## Decision

### Storage Optimization & Archival
- Offload audit_logs entries older than 365 days to Amazon S3 (Deep Glacier) using Aurora UNLOAD.
- Archive monthly RDS snapshots in S3 for 12 years (Deep Glacier).
- Validate exported/deleted data against a restored snapshot before final purge.
- Drop four identified large tables to reclaim storage.
- Implement table partitioning for audit_logs and other large tables to streamline future lifecycle management.

### Aurora PostgreSQL Version Upgrade
- Upgrade Aurora PostgreSQL from 15.10 to 17.5 for improved JSON handling, performance, and feature set.

### Aurora Pricing Model Change
- Evaluate and migrate from Aurora Standard to Aurora IO-Optimized to reduce IOPS costs (currently ~35% of cluster cost).

### Instance Type Migration
- Migrate from legacy Intel-based instances to Graviton R8G for better price/performance ratio.

### Database Maintenance
- Run manual vacuum during low peak hours to clean dead tuples.
- Tune maintenance_work_mem (currently >3GB; recommended ~1GB) for optimal vacuum performance.
- Use pg_repack to reorganize bloated tables.
- Monitor transaction IDs to prevent wraparound issues.

### Query & Connection Optimization
- Introduce RDS Proxy or PgBouncer for connection pooling.
- Optimize queries and ensure proper indexing.

## Alternatives Considered

- Status Quo: Continue with current architecture, risking further cost and performance degradation.
- Archival Methods: AWS DMS, manual psql/copy, snapshot export (less scalable, less granular).
- Validation: Direct deletion vs. snapshot cross-check for safety.
- Instance Types: Intel vs. Graviton (Graviton offers better performance/cost).
- Pricing Models: Aurora Standard vs. IO-Optimized (IO-Optimized reduces IOPS cost for high-throughput workloads).

## Consequences

- Benefits: Significant storage cost reduction, improved performance, safer data lifecycle management, future-proofing via partitioning and version upgrade, better price/performance with Graviton and IO-Optimized.
- Risks: Downtime required for deletion and migration steps, risk of accidental data loss (mitigated by snapshot validation), operational complexity during migration.
- Trade-offs: Temporary downtime (up to 10 hours), phased rollout to minimize risk, need for UAT before production changes.

## Implementation Plan (Phased)

1. **Drop Large Tables**
   - Drop tables: customer_stage_24Dec2024, autocircle_multibureau_2Feb24, customer_stage_14Jun2024, customer_stage_20Jan24.
   - Monitor storage reclamation.

2. **Snapshot Restore & Audit_logs Validation**
   - Restore RDS snapshot to a staging cluster.
   - Cross-check audit_logs entries to ensure only intended data is targeted for deletion.

3. **Archive & Purge Audit_logs**
   - Create S3 bucket with Deep Glacier lifecycle policy.
   - Archive monthly RDS snapshots in S3 for 12 years.
   - Validate export integrity (row counts, sample data).
   - Delete audit_logs entries older than 365 days (drop partitions or batch delete).

4. **Manual Vacuum & Table Repacking**
   - Run manual vacuum during low peak hours.
   - Tune maintenance_work_mem for optimal vacuum.
   - Use pg_repack for bloated tables.
   - Monitor transaction IDs.

5. **Aurora PostgreSQL Version Upgrade**
   - Plan and execute upgrade from 15.10 to 17.5.
   - Test application compatibility.

6. **Instance & Pricing Model Migration**
   - Migrate to Graviton R8G instances.
   - Assess and migrate to Aurora IO-Optimized.

7. **Query & Connection Optimization**
   - Implement RDS Proxy or PgBouncer.
   - Review and optimize queries and indexes.

8. **UAT Testing & Production Rollout**
   - Perform UAT on staging cluster.
   - Roll out changes to production with rollback plan.

## Workflow Overview

```mermaid
flowchart TD
    A[Drop 4 Large Tables] --> B[Restore Snapshot & Validate audit_logs]
    B --> C[Export audit_logs >365d to S3 (Deep Glacier)]
    C --> D[Archive Monthly Snapshots in S3 (12 years)]
    D --> E[Validate Export & Delete audit_logs >365d]
    E --> F[Manual Vacuum & pg_repack]
    F --> G[Aurora Version Upgrade]
    G --> H[Instance & Pricing Model Migration]
    H --> I[Query & Connection Optimization]
    I --> J[UAT Testing & Production Rollout]
```

## References

- [Aurora PostgreSQL UNLOAD to S3](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Reference.UNLOAD.html)
- [AWS S3 Lifecycle Policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-configuration-examples.html)
- [Aurora IO-Optimized](https://aws.amazon.com/rds/aurora/pricing/)
- [Graviton Instances](https://aws.amazon.com/ec2/graviton/)
## Workflow Flowchart

Below is a textual flowchart for the Aurora PostgreSQL RDS optimization workflow:

1. Restore RDS snapshot to staging cluster
   ↓
2. Validate audit_logs entries for deletion
   ↓
3. Drop four large tables (~48% storage)
   ↓
4. Archive monthly RDS snapshots in S3 (Deep Glacier) for 12 years
   ↓
5. Delete audit_logs entries older than 365 days (drop partitions or batch delete)
   ↓
6. Run manual vacuum during low peak hours and tune maintenance_work_mem
   ↓
7. Use pg_repack to reorganize bloated tables and monitor transaction IDs
   ↓
8. Upgrade Aurora PostgreSQL from 15.10 to 17.5 and test application compatibility
   ↓
9. Migrate to Graviton R8G instances and assess migration to Aurora IO-Optimized pricing model
   ↓
10. Implement RDS Proxy or PgBouncer for connection pooling
   ↓
11. Review and optimize queries and indexes
   ↓
12. Perform UAT on staging cluster and roll out changes to production with rollback plan
- [pg_repack](https://reorg.github.io/pg_repack/)
## High-Level Architecture Diagram

Aurora PostgreSQL RDS Optimization & Audit Logs Cleanup

- AWS Aurora PostgreSQL Cluster (Primary & Replicas)
    - Large Tables (audit_logs, customer_stage_24Dec2024, autocircle_multibureau_2Feb24, customer_stage_14Jun2024, customer_stage_20Jan24)
    - Partitioned audit_logs Table
- AWS S3 (Deep Glacier)
    - Monthly RDS Snapshots (12 years retention)
- Staging Cluster (Restored Snapshot)
    - Used for validation of audit_logs entries before deletion
- Graviton R8G Instances (Post-migration)
- Aurora IO-Optimized Pricing Model (Post-migration)
- RDS Proxy / PgBouncer (Connection Pooling)
- Backend Maintenance (Vacuum, pg_repack, bloat control)
- Monitoring & Alerts (AWS CloudWatch, Transaction ID wraparound)

```
Aurora PostgreSQL Cluster
    ├── Large Tables
    ├── Partitioned audit_logs
    ├── Replicas
    ├── RDS Proxy / PgBouncer
    └── Monitoring & Alerts
         ↓
Staging Cluster (Snapshot Restore & Validation)
         ↓
Monthly Snapshots → S3 (Deep Glacier)
         ↓
Graviton R8G & IO-Optimized (Migration)
         ↓
Backend Maintenance (Vacuum, pg_repack)
```
## Validation & Cleanup SQL for Restored Snapshot

**1. Check if tables exist in restored DB:**
```sql
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename IN (
    'customer_stage_24Dec2024',
    'autocircle_multibureau_2Feb24',
    'customer_stage_14Jun2024',
    'customer_stage_20Jan24'
  );
```

**2. Sample data verification (top 5 rows):**
```sql
SELECT * FROM customer_stage_24Dec2024 LIMIT 5;
SELECT * FROM autocircle_multibureau_2Feb24 LIMIT 5;
SELECT * FROM customer_stage_14Jun2024 LIMIT 5;
SELECT * FROM customer_stage_20Jan24 LIMIT 5;
```

**3. Check table size:**
```sql
SELECT
  tablename,
  pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS size
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename IN (
    'customer_stage_24Dec2024',
    'autocircle_multibureau_2Feb24',
    'customer_stage_14Jun2024',
    'customer_stage_20Jan24'
  );
```

**4. Drop tables (after validation):**
```sql
DROP TABLE IF EXISTS customer_stage_24Dec2024 CASCADE;
DROP TABLE IF EXISTS autocircle_multibureau_2Feb24 CASCADE;
DROP TABLE IF EXISTS customer_stage_14Jun2024 CASCADE;
DROP TABLE IF EXISTS customer_stage_20Jan24 CASCADE;
```