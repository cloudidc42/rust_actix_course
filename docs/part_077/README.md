# Part 077: Database Backup and Recovery

## บทนำ

การสำรองข้อมูล (Backup) และการกู้คืนข้อมูล (Recovery) เป็นสิ่งสำคัญมากสำหรับระบบ production บทนี้จะครอบคลุมกลยุทธ์ต่างๆ สำหรับ PostgreSQL และการ automate กระบวนการ backup

## 1. pg_dump และ pg_restore

### 1.1 การใช้ pg_dump

```bash
# Backup ทั้ง database เป็น SQL format
pg_dump \
    --host=localhost \
    --port=5432 \
    --username=postgres \
    --dbname=mydb \
    --file=backup.sql \
    --verbose

# Backup เป็น custom format (binary, รองรับ parallel restore)
pg_dump \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --format=custom \
    --file=backup.dump \
    --compress=9 \
    --verbose

# Backup เฉพาะ schema (โครงสร้างไม่มีข้อมูล)
pg_dump \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --schema-only \
    --file=schema.sql

# Backup เฉพาะข้อมูล (ไม่มีโครงสร้าง)
pg_dump \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --data-only \
    --file=data.sql

# Backup เฉพาะบาง tables
pg_dump \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --table=users \
    --table=products \
    --file=partial_backup.sql

# ข้ามบาง tables (เช่น logs)
pg_dump \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --exclude-table=access_logs \
    --exclude-table=audit_logs \
    --file=backup_no_logs.sql

# Backup พร้อม compress
pg_dump \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --format=custom \
    | gzip > backup_$(date +%Y%m%d_%H%M%S).dump.gz
```

### 1.2 pg_dumpall

```bash
# Backup ทุก databases รวม roles และ settings
pg_dumpall \
    --host=localhost \
    --username=postgres \
    --file=all_databases.sql

# Backup เฉพาะ globals (roles, tablespaces)
pg_dumpall \
    --host=localhost \
    --username=postgres \
    --globals-only \
    --file=globals.sql
```

### 1.3 pg_restore

```bash
# Restore จาก custom format
pg_restore \
    --host=localhost \
    --username=postgres \
    --dbname=mydb_restored \
    --verbose \
    backup.dump

# Restore แบบ parallel (เร็วขึ้น)
pg_restore \
    --host=localhost \
    --username=postgres \
    --dbname=mydb_restored \
    --jobs=4 \
    --verbose \
    backup.dump

# Restore เฉพาะ table
pg_restore \
    --host=localhost \
    --username=postgres \
    --dbname=mydb \
    --table=users \
    backup.dump

# สร้าง database ใหม่แล้ว restore
createdb -U postgres mydb_new
pg_restore -U postgres -d mydb_new backup.dump

# Restore จาก SQL format
psql \
    --host=localhost \
    --username=postgres \
    --dbname=mydb_restored \
    --file=backup.sql
```

## 2. Automated Backup Script

### 2.1 Basic Backup Script

```bash
#!/bin/bash
# scripts/backup.sh

set -euo pipefail

# Configuration
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-mydb}"
DB_USER="${DB_USER:-postgres}"
BACKUP_DIR="${BACKUP_DIR:-/backups}"
RETENTION_DAYS="${RETENTION_DAYS:-30}"
BACKUP_FORMAT="${BACKUP_FORMAT:-custom}"

# Timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.dump"

# สร้าง backup directory ถ้าไม่มี
mkdir -p "$BACKUP_DIR"

echo "Starting backup of ${DB_NAME} at $(date)"
echo "Backup file: ${BACKUP_FILE}"

# ทำ backup
PGPASSWORD="$DB_PASSWORD" pg_dump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --format="$BACKUP_FORMAT" \
    --compress=9 \
    --file="$BACKUP_FILE" \
    --verbose

BACKUP_SIZE=$(du -sh "$BACKUP_FILE" | cut -f1)
echo "Backup complete: ${BACKUP_FILE} (${BACKUP_SIZE})"

# ลบ backups เก่า
echo "Removing backups older than ${RETENTION_DAYS} days..."
find "$BACKUP_DIR" \
    -name "${DB_NAME}_*.dump" \
    -mtime "+${RETENTION_DAYS}" \
    -delete \
    -print

echo "Backup job complete at $(date)"
```

