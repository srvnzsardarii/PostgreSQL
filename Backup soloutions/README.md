# PostgreSQL Backup,HA and Disaster Recovery Guide
A practical reference for designing and operating backups for PostgreSQL on Ubuntu:logical and physical backups,PITR,high availability(HA) and a disaster recovery(DR) site.
> Commands assume Ubuntu 22.04/24.04 and PostgreSQL 14+(paths use version `16`;adjust to your version).
> Features that depend on a version are marked.
> Always test every procedure in a non-production environment first.
## Table of Contents
1. [Key Concepts: HA vs Backup vs DR](#1-key-concepts-ha-vs-backup-vs-dr)
2. [Requirements Gathering (RPO / RTO)](#2-requirements-gathering-rpo--rto)
3. [Backup Strategy Overview](#3-backup-strategy-overview)
4. [Logical Backup](#4-logical-backup)
5. [Physical Backup and PITR](#5-physical-backup-and-pitr)
6. [Backup Tools Comparison](#6-backup-tools-comparison)
7. [pgBackRest Setup (Recommended)](#7-pgbackrest-setup-recommended)
8. [High Availability Design](#8-high-availability-design)
9. [Disaster Recovery Site Design](#9-disaster-recovery-site-design)
10. [Reference Architecture and Backup Plan](#10-reference-architecture-and-backup-plan)
11. [Retention, Storage, Security](#11-retention-storage-security)
12. [Restore Runbooks](#12-restore-runbooks)
13. [Verification and DR Drills](#13-verification-and-dr-drills)
14. [Monitoring and Alerting](#14-monitoring-and-alerting)
15. [Requirements Checklist](#15-requirements-checklist)
16. [References](#16-references)

## 1.Key Concepts:HA vs Backup vs DR
| Concept | Purpose | Protects against | Does NOT protect against |
|---------|---------|------------------|--------------------------|
|Replication/HA|Keep service running when a node fails(automatic failover)|Server,disk,OS,network node failure|`DROP TABLE`,bad `UPDATE`,corruption,ransomware(the mistake replicates instantly)|
|Backup|Restore data to a past point in time|Human error,logical corruption,ransomware,bugs|Long downtime during restore(slow RTO)|
|DR(Disaster Recovery)|Recover service at another site|Datacenter/region loss|Logical errors,unless combined with backups or delayed replica|

Rule:replication is not a backup.
***3-2-1 rule***:3 copies of data on 2 different media/systems,ith 1 copy off-site(ideally immutable).

## 2.Requirements Gathering(RPO/RTO)
Define these with the business before choosing tools.
| Term | Meaning | Drives |
|------|---------|--------|
|RPO(Recovery Point Objective)|Maximum acceptable data loss(time)|WAL archiving frequency,sync vs async replication|
|RTO(Recovery Time Objective)|Maximum acceptable downtime|HA automation,standby readiness,backup restore speed|

Also collect: database size and daily change rate (WAL volume),growth forecast,compliance/retention rules(e.g. 7 years),encryption requirements,backup window,network bandwidth to DR,budget.

## 3.Backup Strategy Overview
Recommended baseline for production:
1. Physical backups with WAL archiving + PITR(pgBackRest)-primary recovery method.
2. Logical backups(`pg_dump`) of critical databases-secondary,portable,per object restore.
3. Streaming replication for HA,plus an async replica at the DR site.
4. Optional delayed replica(e.g. 1-4 h) for fast recovery from human error.
5. Regular,automated restore tests.

## 4.Logical Backup
Exports SQL or a custom archive of objects and data.Consistent snapshot via MVCC,does not block normal reads/writes
## 4.1.Commands
```bash
# Single database,directory format,4 parallel jobs,zstd compression
pg_dump -h localhost -U postgres -Fd -j 4 --compress=zstd:5 -f /backup/logical/appdb_$(date +%F) appdb
# Single database,custom format
pg_dump -U postgres -Fc -f /backup/logical/appdb_$(date +%F).dump appdb
# Global objects(roles,tablespaces)-NOT included in pg_dump
pg_dumpall -U postgres --globals-only > /backup/logical/globals_$(date +%F).sql
# Schema only(useful for documentation/migrations)
pg_dump -U postgres --schema-only -f appdb_schema.sql appdb
```
# Restore:
```bash
# Create target DB and restore in parallel
createdb -U postgres appdb_restored
pg_restore -U postgres -d appdb_restored -j 4 --no-owner /backup/logical/appdb_2026-10-22
# Restore single table
pg_restore -U postgres -d appdb -t payment_transactions /backup/logical/appdb.dump
# Restore globals first on a new server
psql -U postgres -f /backup/logical/globals_2026-10-22.sql
```
## 4.2.Best practices
- Always back up globals(`pg_dumpall --globals-only`) alongside database dumps.
- Run dumps from a standby to reduce load on the primary(note:long dumps on a standby can conflict with replication;tune `max_standby_streaming_delay` or `hot_standby_feedback`).
- Use `pg_dump` from the same or newer client version than the server.
- Store dump files with checksums(`sha256sum`) and encrypt them (`gpg` or `age`).
- Logical backup does not give PITR;RPO equals the dump interval.
- Large databases (> few hundred GB):dump and restore time becomes hours/days-rely on physical backups instead.

## 5. Physical Backup and PITR
Copies the data directory(files) plus the WAL(write ahead log) needed for consistency.With continuous WAL archiving you can restore to any point in time(PITR).
## 5.1.Required PostgreSQL settings
`/etc/postgresql/16/main/postgresql.conf`:
```ini
wal_level = replica              # or logical if you use logical replication
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
archive_timeout = 60             # force WAL switch at least every 60s (bounds RPO on quiet systems)
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 2GB              # safety margin for standbys
wal_compression = on
full_page_writes = on            # keep on
data_checksums                   # enable at initdb time (initdb --data-checksums) to detect corruption
```
## 5.2.pg_basebackup(simple method)
```bash
# Full base backup,tar+gzip with WAL streamed,progress shown
pg_basebackup -h primary -U replicator -D /backup/base/$(date +%F) -Ft -z -X stream -P --checkpoint=fast --wal-method=stream
# PostgreSQL 17+:incremental backups
pg_basebackup -D /backup/incr1 --incremental=/backup/base/full/backup_manifest ...
pg_combinebackup /backup/base/full /backup/incr1 -o /restore/combined
```
`pg_basebackup` alone does not manage retention,WAL archive cleanup,compression policies or verification. Use a backup manager for production.
## 5.3.Point-In-Time Recovery(concept)
```
 Full backup (Sunday)  +  WAL files archived continuously
 |------------------------------|------------ ... ----------> now
                                 ^ recover to 2026-10-22 14:31:05 (just before bad DELETE)
```
Restore base backupn then replay WAL up to `recovery_target_time`,`recovery_target_lsn`,a named restore point (`SELECT pg_create_restore_point('before_migration')`) or an XID.
## 5.4.Snapshot-based backups
Possible with LVM,ZFS,Ceph or cloud volume snapshots.Requirements: all data+WAL on a consistent snapshot (or use `pg_backup_start()`/`pg_backup_stop()` around the snapshot) and WAL archiving enabled for PITR. 
Test restores regularly.

## 7.pgBackRest Setup(Recommended)
## 7.1.Install
```bash
sudo apt update
sudo apt install -y pgbackrest
```
## 7.2.Configuration
`/etc/pgbackrest/pgbackrest.conf`(on the database host, local repository example):
```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=6
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=CHANGE_ME_LONG_RANDOM
compress-type=zst
compress-level=3
process-max=4
start-fast=y
archive-async=y
spool-path=/var/spool/pgbackrest
log-level-console=info

# Second repository: S3/off-site(3-2-1 rule).Enable object lock on the bucket.
# repo2-type=s3
# repo2-s3-endpoint=s3.example.com
# repo2-s3-bucket=pg-backups-dr
# repo2-s3-region=eu-west-1
# repo2-s3-key=ACCESS_KEY
# repo2-s3-key-secret=SECRET_KEY
# repo2-path=/pgbackrest
# repo2-retention-full=4
# repo2-cipher-type=aes-256-cbc
# repo2-cipher-pass=CHANGE_ME_ANOTHER

[main]
pg1-path=/var/lib/postgresql/16/main
```

```bash
sudo chmod 640 /etc/pgbackrest/pgbackrest.conf
sudo chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
sudo mkdir -p /var/lib/pgbackrest /var/spool/pgbackrest
sudo chown postgres:postgres /var/lib/pgbackrest /var/spool/pgbackrest
```
Set `archive_command` as in section 5.1, then restart PostgreSQL.
## 7.3.Initialize and run
```bash
sudo -u postgres pgbackrest --stanza=main stanza-create
sudo -u postgres pgbackrest --stanza=main check          # validates archiving works
sudo -u postgres pgbackrest --stanza=main backup --type=full
sudo -u postgres pgbackrest --stanza=main info
```
## 7.4.Schedule(cron for user `postgres`)
```cron
# Full backup Sunday 01:00, differential Mon-Sat 01:00
0 1 * * 0   pgbackrest --stanza=main --type=full backup
0 1 * * 1-6 pgbackrest --stanza=main --type=diff backup
# Logical dump of critical DBs at 03:00 (optional second line of defense)
0 3 * * *   /usr/local/bin/pg_logical_backup.sh
```
## 7.5.Backing up from a standby
```ini
# on the repo/standby host
[main]
pg1-path=/var/lib/postgresql/16/main
pg2-host=primary.example.com
pg2-path=/var/lib/postgresql/16/main
backup-standby=y
```
This reduces load on the primary (the primary is still contacted for start/stop backup and WAL archiving).

