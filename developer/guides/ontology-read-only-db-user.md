# Read-only DB user for a Postgres source

> Mirrored from [`iblai/ontology`](https://github.com/iblai/ontology) · [`docs/read-only-db-user.md`](https://github.com/iblai/ontology/blob/main/docs/read-only-db-user.md). This page is generated — edit it in the repository, not here.

> How to provision the dedicated **read-only** Postgres role the ontology uses to
> connect to a source database. This assumes you already have connection details
> for the target database (host, port, database name) and an **admin/owner**
> account you can use *once* to create the role.

The ontology **never** connects to a source with an admin/superuser account. Its
[read-only safety suite](https://github.com/iblai/ontology/blob/main/docs/components/05-service-discovery.md) attempts seven write
operations (CREATE/INSERT/UPDATE/DELETE/DROP/ALTER/TRUNCATE) and requires **all
seven to be denied** before it will provision the source. A superuser passes all
writes, so the suite (correctly) marks it `FAILED` and refuses to proceed. The fix
is a dedicated role that can only `SELECT`.

## What you need

| Property | Example / placeholder | Notes |
|---|---|---|
| Host | `db.example.internal` | Hostname or IP of the Postgres server. Prefer a **stable** hostname over an IP that can change. |
| Port | `5432` | |
| Database | `appdb` | The database the ontology will read. |
| Admin user (setup only) | `postgres` / `admin` | An account allowed to `CREATE ROLE` and `GRANT`. Used **once**, never by the ontology. |
| Read-only role (created below) | `ontology_ro` | What the ontology actually connects as. |

Set these as shell variables so the commands below are copy-paste:

```bash
DB_HOST=db.example.internal
DB_PORT=5432
DB_NAME=appdb
DB_ADMIN=postgres            # admin/owner used only to create the role
RO_PW=$(openssl rand -base64 24 | tr -dc 'A-Za-z0-9' | head -c 28)   # generated read-only password
echo "Save this read-only password: $RO_PW"
```

> Any Postgres client works. If you don't have `psql` on the host, run it from a
> throwaway container: `docker run --rm -i postgres:16-alpine psql "$@"` (add
> `--network <net>` and use the DB's network hostname if the DB is only reachable
> on a private Docker/VPC network).

## 1. Create the read-only role

Run these statements **once**, connected to the target database as the admin/owner
account. Replace `appdb` with your database name and `<STRONG_PASSWORD_HERE>` with a
generated password. The block is idempotent — safe to re-run to rotate the password
or re-apply grants. This is plain SQL: paste it into any client (pgAdmin, DBeaver,
`psql`, …).

```sql
-- Create the login role if it does not already exist
DO $$
BEGIN
   IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'ontology_ro') THEN
      CREATE ROLE ontology_ro WITH LOGIN;
   END IF;
END $$;

-- Set / rotate its password
ALTER ROLE ontology_ro WITH PASSWORD '<STRONG_PASSWORD_HERE>';

-- Allow it to connect and read the public schema (repeat per schema if needed)
GRANT CONNECT ON DATABASE appdb TO ontology_ro;
GRANT USAGE ON SCHEMA public TO ontology_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ontology_ro;

-- Auto-grant SELECT on tables created in future (see caveat below)
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO ontology_ro;
```

### Scripted alternative (psql)

The same thing driven from the shell, with the password/DB name injected via psql
variables so no secret is typed inline. `-i` is required so the heredoc reaches
`psql` on stdin.

```bash
PGPASSWORD='<ADMIN_PASSWORD>' psql \
  -h "$DB_HOST" -p "$DB_PORT" -U "$DB_ADMIN" -d "$DB_NAME" \
  -v ON_ERROR_STOP=1 -v ro_pw="$RO_PW" <<'SQL'
DO $$
BEGIN
   IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'ontology_ro') THEN
      CREATE ROLE ontology_ro WITH LOGIN;
   END IF;
END $$;
ALTER ROLE ontology_ro WITH PASSWORD :'ro_pw';
GRANT CONNECT ON DATABASE :"DBNAME" TO ontology_ro;
GRANT USAGE ON SCHEMA public TO ontology_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ontology_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO ontology_ro;
SQL
```

> `:"DBNAME"` — psql substitutes the database name automatically; no need to pass
> it. If your client doesn't support that, replace it with the literal DB name.

`ALTER DEFAULT PRIVILEGES` grants `SELECT` on **future** tables too — but only for
tables created by the role that ran the statement (the admin account). If your
schema migrations run as a *different* role, re-run
`GRANT SELECT ON ALL TABLES IN SCHEMA public TO ontology_ro;` after each migration,
or run the `ALTER DEFAULT PRIVILEGES` as (or `FOR ROLE`) the migrating role.

Multiple schemas? Repeat the `USAGE` / `SELECT` / `ALTER DEFAULT PRIVILEGES` grants
for each schema (they are not recursive across schemas).

## 2. Verify (read works, writes denied)

```bash
# READ — should print the table count
PGPASSWORD="$RO_PW" psql -h "$DB_HOST" -p "$DB_PORT" -U ontology_ro -d "$DB_NAME" -tAc \
  "select count(*) from information_schema.tables where table_schema='public';"

# WRITE — must be DENIED ("permission denied for schema public")
PGPASSWORD="$RO_PW" psql -h "$DB_HOST" -p "$DB_PORT" -U ontology_ro -d "$DB_NAME" \
  -c "CREATE TABLE __ro_probe (id int);"
```

The `permission denied` message is one of the ontology safety suite's
`PERMISSION_INDICATORS`, so the suite will report this role as `PASSED`.

## 3. Wire it into the ontology

Put the secret only in `.env.mcp` (git-ignored), never in `tools.yaml`:

```bash
# .env.mcp
SRC_DB_HOST=db.example.internal
SRC_DB_PORT=5432
SRC_DB_NAME=appdb
SRC_RO_USER=ontology_ro
SRC_RO_PASSWORD=<generated-read-only-password>
```

```yaml
# config/tools.yaml
kind: source
name: my-postgres
type: postgres
host: ${SRC_DB_HOST}
port: ${SRC_DB_PORT}
database: ${SRC_DB_NAME}
user: ${SRC_RO_USER}
password: ${SRC_RO_PASSWORD}
```

Then generate the Toolbox config and validate:

```bash
ontology mcp validate       # checks the DSL
ontology mcp build          # writes config/generated/toolbox.yaml (deploy up does this too)
```

The `mcp-toolbox` container must be able to reach `SRC_DB_HOST` — ensure network
routing/firewall allows it (for a DB on a private Docker/VPC network, attach the
toolbox to that network and use the DB's network hostname). Note the Toolbox
connects to every source at startup, so keep only reachable sources in
`tools.yaml` (see [deployment.md](/developer/guides/ontology-deployment#mcp-toolbox-config-generation)).

## MySQL variant

The same principle applies to a MySQL source. Run as an admin, replacing `<db>`
with your database name and the password:

```sql
CREATE USER IF NOT EXISTS 'ontology_ro'@'%' IDENTIFIED BY '<STRONG_PASSWORD_HERE>';
GRANT SELECT ON `<db>`.* TO 'ontology_ro'@'%';
FLUSH PRIVILEGES;
```

Verify (read works, write denied — `ERROR 1142 ... command denied`):

```bash
mysql -h "$DB_HOST" -P 3306 -u ontology_ro -p"$RO_PW" -N \
  -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='<db>';"
mysql -h "$DB_HOST" -P 3306 -u ontology_ro -p"$RO_PW" \
  -e "CREATE TABLE \`<db>\`.__ro_probe (id int);"   # must be denied
```

Wire it into `tools.yaml` as a `type: mysql` source; its tools use `mysql-sql`
with `?` placeholders (not `$1`). See the `client-mysql` source and the
`list-mysql-tables` / `describe-mysql-table` tools for a working example.

## Rotate / revoke

```sql
-- rotate password (or just re-run step 1)
ALTER ROLE ontology_ro WITH PASSWORD '<NEW_PASSWORD>';

-- revoke everything and remove the role
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM ontology_ro;
REVOKE ALL ON SCHEMA public FROM ontology_ro;
REVOKE ALL ON DATABASE appdb FROM ontology_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public REVOKE SELECT ON TABLES FROM ontology_ro;
DROP ROLE ontology_ro;
```