### 2.2 Advanced Backup Script with S3

```bash
#!/bin/bash
# scripts/backup-s3.sh

set -euo pipefail

# Configuration
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-mydb}"
DB_USER="${DB_USER:-postgres}"
S3_BUCKET="${S3_BUCKET:-my-backups}"
S3_PREFIX="${S3_PREFIX:-postgres}"
RETENTION_DAYS="${RETENTION_DAYS:-30}"
GPG_KEY="${GPG_KEY:-}"  # Optional: encrypt backup

# Timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

BACKUP_FILE="${TEMP_DIR}/${DB_NAME}_${TIMESTAMP}.dump"

# Log function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# ทำ backup
log "Starting backup of ${DB_NAME}"

PGPASSWORD="$DB_PASSWORD" pg_dump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --format=custom \
    --compress=9 \
    --file="$BACKUP_FILE"

log "Database dump complete: $(du -sh $BACKUP_FILE | cut -f1)"

# Encrypt ถ้ามี GPG key
if [ -n "$GPG_KEY" ]; then
    log "Encrypting backup with GPG key ${GPG_KEY}"
    gpg --recipient "$GPG_KEY" \
        --encrypt \
        --output "${BACKUP_FILE}.gpg" \
        "$BACKUP_FILE"
    UPLOAD_FILE="${BACKUP_FILE}.gpg"
else
    UPLOAD_FILE="$BACKUP_FILE"
fi

# Upload ไป S3
S3_KEY="${S3_PREFIX}/${DB_NAME}/$(date +%Y/%m/%d)/$(basename $UPLOAD_FILE)"
log "Uploading to s3://${S3_BUCKET}/${S3_KEY}"

aws s3 cp "$UPLOAD_FILE" "s3://${S3_BUCKET}/${S3_KEY}" \
    --storage-class STANDARD_IA \
    --metadata "database=${DB_NAME},timestamp=${TIMESTAMP}"

log "Upload complete"

# ตรวจสอบว่า upload สำเร็จ
aws s3 ls "s3://${S3_BUCKET}/${S3_KEY}" || {
    log "ERROR: Upload verification failed!"
    exit 1
}

# ลบ backups เก่าใน S3
log "Cleaning up backups older than ${RETENTION_DAYS} days in S3"

CUTOFF_DATE=$(date -d "${RETENTION_DAYS} days ago" +%Y-%m-%d)
aws s3 ls "s3://${S3_BUCKET}/${S3_PREFIX}/${DB_NAME}/" --recursive \
    | awk -v cutoff="$CUTOFF_DATE" '$1 < cutoff {print $4}' \
    | while read -r key; do
        log "Deleting old backup: s3://${S3_BUCKET}/${key}"
        aws s3 rm "s3://${S3_BUCKET}/${key}"
    done

log "Backup job complete"

# ส่ง notification (optional)
if [ -n "${SLACK_WEBHOOK:-}" ]; then
    curl -s -X POST "$SLACK_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "{
            \"text\": \"✅ Database backup complete: ${DB_NAME} ($(date))\",
            \"attachments\": [{
                \"color\": \"good\",
                \"fields\": [
                    {\"title\": \"Database\", \"value\": \"${DB_NAME}\", \"short\": true},
                    {\"title\": \"S3 Key\", \"value\": \"${S3_KEY}\", \"short\": false}
                ]
            }]
        }"
fi
```

## 3. Backup Rotation

### 3.1 Rotation Strategy

