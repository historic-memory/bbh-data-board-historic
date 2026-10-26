---
title: 'WO-10: Plugin business_generator — migrasi registry + generator kode bisnis'
description: 'Plugin baru yang terikat ke koneksi default untuk menangani: (1) migrasi tabel registri (dbms, dbms_secrets, db_registry, audit), (2) CLI console provisioning dbms add/list/rm & create-branch via --key, (3) generator scaffolding kode bisnis per-entity, (4) HTTP admin API /api/admin/dbms. Menerapkan prinsip user: "tabel registry milik database utama → wajib lewat migrasi."'
status: draft
created: "2026-10-26"
updated: "2026-10-26"
revision: 1
---
# WO-10: Plugin `business_generator` — Migrasi Registry + Generator Kode Bisnis

## Masalah yang diselesaikan
Tabel registri (`dbms`, `dbms_secrets`, `db_registry`, `db_registry_audit`) **sebelumnya dibuat via `EnsureRegistrySchema()`** (DDL idempotent di bootstrap), bukan lewat migrasi versioned. Ini masalah karena:
1. Tabel tak terlihat di `migrate status`.
2. Tidak ada tracking versi migrasi untuk infrastruktur inti.
3. Bertentangan dengan prinsip user: *"tabel registry milik database utama → harus lewat migrasi."*

Solusi: **plugin baru `business_generator`** yang:
- Terikat ke koneksi `default` (dapat melewati guard D-13).
- Menyediakan migrasi versioned DDL registri.
- Membawa semua fitur CLI `bin/business` (provisioning) sebagai console commands.
- Menyiapkan generator scaffolding kode bisnis.
- Menyediakan HTTP admin API untuk kelola registry.

## Keputusan Desain
| # | Keputusan | Nilai |
|---|---|---|
| E-1 | Bound connection | `default` — satu saja; guard D-13 mengizinkan core bound on default |
| E-2 | Sumber DDL | Semua DDL bergerak dari Go constants di `registry_schema.go` → dipakai juga oleh migrasi SQL |
| E-3 | EnsureRegistrySchema | **DILEPAS total** (tanpa fallback); setelah migrasi jalan, tabel selalu ada |
| E-4 | CLI | Pindah seluruh perintah dari `bin/business` ke `ConsoleCommands()` plugin → tidak perlu binary terpisah lagi |
| E-5 | Generator | `generate entity <name> --fields field:type,...` → scaffold model/service/handler/migration per-entity |
| E-6 | Admin API | `GET/POST/PUT/DELETE /api/admin/dbms` + `/api/admin/registry` untuk CRUD server & koneksi logis |

## Scope & Out of Scope
### In Scope
- [ ] Migrasi DDL registri (`dbms`, `dbms_secrets`, `db_registry`, audit + trigger)
- [ ] Pindahkan DDL constants dari `internal/db/registry_schema.go` ke `plugins/business_generator/migrations/postgres/`
- [ ] Hapus `EnsureRegistrySchema()` dari `bootstrap.go` dan `console/migrate.go`
- [ ] Console commands: `dbms add/list/rm`, `create-branch`, `ping`, `migrate up`, `list-status`
- [ ] Skeleton generator: `generate entity ...` (model, service, handler, route stub, migration file)
- [ ] Admin API endpoints: DBMS registry management

### Out of Scope
- Refactoring `bin/business` (bisa tetap ada tapi deprecate; atau hapus total di WO lain)
- Complex generation logic (hanya scaffolding skeleton)
- Database sharding / replication
- KMS integration

## Task 1: Struktur plugin & bootstrap
```
plugins/business_generator/
    plugin.go             # ID, ConnectionName, RegisterServices, Routes, ConsoleCommands
    migrations/postgres/
        2026_10_26_150000_create_db_registry.down.sql
        2026_10_26_150000_create_db_registry.up.sql
    handlers/
        admin.go          # CRUD dbms + registry via HTTP
        admin_handler.go  # struct with ServiceDeps
    services/
        admin_service.go  # DBMS operations (add, list, rm with guard)
        registry_ops.go   # Registry CRUD ops
    generators/
        entity.go         # generate entity command handler
        template.go       # Go templates for scaffolding
```

