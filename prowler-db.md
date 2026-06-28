# Database Access Guide

Direct database access is useful for troubleshooting issues that have no UI or API
surface — verifying data integrity, checking user records, or inspecting scan state
at the SQL level.

## When to Use

- Verifying a user exists after sign-up issues
- Checking provider or scan records that appear stuck
- Confirming migration state after an API upgrade
- Investigating data inconsistencies reported by the UI

For most operational tasks (stuck scans, password resets, user deletion), prefer the
Django shell commands documented in the README Troubleshooting section — they handle
ORM relationships and cascading correctly. Use direct SQL only when you need to inspect
raw data or when the Django shell isn't sufficient.

## Connecting via ECS Exec

Connect to the **Postgres container** (not the API container):

```bash
aws ecs execute-command \
  --cluster prowler \
  --task <POSTGRES_TASK_ID> \
  --container postgres \
  --interactive \
  --command "/bin/sh"
```

Find the task ID:
```bash
aws ecs list-tasks --cluster prowler --service-name prowler-postgres --query 'taskArns[0]' --output text
```

Once inside the container, connect to the database:

```bash
psql -U prowler_admin -d prowler_db
```

No `-h` flag needed — you're already inside the Postgres container, so it connects
via the local Unix socket.

## Common Queries

### List all users
```sql
SELECT id, email, name, is_active, date_joined FROM users ORDER BY date_joined;
```

### Check provider connection state
```sql
SELECT id, provider, alias, connection_last_checked_at, connected
FROM providers
WHERE is_deleted = false
ORDER BY inserted_at;
```

### Check recent scan status
```sql
SELECT id, name, state, progress, started_at, completed_at
FROM scans
ORDER BY inserted_at DESC
LIMIT 10;
```

### Check Celery task results
```sql
SELECT task_id, task_name, status, date_done
FROM django_celery_results_taskresult
ORDER BY date_done DESC
LIMIT 10;
```

### Check database size
```sql
SELECT pg_size_pretty(pg_database_size('prowler_db')) AS db_size;
```

### List tables and row counts (approximate)
```sql
SELECT relname AS table_name, n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC
LIMIT 20;
```

## Notes

- The database user is `prowler_admin` with full access to all tables.
- The database name is `prowler_db`.
- Tables use row-level security (RLS) based on `tenant_id`. Direct SQL bypasses RLS
  since the admin user owns the tables. Be careful with UPDATE/DELETE operations.
- The database is persisted on EFS at `/var/lib/postgresql/data`. Restarting the
  Postgres service does not lose data.