```bash
#!/bin/bash
# scripts/backup-rotation.sh
# Backup strategy: keep daily for 7 days, weekly for 4 weeks, monthly for 12 months

set -euo pipefail

DB_NAME="${DB_NAME:-mydb}"
BACKUP_DIR="${BACKUP_DIR:-/backups}"
S3_BUCKET="${S3_BUCKET:-my-backups}"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DOW=$(date +%u)      # Day of week (1=Monday, 7=Sunday)
DOM=$(date +%d)      # Day of month
MONTH=$(date +%m)

BACKUP_FILE="/tmp/${DB_NAME}_${TIMESTAMP}.dump"

# ทำ backup
pg_dump -U postgres -Fc -Z9 "$DB_NAME" > "$BACKUP_FILE"

# กำหนด backup type
if [ "$DOM" = "01" ]; then
    BACKUP_TYPE="monthly"
    RETAIN=12
elif [ "$DOW" = "7" ]; then
    BACKUP_TYPE="weekly"
    RETAIN=4
else
    BACKUP_TYPE="daily"
    RETAIN=7
fi

# Upload ด้วย prefix ตาม type
S3_KEY="${DB_NAME}/${BACKUP_TYPE}/${DB_NAME}_${TIMESTAMP}.dump"
aws s3 cp "$BACKUP_FILE" "s3://${S3_BUCKET}/${S3_KEY}"

# ลบ local temp file
rm "$BACKUP_FILE"

# Cleanup old backups ของ type นี้
BACKUPS=$(aws s3 ls "s3://${S3_BUCKET}/${DB_NAME}/${BACKUP_TYPE}/" \
    | sort -r \
    | awk 'NR > '"$RETAIN"' {print $4}')

for backup in $BACKUPS; do
    echo "Deleting old ${BACKUP_TYPE} backup: $backup"
    aws s3 rm "s3://${S3_BUCKET}/${DB_NAME}/${BACKUP_TYPE}/$backup"
done

echo "Backup rotation complete. Type: ${BACKUP_TYPE}, Retained: ${RETAIN}"
```

### 3.2 Cron Jobs สำหรับ Backup

```bash
# crontab -e
# Daily backup at 2am
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# Weekly backup on Sunday at 3am (ด้วย rotation)
0 3 * * 0 /opt/scripts/backup-rotation.sh >> /var/log/backup.log 2>&1

# Monthly backup on 1st at 4am
0 4 1 * * /opt/scripts/backup-rotation.sh >> /var/log/backup.log 2>&1
```

```yaml
# k8s/cronjob-backup.yaml - Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"  # ทุกวันตอน 2am UTC
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          
          containers:
            - name: backup
              image: postgres:15-alpine
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secrets
                      key: password
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: access-key-id
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: secret-access-key
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  BACKUP_FILE="/tmp/mydb_${TIMESTAMP}.dump"
                  
                  # Dump database
                  pg_dump \
                    -h postgres \
                    -U postgres \
                    -d mydb \
                    -Fc \
                    -Z9 \
                    -f "$BACKUP_FILE"
                  
                  echo "Backup size: $(du -sh $BACKUP_FILE)"
                  
                  # Upload ไป S3
                  apk add --no-cache aws-cli
                  aws s3 cp "$BACKUP_FILE" \
                    "s3://my-backups/postgres/mydb/$TIMESTAMP.dump"
                  
                  echo "Backup complete: $TIMESTAMP"
```

## 4. Point-in-time Recovery (PITR)

### 4.1 Configure WAL Archiving

```bash
# postgresql.conf - ตั้งค่า WAL archiving
wal_level = replica
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-backups/wal/%f'
archive_timeout = 300  # Archive WAL ทุก 5 นาที

# Recovery configuration (recovery.conf หรือ postgresql.conf)
restore_command = 'aws s3 cp s3://my-backups/wal/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
```

### 4.2 สร้าง Base Backup สำหรับ PITR

```bash
# สร้าง base backup
pg_basebackup \
    --host=localhost \
    --username=replicator \
    --pgdata=/backups/base \
    --wal-method=stream \
    --checkpoint=fast \
    --progress \
    --verbose

# หรือ backup ไปยัง S3 โดยตรง (ต้องใช้ pgBackRest หรือ Barman)
```

### 4.3 pgBackRest (WAL-based Backup Tool)

```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/backups/pgbackrest
repo1-retention-full=2
repo1-retention-archive=14
log-level-console=info
log-level-file=detail

# S3 storage
repo1-type=s3
repo1-s3-bucket=my-backups
repo1-s3-endpoint=s3.amazonaws.com
repo1-s3-region=ap-southeast-1
repo1-s3-key=AKIAIOSFODNN7EXAMPLE
repo1-s3-key-secret=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[mydb]
pg1-path=/var/lib/postgresql/15/main
pg1-host=localhost
pg1-user=postgres
```

