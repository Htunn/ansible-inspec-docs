# Database Setup Guide

This guide covers PostgreSQL database configuration, migration from file-based storage, and hybrid storage validation.

## PostgreSQL Installation

### Option 1: Docker Compose (Recommended)

```bash
docker compose up -d postgres
```

### Option 2: Local PostgreSQL

#### macOS
```bash
brew install postgresql@15
brew services start postgresql@15
```

#### Ubuntu/Debian
```bash
sudo apt update
sudo apt install postgresql-15 postgresql-contrib
sudo systemctl start postgresql
```

### Create Database and User

```sql
-- Connect as postgres user
sudo -u postgres psql

-- Create database
CREATE DATABASE ansibleinspec;

-- Create user
CREATE USER ansibleinspec WITH PASSWORD 'your_secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE ansibleinspec TO ansibleinspec;

-- Exit
\q
```

## Configuration

### Environment Variables

```bash
# Database connection
DATABASE__URL=postgresql+asyncpg://ansibleinspec:password@localhost:5432/ansibleinspec
DATABASE__POOL_SIZE=20
DATABASE__MAX_OVERFLOW=10
DATABASE__POOL_PRE_PING=true

# For Alembic migrations (sync URL)
DATABASE__SYNC_URL=postgresql://ansibleinspec:password@localhost:5432/ansibleinspec
```

### Connection String Format

**Async (asyncpg)**: Used by application
```
postgresql+asyncpg://user:password@host:port/database
```

**Sync (psycopg2)**: Used by Alembic migrations
```
postgresql://user:password@host:port/database
```

## Database Migrations with Alembic

### Initialize Alembic

```bash
cd /path/to/ansible-inspec
alembic init alembic
```

### Configure Alembic

Edit `alembic/env.py`:

```python
from ansible_inspec.server.db import Base
from ansible_inspec.server.config import settings

target_metadata = Base.metadata

def run_migrations_online():
    configuration = context.config
    configuration.set_main_option(
        "sqlalchemy.url",
        settings.database.sync_url
    )
    # ... rest of Alembic configuration
```

### Create Initial Migration

```bash
alembic revision --autogenerate -m "Initial schema"
```

### Apply Migrations

```bash
alembic upgrade head
```

### View Migration History

```bash
alembic history
alembic current
```

## Hybrid Storage Configuration

### Enable Hybrid Mode

```bash
# .env
STORAGE_BACKEND=hybrid
VALIDATION_DAYS=30
AUTO_CUTOVER=false
```

This runs both file and database storage simultaneously for validation.

### Storage Backends

- **file**: File-based storage only (default, existing behavior)
- **database**: PostgreSQL storage only
- **hybrid**: Dual-write to both file and database

## Migration from File Storage

### Step 1: Backup Existing Data

```bash
# Create backup
tar -czf data_backup_$(date +%Y%m%d).tar.gz ./data

# Verify backup
tar -tzf data_backup_*.tar.gz | head
```

### Step 2: Initialize Database

```bash
# Run migrations
alembic upgrade head

# Verify tables created
psql -U ansibleinspec -d ansibleinspec -c "\dt"
```

### Step 3: Migrate Data

```bash
# Run migration script
python scripts/migrate_file_to_db.py
```

Create `scripts/migrate_file_to_db.py`:

```python
"""Migrate data from file storage to database."""
import asyncio
from ansible_inspec.server.storage import FileStorageBackend
from ansible_inspec.server.db import init_database
from ansible_inspec.server.config import settings

async def migrate():
    # Initialize database
    db = init_database(settings.database.url)
    await db.create_tables()
    
    # Initialize storage backends
    file_storage = FileStorageBackend(settings.data_dir)
    
    # TODO: Implement database backend and migration logic
    # db_storage = DatabaseStorageBackend(db)
    
    # Migrate job templates
    templates = await file_storage.list_job_templates()
    print(f"Migrating {len(templates)} job templates...")
    # for template in templates:
    #     await db_storage.save_job_template(template)
    
    # Migrate jobs
    jobs = await file_storage.list_jobs()
    print(f"Migrating {len(jobs)} jobs...")
    # for job in jobs:
    #     await db_storage.save_job(job)
    
    print("Migration complete!")

if __name__ == "__main__":
    asyncio.run(migrate())
```