## Task 2: Migrasi DDL
File: `plugins/business_generator/migrations/postgres/2026_10_26_150000_create_db_registry.up.sql`

```sql
-- SERVER: registered once, referenced by key
CREATE TABLE IF NOT EXISTS dbms (
    id             uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    key            varchar(100) NOT NULL UNIQUE,
    driver         varchar(50)  NOT NULL DEFAULT 'postgres',
    host           varchar(255) NOT NULL,
    port           int          NOT NULL DEFAULT 5432,
    user_name      varchar(150) NOT NULL,
    sslmode        varchar(50)  NOT NULL DEFAULT 'verify-full',
    max_open_conns int          NOT NULL DEFAULT 25,
    max_idle_conns int          NOT NULL DEFAULT 5,
    label          varchar(150),
    status         varchar(20)  NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    created_at     timestamptz  NOT NULL DEFAULT now(),
    updated_at     timestamptz  NOT NULL DEFAULT now()
);
CREATE INDEX idx_dbms_status ON dbms(status);

-- SECRETS: one row per server, AES-GCM envelope
CREATE TABLE IF NOT EXISTS dbms_secrets (
    dbms_id      uuid PRIMARY KEY REFERENCES dbms(id) ON DELETE CASCADE,
    password_enc text NOT NULL,
    key_version  int  NOT NULL DEFAULT 1,
    rotated_at   timestamptz,
    updated_at   timestamptz NOT NULL DEFAULT now()
);
REVOKE ALL ON dbms_secrets FROM PUBLIC;

-- REGISTRY: logical connections pointing to a dbms_key, NO credentials
CREATE TABLE IF NOT EXISTS db_registry (
    id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    name          varchar(100) NOT NULL UNIQUE,
    role          varchar(20)  NOT NULL CHECK (role IN ('main','business','branch')),
    dbms_key      varchar(100) NOT NULL REFERENCES dbms(key) ON UPDATE CASCADE,
    database_name varchar(150) NOT NULL,
    label         varchar(150),
    status        varchar(20)  NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
    created_at    timestamptz  NOT NULL DEFAULT now(),
    updated_at    timestamptz  NOT NULL DEFAULT now()
);
CREATE INDEX idx_db_registry_role_status ON db_registry(role, status);

-- AUDIT
CREATE TABLE IF NOT EXISTS db_registry_audit (
    id             bigserial PRIMARY KEY,
    registry_id    uuid,
    name           varchar(100),
    action         varchar(20) NOT NULL CHECK (action IN ('create','update','suspend','delete','rotate')),
    actor          varchar(150) DEFAULT current_user,
    changed_fields text[],
    at             timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_db_registry_audit_at ON db_registry_audit(at DESC);

-- TRIGGERS
CREATE OR REPLACE FUNCTION trg_db_registry_audit_fn() RETURNS trigger AS $$
DECLARE
    changed text[] := '{}';
    col text;
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO db_registry_audit (registry_id, name, action, actor, changed_fields)
        VALUES (OLD.id, OLD.name, 'delete', current_user, '{}');
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO db_registry_audit (registry_id, name, action, actor, changed_fields)
        VALUES (NEW.id, NEW.name, 'create', current_user, '{}');
    ELSE
        FOR col IN SELECT jsonb_object_keys(to_jsonb(NEW)) LOOP
            IF NOT (col IN ('updated_at', 'rotated_at')) THEN
                IF (to_jsonb(OLD) -> col) IS DISTINCT FROM (to_jsonb(NEW) -> col) THEN
                    changed := array_append(changed, col);
                END IF;
            END IF;
        END LOOP;
        INSERT INTO db_registry_audit (registry_id, name, action, actor, changed_fields)
        VALUES (
            NEW.id, NEW.name,
            CASE WHEN 'status' = ANY(changed) AND NEW.status = 'suspended' THEN 'suspend' ELSE 'update' END,
            current_user, changed);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION trg_dbms_secrets_audit_fn() RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO db_registry_audit (registry_id, action, actor, changed_fields)
        VALUES (OLD.dbms_id, 'delete', current_user, '{}');
    ELSE
        INSERT INTO db_registry_audit (registry_id, action, actor, changed_fields)
        VALUES (NEW.dbms_id, 'rotate', current_user, '{password_enc}');
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_db_registry_audit ON db_registry;
CREATE TRIGGER trg_db_registry_audit AFTER INSERT OR UPDATE OR DELETE ON db_registry
FOR EACH ROW EXECUTE FUNCTION trg_db_registry_audit_fn();

DROP TRIGGER IF EXISTS trg_dbms_secrets_audit ON dbms_secrets;
CREATE TRIGGER trg_dbms_secrets_audit AFTER INSERT OR UPDATE OR DELETE ON dbms_secrets
FOR EACH ROW EXECUTE FUNCTION trg_dbms_secrets_audit_fn();

-- Seed default servers (idempotent)
INSERT INTO dbms (key, driver, host, port, user_name, sslmode, label, status)
VALUES ('main-dbms', 'postgres', 'postgres', 5432, 'postgres', 'disable', 'Main framework database', 'active')
ON CONFLICT (key) DO NOTHING;

INSERT INTO dbms (key, driver, host, port, user_name, sslmode, label, status)
VALUES ('business-dbms', 'postgres', 'postgres-business', 5432, 'postgres', 'disable', 'Server Cabang', 'active')
ON CONFLICT (key) DO NOTHING;
```