```bash
# ทำ full backup
pgbackrest --stanza=mydb backup --type=full

# ทำ incremental backup
pgbackrest --stanza=mydb backup --type=incr

# Restore ไปยัง time ที่ต้องการ
pgbackrest --stanza=mydb --target="2024-01-15 14:30:00" restore

# ดู backup list
pgbackrest --stanza=mydb info
```

## 5. Testing Backups

### 5.1 Backup Testing Script

```bash
#!/bin/bash
# scripts/test-backup.sh
# ทดสอบว่า backup ใช้ restore ได้จริง

set -euo pipefail

BACKUP_FILE="${1:-}"
TEST_DB="${2:-mydb_restore_test}"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file> [test_db_name]"
    exit 1
fi

echo "Testing backup: $BACKUP_FILE"

# สร้าง test database
dropdb --if-exists -U postgres "$TEST_DB"
createdb -U postgres "$TEST_DB"

# Restore
echo "Restoring backup..."
pg_restore \
    -U postgres \
    -d "$TEST_DB" \
    --verbose \
    "$BACKUP_FILE"

# ตรวจสอบ row counts
echo "Verifying data integrity..."
psql -U postgres -d "$TEST_DB" << 'EOF'
-- ตรวจสอบ tables ที่ restore มา
SELECT 
    table_name,
    (SELECT COUNT(*) FROM information_schema.columns WHERE table_name = t.table_name) AS column_count
FROM information_schema.tables t
WHERE table_schema = 'public'
ORDER BY table_name;

-- ตรวจสอบ record counts
SELECT 
    schemaname,
    tablename,
    n_live_tup AS estimated_rows
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
EOF

echo "Restore test complete!"

# Cleanup
echo "Cleaning up test database..."
dropdb -U postgres "$TEST_DB"

echo "✅ Backup is valid and restorable!"
```

### 5.2 Automated Backup Verification

```rust
// src/backup_verify.rs
use sqlx::PgPool;
use std::collections::HashMap;

struct BackupVerifier {
    pool: PgPool,
    expected_tables: Vec<String>,
}

impl BackupVerifier {
    async fn verify(&self) -> anyhow::Result<VerificationResult> {
        let mut result = VerificationResult::default();
        
        // ตรวจสอบว่า tables ครบ
        let actual_tables: Vec<String> = sqlx::query_scalar!(
            "SELECT table_name FROM information_schema.tables 
             WHERE table_schema = 'public' 
             ORDER BY table_name"
        )
        .fetch_all(&self.pool)
        .await?;
        
        for expected in &self.expected_tables {
            if !actual_tables.contains(expected) {
                result.missing_tables.push(expected.clone());
            }
        }
        
        // ตรวจสอบ row counts
        for table in &actual_tables {
            let count: i64 = sqlx::query_scalar(
                &format!("SELECT COUNT(*) FROM {}", table)
            )
            .fetch_one(&self.pool)
            .await?;
            
            result.table_counts.insert(table.clone(), count);
        }
        
        // ตรวจสอบ constraints
        let constraint_violations = sqlx::query!(
            r#"
            SELECT conname, contype
            FROM pg_constraint
            WHERE NOT convalidated
            "#
        )
        .fetch_all(&self.pool)
        .await?;
        
        result.is_valid = result.missing_tables.is_empty() 
            && constraint_violations.is_empty();
        
        Ok(result)
    }
}

#[derive(Default)]
struct VerificationResult {
    is_valid: bool,
    missing_tables: Vec<String>,
    table_counts: HashMap<String, i64>,
    errors: Vec<String>,
}
```

## 6. Migration Safety

### 6.1 Safe Migration Patterns

