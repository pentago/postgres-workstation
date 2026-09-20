# Operations

## Environment

Credentials are loaded automatically by `direnv` from `pass`:
- `$PG_USER` — database user
- `$PG_PASSWORD` — database password
- `$PUID` / `$PGID` — host user mapping

The postgres container service name is `pgvector`, database is `agent-memory`.

## Dump

### Plain SQL (portable, recommended)

```bash
docker exec -e PGPASSWORD=$PG_PASSWORD pgvector \
  pg_dump -U $PG_USER -d agent-memory --no-owner -Fp \
  > agent-memory-$(date +%Y-%m-%d).sql
```

Then strip the `\restrict` / `\unrestrict` lines that some psql extensions inject:

```bash
sed -i '/^\\restrict/d; /^\\unrestrict/d' agent-memory-$(date +%Y-%m-%d).sql
```

### With INSERTs (compatible with Adminer import)

Adminer chokes on `COPY FROM stdin`. Use `--column-inserts` for row-by-row INSERTs:

```bash
docker exec -e PGPASSWORD=$PG_PASSWORD pgvector \
  pg_dump -U $PG_USER -d agent-memory --no-owner --column-inserts \
  > agent-memory-$(date +%Y-%m-%d)-inserts.sql
```

Larger file, slower restore, but can be pasted into Adminer.

### Custom format (for pg_restore)

```bash
docker exec -e PGPASSWORD=$PG_PASSWORD pgvector \
  pg_dump -U $PG_USER -d agent-memory --no-owner -Fc \
  > agent-memory-$(date +%Y-%m-%d).dump
```

## Restore

### Create the database first

```bash
docker exec -e PGPASSWORD=$PG_PASSWORD pgvector \
  psql -U $PG_USER -d postgres -c "CREATE DATABASE \"agent-memory\";"
```

Quote the hyphen so psql doesn't interpret it as a flag.

### Restore plain SQL dump

```bash
# pipe directly — no intermediate file copy needed (container is read-only)
docker exec -i -e PGPASSWORD=$PG_PASSWORD pgvector \
  psql -U $PG_USER -d agent-memory < agent-memory-2026-07-28.sql
```

### Restore custom format dump

```bash
# via a helper container on the same network
docker run --rm \
  --network postgres-memory_db \
  -v $(pwd)/agent-memory-2026-07-28.dump:/tmp/dump.dump:ro \
  -e PGPASSWORD=$PG_PASSWORD \
  --entrypoint pg_restore \
  postgres:18-alpine3.24 \
  -U $PG_USER -h postgres -d agent-memory -v /tmp/dump.dump
```

## Notes

- Container filesystem is `read_only: true` — you cannot `docker cp` into it. Always pipe via `docker exec -i`.
- Flag order matters for psql: `-d agent-memory` works when quoted, `-d agent_memory` works without quotes (underscore only).
- The `\restrict` / `\unrestrict` lines are injected by a local psql extension. They are not valid SQL and must be stripped before feeding to any non-psql tool (Adminer, pg_restore, etc.).