## Task 3: Drop `EnsureRegistrySchema` callers
Files to modify:
- `internal/app/bootstrap.go`: Remove `db.EnsureRegistrySchema(gdb)` call (~line 65)
- `internal/console/migrate.go`: Remove `ensureRegistrySchema()` function (~line 315)
- `internal/businesscli/registry.go`: Remove `ensureRegistrySchema(main)` wrapper (~line 18)

After removal, all registry access will assume tables exist (they should always be there after migration runs). Error clear: `relation "dbms" does not exist` → must run migration first.

## Task 4: Plugin boilerplate
```go
// plugins/business_generator/plugin.go

package business_generator

import (
    "github.com/gin-gonic/gin"
    "github.com/spf13/cobra"
    "gorm.io/gorm"
    
    "github.com/rolldone/donisgo/internal/plugins"
    bizhandler "github.com/rolldone/donisgo/plugins/business_generator/handlers"
    bizerservice "github.com/rolldone/donisgo/plugins/business_generator/services"
)

type Plugin struct {
    db   *gorm.DB
    svc  *bizerservice.AdminService
}

func New() plugins.Plugin { return &Plugin{} }

func (p *Plugin) ID() string { return "business_generator" }

// ConnectionName implements ConnectionBinder — binds to default.
func (p *Plugin) ConnectionName() string { return "default" }

func (p *Plugin) RegisterServices(deps plugins.ServiceDeps) error {
    p.db = deps.DB  // uses default connection
    p.svc = bizerservice.NewAdminService(p.db)
    return nil
}

func (p *Plugin) RegisterMiddleware() []plugins.MiddlewareDescriptor {
    return nil
}

func (p *Plugin) RegisterRoutes(_ *gin.Engine, _, api *gin.RouterGroup) error {
    h := bizhandler.NewAdminHandler(p.svc)
    group := api.Group("/admin")
    group.GET("/dbms", h.ListDBMS)
    group.POST("/dbms", h.AddDBMS)
    group.PUT("/dbms/:key", h.UpdateDBMS)
    group.DELETE("/dbms/:key", h.RemoveDBMS)
    group.GET("/registry", h.ListRegistry)
    return nil
}

// ConsoleCommands returns CLI provisioning commands.
func (p *Plugin) ConsoleCommands() []*cobra.Command {
    return []*cobra.Command{
        cmdDBMSAdd(),
        cmdDBMSList(),
        cmdDBMSRM(),
        cmdCreateBranch(),
        cmdGenerateEntity(),
    }
}

func (p *Plugin) Seed() error {
    return nil  // no-op in phase 1
}
```

## Task 5: Registration in main files
Update `cmd/server/main.go` and `cmd/console/main.go`:
```go
// Add to plugins registration loop or manual registration:
import _ "github.com/rolldone/donisgo/plugins/business_generator"
```

This auto-registers the plugin so its routes, services, and console commands are available without explicit import elsewhere.