```sql
-- migrations/001_initial.sql

-- ✅ Safe: เพิ่ม column ที่มี default value
ALTER TABLE users ADD COLUMN IF NOT EXISTS 
    last_login TIMESTAMP DEFAULT NOW();

-- ✅ Safe: เพิ่ม index แบบ CONCURRENT (ไม่ lock table)
CREATE INDEX CONCURRENTLY IF NOT EXISTS 
    idx_users_email ON users(email);

-- ✅ Safe: เพิ่ม NOT NULL ด้วยวิธีปลอดภัย
-- Step 1: เพิ่ม column ที่ allow NULL ก่อน
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Step 2: Populate ข้อมูล
UPDATE users SET phone = 'unknown' WHERE phone IS NULL;

-- Step 3: เพิ่ม constraint หลังจาก fill ข้อมูลแล้ว
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
ALTER TABLE users ALTER COLUMN phone SET DEFAULT 'unknown';

-- ❌ Unsafe: เพิ่ม column NOT NULL โดยไม่มี default (lock ทั้ง table!)
-- ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;

-- ✅ Safe: Rename column ด้วยวิธี backward compatible
-- Step 1: เพิ่ม column ใหม่
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Step 2: Copy data
UPDATE users SET full_name = name;

-- Step 3: Update app ให้ใช้ column ใหม่ก่อน
-- (deploy app version ที่รองรับทั้ง column เก่าและใหม่)

-- Step 4: Drop column เก่า (ใน migration ถัดไป)
-- ALTER TABLE users DROP COLUMN name;
```

### 6.2 Migration กับ sqlx

```rust
// migrations/001_create_users.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- migrations/002_add_phone.sql
-- เพิ่ม column แบบปลอดภัย
ALTER TABLE users ADD COLUMN IF NOT EXISTS 
    phone VARCHAR(20);

-- migrations/003_make_phone_required.sql
-- ทำให้ NOT NULL หลังจากมีข้อมูลแล้ว
UPDATE users SET phone = '' WHERE phone IS NULL;
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
ALTER TABLE users ALTER COLUMN phone SET DEFAULT '';
```

```rust
// src/db.rs
use sqlx::{PgPool, migrate};

pub async fn run_migrations(pool: &PgPool) -> anyhow::Result<()> {
    // ตรวจสอบ pending migrations
    let migrator = sqlx::migrate!("./migrations");
    
    println!("Applied migrations: {:?}", 
        migrator.migrations.iter()
            .map(|m| &m.description)
            .collect::<Vec<_>>()
    );
    
    // Run with lock (ป้องกัน concurrent migration)
    migrator.run(pool).await
        .map_err(|e| anyhow::anyhow!("Migration error: {}", e))?;
    
    Ok(())
}

pub async fn check_pending_migrations(pool: &PgPool) -> anyhow::Result<Vec<String>> {
    let migrator = sqlx::migrate!("./migrations");
    let applied = migrator.get_applied_migrations(pool).await?;
    
    let pending: Vec<String> = migrator.migrations
        .iter()
        .filter(|m| !applied.iter().any(|a| a.version == m.version))
        .map(|m| format!("{}: {}", m.version, m.description))
        .collect();
    
    Ok(pending)
}
```

## 7. Disaster Recovery Checklist

### 7.1 Recovery Time Objectives

```
RTO (Recovery Time Objective): เวลาสูงสุดที่ยอมรับ downtime ได้
  - Critical: < 1 ชั่วโมง
  - Important: < 4 ชั่วโมง
  - Standard: < 24 ชั่วโมง

RPO (Recovery Point Objective): ข้อมูลสูงสุดที่ยอมสูญเสียได้
  - Critical: < 1 นาที (ต้องใช้ streaming replication)
  - Important: < 1 ชั่วโมง (hourly backups)
  - Standard: < 24 ชั่วโมง (daily backups)
```

### 7.2 Disaster Recovery Checklist Document

```markdown
## Pre-Disaster Preparation

- [ ] Automated daily backups configured
- [ ] Backup retention policy in place (daily 7d, weekly 4w, monthly 12m)
- [ ] Backups stored in separate region/cloud
- [ ] Backup encryption configured
- [ ] Backup restoration tested within last 30 days
- [ ] Recovery runbook documented and accessible
- [ ] On-call rotation established
- [ ] Monitoring alerts configured

## During Disaster

1. [ ] Assess scope of data loss/corruption
2. [ ] Notify stakeholders
3. [ ] Identify last known good backup
4. [ ] Provision recovery environment
5. [ ] Begin restore process
6. [ ] Verify data integrity after restore
7. [ ] Run application smoke tests
8. [ ] Update DNS/load balancer to point to recovery environment
9. [ ] Monitor for issues

## Post-Disaster

- [ ] Document timeline of events
- [ ] Identify root cause
- [ ] Implement preventive measures
- [ ] Review and update DR plan
- [ ] Conduct post-mortem
```

