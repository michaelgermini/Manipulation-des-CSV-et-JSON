# Chapitre 22 : Backup & Disaster Recovery — 3-2-1 Rule

## 🎯 Objectifs du chapitre

À la fin de ce chapitre, vous saurez :
- **Implémenter** 3-2-1 backup rule
- **Automatiser** backups CSV/JSON
- **Tester** recovery procedures
- **Documenter** RTO/RPO
- **Monitorer** intégrité backups

---

## 3-2-1 Backup Rule

```
✅ 3 COPIES de données
   ├─ Copy 1: Production (S3/Database)
   ├─ Copy 2: Nearline (local backup)
   └─ Copy 3: Offsite (cold storage)

✅ 2 FORMATS de stockage
   ├─ Format 1: CSV/JSON (readable)
   └─ Format 2: Parquet/Archive (compressed)

✅ 1 LOCATION offsite
   └─ Différent region/continent
```

---

## Backup Automation

### Python backup script

```python
import boto3
import shutil
import os
from datetime import datetime, timedelta

class BackupManager:
    def __init__(self, local_dir, s3_bucket, archive_bucket):
        self.local_dir = local_dir
        self.s3 = boto3.client('s3')
        self.s3_bucket = s3_bucket
        self.archive_bucket = archive_bucket
    
    def backup_daily(self):
        """Daily backup: CSV → S3 + Archive"""
        
        # 1. Local backup
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        backup_dir = f"/backups/csv_{timestamp}"
        os.makedirs(backup_dir)
        
        for file in os.listdir(self.local_dir):
            if file.endswith('.csv'):
                src = os.path.join(self.local_dir, file)
                dst = os.path.join(backup_dir, file)
                shutil.copy2(src, dst)
        
        # 2. Compress
        archive_path = f"/archives/csv_{timestamp}.tar.gz"
        shutil.make_archive(
            archive_path.replace('.tar.gz', ''),
            'gztar',
            backup_dir
        )
        
        # 3. Upload to S3 (nearline)
        self.s3.upload_file(
            archive_path,
            self.s3_bucket,
            f"daily/{timestamp}.tar.gz"
        )
        
        # 4. Upload to Glacier (offsite)
        self.s3.upload_file(
            archive_path,
            self.archive_bucket,
            f"archives/{timestamp}.tar.gz"
        )
        
        print(f"✓ Backup completed: {archive_path}")
    
    def restore(self, timestamp):
        """Restore from backup"""
        
        # Download from S3
        backup_file = f"/tmp/csv_{timestamp}.tar.gz"
        self.s3.download_file(
            self.s3_bucket,
            f"daily/{timestamp}.tar.gz",
            backup_file
        )
        
        # Extract
        restore_dir = f"/restored/csv_{timestamp}"
        shutil.unpack_archive(backup_file, restore_dir)
        
        print(f"✓ Restored to {restore_dir}")
        return restore_dir
```

### Cron job backup schedule

```bash
#!/bin/bash
# /etc/cron.d/csv-backup

# Daily backup at 2am
0 2 * * * /usr/local/bin/backup_daily.sh

# Weekly full backup at midnight Sunday
0 0 * * 0 /usr/local/bin/backup_weekly.sh

# Monthly archive to cold storage at 1am 1st
0 1 1 * * /usr/local/bin/backup_monthly.sh

# Verify backups every 6 hours
0 */6 * * * /usr/local/bin/verify_backups.sh

# Cleanup old backups (>90 days)
0 3 * * * /usr/local/bin/cleanup_old_backups.sh
```

---

## Recovery Testing

### Backup verification script

```python
def verify_backup_integrity(backup_file):
    """Vérifier intégrité du backup"""
    
    import hashlib
    import json
    
    checks = {}
    
    # 1. File exists
    if not os.path.exists(backup_file):
        checks['file_exists'] = False
        return checks
    checks['file_exists'] = True
    
    # 2. File size reasonable
    file_size = os.path.getsize(backup_file)
    checks['file_size_mb'] = file_size / 1024 / 1024
    checks['reasonable_size'] = 100 < file_size < 10_000_000_000
    
    # 3. File readable
    try:
        if backup_file.endswith('.tar.gz'):
            import tarfile
            tar = tarfile.open(backup_file)
            checks['readable'] = True
            checks['file_count'] = len(tar.getmembers())
            tar.close()
        elif backup_file.endswith('.csv'):
            df = pd.read_csv(backup_file, nrows=10)
            checks['readable'] = True
            checks['row_count_sample'] = len(df)
    except Exception as e:
        checks['readable'] = False
        checks['error'] = str(e)
    
    # 4. Checksum
    with open(backup_file, 'rb') as f:
        checksums = hashlib.sha256(f.read()).hexdigest()
    checks['sha256'] = checksums
    
    return checks

# Usage
verification = verify_backup_integrity('/backups/data_20250101.tar.gz')
print(json.dumps(verification, indent=2))
```

### Test restore procedure

```python
def test_restore(backup_timestamp):
    """Tester procédure de restore"""
    
    # 1. Restore
    restore_dir = backup_manager.restore(backup_timestamp)
    
    # 2. Validate
    restored_files = os.listdir(restore_dir)
    assert len(restored_files) > 0, "No files restored!"
    
    # 3. Integrity check
    for file in restored_files:
        if file.endswith('.csv'):
            df = pd.read_csv(os.path.join(restore_dir, file))
            assert len(df) > 0, f"{file} is empty!"
    
    # 4. Compare with production (sample)
    prod_file = f"/data/{restored_files[0]}"
    restored_file = os.path.join(restore_dir, restored_files[0])
    
    prod_md5 = hashlib.md5(open(prod_file, 'rb').read()).hexdigest()
    restored_md5 = hashlib.md5(open(restored_file, 'rb').read()).hexdigest()
    
    if prod_md5 == restored_md5:
        print("✓ Restore test PASSED - Files match!")
    else:
        print("❌ Restore test FAILED - Files differ!")
    
    return prod_md5 == restored_md5
```

---

## RTO/RPO Definition

```
RTO (Recovery Time Objective) = Maximum acceptable downtime
RPO (Recovery Point Objective) = Maximum acceptable data loss

Examples:

1. Critical production database
   ├─ RTO: 15 minutes
   ├─ RPO: 5 minutes
   └─ Strategy: Hourly snapshots + replication

2. Customer CSV exports
   ├─ RTO: 1 hour
   ├─ RPO: 1 day
   └─ Strategy: Daily backups + S3 versioning

3. Archive data
   ├─ RTO: 1 week
   ├─ RPO: 1 month
   └─ Strategy: Monthly Glacier archive
```

---

## 🎓 Exercices pratiques

### Exercice 22.1 : Backup automation
Créez script backup quotidien + archivage.

### Exercice 22.2 : Restore testing
Testez procédure de restoration.

### Exercice 22.3 : RTO/RPO
Définissez pour 3 use cases.

### Exercice 22.4 : Monitoring
Alertez si backup échoue.

---

**Voir aussi: [Chapitre 23 : Secrets Management](./23_Secrets_Management.md)**