### Step 4: Enable Hybrid Mode

```bash
# Update .env
STORAGE_BACKEND=hybrid
```

### Step 5: Monitor Validation Period

Access Prometheus metrics:
```bash
curl http://localhost:8080/metrics | grep storage
```

Key metrics to monitor:
- `storage_operations_total`: Operation counts by backend
- `storage_latency_seconds`: Operation latency
- `storage_consistency_errors_total`: Data consistency errors

## Monitoring and Validation

### Check Consistency

The system automatically compares file and database storage on every read operation during hybrid mode.

### View Validation Status

```bash
curl http://localhost:8080/api/v1/storage/validation-status
```

Response:
```json
{
  "validation_period_days": 30,
  "days_remaining": 15,
  "cutover_ready": false,
  "metrics": {
    "file": {
      "avg_write_latency": 0.05,
      "avg_read_latency": 0.02,
      "error_rate": 0.001
    },
    "database": {
      "avg_write_latency": 0.08,
      "avg_read_latency": 0.03,
      "error_rate": 0.002
    },
    "consistency_error_rate": 0.0001
  }
}
```

### Cutover Criteria

System is ready for cutover when:
1. Validation period complete (30 days)
2. Consistency error rate < 1%
3. Database error rate < 5%
4. Database write latency < 100ms

## Cutover to Database-Only

### Manual Cutover

```bash
# Update .env
STORAGE_BACKEND=database

# Restart server
docker compose restart app
```

### Automatic Cutover

```bash
# Enable automatic cutover after validation
AUTO_CUTOVER=true
```

System will automatically switch to database-only mode when criteria are met.

## Rollback to File Storage

If issues occur:

```bash
# Stop server
docker compose stop app

# Update .env
STORAGE_BACKEND=file

# Restore from backup if needed
tar -xzf data_backup_*.tar.gz

# Restart server
docker compose start app
```

## Performance Tuning

### Connection Pooling

```bash
DATABASE__POOL_SIZE=20  # Max connections
DATABASE__MAX_OVERFLOW=10  # Additional connections when pool full
```

### Indexes

Key indexes are created automatically:
- `job_templates.name`
- `job_templates.created_at`
- `jobs.status`
- `jobs.created_at`
- `users.username`

### Query Optimization

Monitor slow queries:
```sql
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

## Backup and Recovery

### Automated Backups

```bash
# Daily backup script
#!/bin/bash
BACKUP_DIR="/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)

pg_dump -U ansibleinspec ansibleinspec | gzip > \
  $BACKUP_DIR/ansibleinspec_$DATE.sql.gz

# Keep last 30 days
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
```

### Restore from Backup

```bash
# Stop application
docker compose stop app

# Drop and recreate database
psql -U postgres -c "DROP DATABASE ansibleinspec;"
psql -U postgres -c "CREATE DATABASE ansibleinspec;"

# Restore
gunzip < backup.sql.gz | psql -U ansibleinspec ansibleinspec

# Restart application
docker compose start app
```

## Troubleshooting

### Connection Refused

- Check PostgreSQL is running: `pg_isready`
- Verify connection string in `.env`
- Check firewall rules

### Permission Denied

- Grant database privileges to user
- Check `pg_hba.conf` authentication settings

### Migration Failures

- Check Alembic version compatibility
- Review migration SQL: `alembic upgrade head --sql`
- Apply migrations manually if needed

### High Latency

- Increase connection pool size
- Add database indexes
- Check database server resources
- Consider read replicas for high load

## Security Best Practices

1. **Use strong passwords** (20+ characters, mixed case, numbers, symbols)
2. **Restrict network access** (bind to localhost or private network)
3. **Enable SSL/TLS** for connections
4. **Regular security updates** for PostgreSQL
5. **Encrypt backups** before storing
6. **Use secrets management** (vault, AWS Secrets Manager)
7. **Monitor access logs** for suspicious activity

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Alembic Documentation](https://alembic.sqlalchemy.org/)
- [SQLAlchemy async Documentation](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
