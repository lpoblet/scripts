# Focus To-Do export to PostgreSQL

Exports the local data of the Focus To-Do Chromium extension (projects, tasks, subtasks,
pomodoro sessions) and loads it into PostgreSQL. See `SCHEMA.md` for the data model.

## Files

- `focus-todo-export`: reads the extension's IndexedDB from a temporary copy of the
  Chromium profile and writes JSON to `out/`. It never modifies the profile.
- `focus-todo-sqlgen`: turns `out/*.json` into `out/data.sql`, a re-runnable set of
  INSERTs (`ON CONFLICT (id) DO NOTHING`) in dependency order.
- `schema.sql`: idempotent Postgres DDL for the four tables.
- `docker-compose.yml`: optional standalone `postgres:16` container that applies
  `schema.sql` on first start.
- `SCHEMA.md`: fields, relations, and integrity results.

`out/` holds personal data and is git-ignored.

Requires Node.js; there are no npm dependencies. For the most complete export, close
Chromium first, since recent changes may not be flushed yet.

## Usage

```bash
# 1. Export (profile is auto-detected; override with --profile)
./focus-todo-export --dry-run
./focus-todo-export

# 2. Generate data.sql
./focus-todo-sqlgen

# 3a. Load into an existing container (example: the lab postgres-practice container)
docker exec postgres-practice psql -U postgres -c 'CREATE DATABASE focustodo'
docker exec -i postgres-practice psql -v ON_ERROR_STOP=1 -U postgres -d focustodo < schema.sql
docker exec -i postgres-practice psql -v ON_ERROR_STOP=1 -U postgres -d focustodo < out/data.sql

# 3b. Or use the bundled container (schema applied automatically)
POSTGRES_PORT=5433 docker compose up -d --wait
docker exec -i focus-todo-postgres psql -v ON_ERROR_STOP=1 -U focustodo -d focustodo -f /export/data.sql
```

To refresh the data after new pomodoros, repeat steps 1, 2 and the `data.sql` load. Existing
ids are skipped, so only new rows are added. Rows that changed in the extension, such as a
renamed task, keep their old values, because existing ids are not updated. To fully refresh,
truncate the tables first.