## 8. Practical: Backup Automation Script

### 8.1 Complete Backup Automation

```bash
#!/bin/bash
# scripts/backup-automation.sh
# สำหรับ crontab หรือ systemd timer

set -euo pipefail

# ====================================
# Configuration
# ====================================
DB_HOST="${DB_HOST:?DB_HOST required}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:?DB_NAME required}"
DB_USER="${DB_USER:-postgres}"
DB_PASSWORD="${DB_PASSWORD:?DB_PASSWORD required}"

S3_BUCKET="${S3_BUCKET:?S3_BUCKET required}"
S3_PREFIX="${S3_PREFIX:-backups/postgres}"

BACKUP_RETENTION_DAYS="${BACKUP_RETENTION_DAYS:-30}"
MAX_BACKUP_SIZE_MB="${MAX_BACKUP_SIZE_MB:-5000}"

# Alerting (optional)
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
EMAIL_TO="${EMAIL_TO:-}"

# ====================================
# Functions
# ====================================

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a /var/log/backup.log
}

error() {
    log "ERROR: $*" >&2
}

notify_success() {
    local msg="$1"
    log "SUCCESS: $msg"
    
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"✅ $msg\"}" || true
    fi
}

notify_failure() {
    local msg="$1"
    error "$msg"
    
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST "$SLACK_WEBHOOK" \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"❌ $msg\"}" || true
    fi
    
    exit 1
}

# ====================================
# Main
# ====================================

log "=== Backup Job Starting ==="
log "Database: ${DB_NAME}@${DB_HOST}:${DB_PORT}"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
TEMP_DIR=$(mktemp -d)
BACKUP_FILE="${TEMP_DIR}/${DB_NAME}_${TIMESTAMP}.dump"

# Cleanup on exit
cleanup() {
    log "Cleaning up temporary files..."
    rm -rf "$TEMP_DIR"
}
trap cleanup EXIT

# 1. Perform backup
log "Step 1: Creating database dump..."
PGPASSWORD="$DB_PASSWORD" pg_dump \
    --host="$DB_HOST" \
    --port="$DB_PORT" \
    --username="$DB_USER" \
    --dbname="$DB_NAME" \
    --format=custom \
    --compress=9 \
    --file="$BACKUP_FILE" || notify_failure "pg_dump failed"

BACKUP_SIZE_BYTES=$(stat -c%s "$BACKUP_FILE")
BACKUP_SIZE_MB=$((BACKUP_SIZE_BYTES / 1024 / 1024))
log "Backup size: ${BACKUP_SIZE_MB}MB"

# ตรวจสอบขนาด backup
if [ "$BACKUP_SIZE_MB" -gt "$MAX_BACKUP_SIZE_MB" ]; then
    error "WARNING: Backup size (${BACKUP_SIZE_MB}MB) exceeds limit (${MAX_BACKUP_SIZE_MB}MB)"
fi

if [ "$BACKUP_SIZE_BYTES" -eq 0 ]; then
    notify_failure "Backup file is empty!"
fi

# 2. Verify backup
log "Step 2: Verifying backup integrity..."
PGPASSWORD="$DB_PASSWORD" pg_restore \
    --list \
    "$BACKUP_FILE" > /dev/null || notify_failure "Backup verification failed"

log "Backup verification passed"

# 3. Upload ไปยัง S3
S3_KEY="${S3_PREFIX}/${DB_NAME}/$(date +%Y/%m)/${DB_NAME}_${TIMESTAMP}.dump"
log "Step 3: Uploading to s3://${S3_BUCKET}/${S3_KEY}..."

aws s3 cp "$BACKUP_FILE" "s3://${S3_BUCKET}/${S3_KEY}" \
    --storage-class STANDARD_IA \
    --metadata "database=${DB_NAME},timestamp=${TIMESTAMP},size=${BACKUP_SIZE_BYTES}" \
    || notify_failure "S3 upload failed"

# Verify upload
UPLOADED_SIZE=$(aws s3 ls "s3://${S3_BUCKET}/${S3_KEY}" | awk '{print $3}')
if [ "$UPLOADED_SIZE" != "$BACKUP_SIZE_BYTES" ]; then
    notify_failure "S3 upload size mismatch! Local: ${BACKUP_SIZE_BYTES}, S3: ${UPLOADED_SIZE}"
fi

log "Upload verified: ${UPLOADED_SIZE} bytes"

# 4. Cleanup old backups
log "Step 4: Cleaning up backups older than ${BACKUP_RETENTION_DAYS} days..."
CUTOFF=$(date -d "${BACKUP_RETENTION_DAYS} days ago" +%Y-%m-%d)

DELETED=0
aws s3 ls "s3://${S3_BUCKET}/${S3_PREFIX}/${DB_NAME}/" --recursive \
    | awk -v cutoff="$CUTOFF" '$1 < cutoff {print $4}' \
    | while read -r key; do
        log "Deleting old backup: $key"
        aws s3 rm "s3://${S3_BUCKET}/$key"
        DELETED=$((DELETED + 1))
    done

log "Cleanup complete"

# 5. Update backup metadata
log "Step 5: Recording backup metadata..."

psql -h "$DB_HOST" -U "$DB_USER" "$DB_NAME" << SQL 2>/dev/null || true
INSERT INTO backup_history (timestamp, s3_key, size_bytes, status)
VALUES ('$TIMESTAMP', '$S3_KEY', '$BACKUP_SIZE_BYTES', 'success')
ON CONFLICT DO NOTHING;
SQL

# Done!
log "=== Backup Job Complete ==="
notify_success "Database backup complete: ${DB_NAME} (${BACKUP_SIZE_MB}MB) -> s3://${S3_BUCKET}/${S3_KEY}"
```