## Task 6: CLI implementation detail
### `dbms add --key --host --port --user --sslmode --label --password-stdin`
- Validate key regex `^[a-z0-9-]+$`
- Upsert: INSERT/UPDATE into `dbms`
- If new key: INSERT encrypted password to `dbms_secrets` via `internal/secrets.EncryptBlob()`
- Read password from stdin (`--password-stdin`) or interactive prompt
- Print summary: `dbms "<key>" registered at <host>:<port>`
- Audit: trigger fills `db_registry_audit` automatically

### `dbms list`
```
KEY            DRIVER    HOST               PORT  USER      SSLMODE  #CONNECTIONS
business-dbms  postgres  postgres-business  5432  postgres  disable  4
main-dbms      postgres  postgres           5432  postgres  disable  1
```

### `dbms rm --key X`
- Guard: check if `EXISTS SELECT 1 FROM db_registry WHERE dbms_key='X'`
- If yes → `error: cannot remove "X" — still referenced by N branches`
- If no → `DELETE FROM dbms_secrets WHERE dbms_id=(SELECT id FROM dbms WHERE key='X')`; `DELETE FROM dbms WHERE key='X'`

### `create-branch --key --branch-key --branch-name [--db-name]`
- Resolve `dbms` entry via `--key`
- Open maintenance conn to that server (`database=postgres`)
- `CREATE DATABASE <name>` (idempotent)
- INSERT into `db_registry` (only `dbms_key` + `database_name`, NO host/port/user/password)
- Auto-migrate: open new conn to branch DB, apply `plugins/business/migrations/postgres/`
- Print summary

## Task 7: Code generator skeleton
```bash
bin/console generate entity User --fields "name:string,email:string,role:string:enum:admin,user,guest"
```
Generates:
- `plugins/business/models/<entity>.go` — GORM model struct with fields
- `plugins/business/services/entity_<singular>.go` — basic CRUD service methods
- `plugins/business/handlers/entity_<plural>.go` — Gin handlers (list/get/create/update/delete)
- `plugins/business/migrations/postgres/<timestamp>_create_<plural>_<table_names>.up.sql` — CREATE TABLE
- `...down.sql` — DROP TABLE IF EXISTS

Templates in `generators/template.go` using Go's `text/template`. Scaffolding only — business logic left to developer.

## Acceptance
- [ ] `go build ./...` + `go test ./...` lulus
- [ ] `./bin/console migrate up` creates `dbms`, `dbms_secrets`, `db_registry`, `db_registry_audit` in main DB
- [ ] `./bin/console migrate status` shows registry migration applied
- [ ] `./bin/console dbms add --key local-dbms --host localhost --user postgres --password-stdin` registers server + secrets
- [ ] `./bin/console dbms list` displays table with #connections count
- [ ] `./bin/console dbms rm --key wrong` refused if referenced by `db_registry`
- [ ] `./bin/console create-branch --key local-dbms --branch-key surabaya --branch-name "Cabang Surabaya"` creates DB + registry + auto-migrates
- [ ] `curl http://localhost:<port>/api/admin/dbms` lists registered servers
- [ ] `EnsureRegistrySchema` removed from all callers (bootstrap.go, console/migrate.go, businesscli/registry.go)
- [ ] No plaintext passwords anywhere in code/logs
- [ ] After running `./bin/console migrate down --plugin business_generator`, tables drop; re-running `up` recreates them (idempotent)
- [ ] Migration appears in `migration_targets` + `migrations` tables as `business_generator` target

## Dependencies
- Existing: `internal/db/secret.go` (SaveDBMSSecret/RotateDBMSSecret), `internal/db/registry.go` (LoadRegistry/ListDBMS/AddDBMS/RemoveDBMS), `internal/plugins/types.go` (ServiceDeps/ConnectionBinder), `internal/secrets` (AES-GCM)
- Deprecating: `bin/business` CLI (or keep as thin wrapper around console commands)
- Updating: SPEC v3 → v4 (replace EnsureRegistrySchema section with plugin migration approach)

## Notes
- After this WO, `bin/business` becomes unnecessary (all CLI moves to `console`). Optional separate task to delete it.
- Generator is phase-1 scaffolding only; real business entities come later.
- All existing data (`dbms`, `db_registry`) survives migration due to `IF NOT EXISTS` / `ON CONFLICT`.