### 8.2 Backup History Table

```sql
-- migrations/backup_history.sql
CREATE TABLE IF NOT EXISTS backup_history (
    id BIGSERIAL PRIMARY KEY,
    timestamp VARCHAR(20) NOT NULL,
    s3_key TEXT NOT NULL,
    size_bytes BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'success',
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_backup_history_timestamp ON backup_history(timestamp DESC);
```

### 8.3 Restore Script

```bash
#!/bin/bash
# scripts/restore.sh

set -euo pipefail

S3_BUCKET="${S3_BUCKET:?S3_BUCKET required}"
BACKUP_KEY="${1:?Usage: restore.sh <s3-backup-key>}"
TARGET_DB="${2:-mydb_restored}"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

BACKUP_FILE="${TEMP_DIR}/restore.dump"

log "Downloading backup from s3://${S3_BUCKET}/${BACKUP_KEY}..."
aws s3 cp "s3://${S3_BUCKET}/${BACKUP_KEY}" "$BACKUP_FILE"

log "Creating target database: ${TARGET_DB}"
dropdb --if-exists -U postgres "$TARGET_DB"
createdb -U postgres "$TARGET_DB"

log "Restoring backup..."
pg_restore \
    -U postgres \
    -d "$TARGET_DB" \
    --jobs=4 \
    --verbose \
    "$BACKUP_FILE"

log "Verifying restore..."
TABLES=$(psql -U postgres -d "$TARGET_DB" -t \
    -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public'")
log "Tables restored: ${TABLES}"

log "✅ Restore complete! Database: ${TARGET_DB}"
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **pg_dump/pg_restore** - backup และ restore PostgreSQL
2. **Automated scripts** - backup อัตโนมัติพร้อม S3 upload
3. **Backup rotation** - daily/weekly/monthly retention
4. **PITR** - Point-in-time Recovery ด้วย WAL archiving
5. **Testing backups** - ตรวจสอบว่า backup ใช้ได้จริง
6. **Migration safety** - zero-downtime migrations
7. **Disaster recovery** - checklist และ procedures
8. **Complete automation** - end-to-end backup solution

---

[⬅️ Part 076: Kubernetes Basics](../part_076/README.md) | [➡️ Part 078: API Documentation with OpenAPI](../part_078/README.md)
