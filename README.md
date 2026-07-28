# MySQL Persistence Implementation Plan

## Overview
The project already supports MySQL persistence via the MariaDB JDBC driver (MySQL-compatible). The `SqlStorage` class handles MySQL/MariaDB/PostgreSQL/SQLite. This plan covers configuration, migration from JSON, and any code adjustments needed for production MySQL use.

---

## Current State Analysis

### Already Implemented
- `SqlStorage.java:44-46` - MySQL/MariaDB connection via `jdbc:mariadb://` URL
- `EconomyConfig.java:22-33` - Storage config with `type`, `host`, `port`, `database`, `user`, `password`, `tablePrefix`, `poolSize`
- `EconomyManager.java:52-59` - Storage initialization based on config type
- `build.gradle:38` - MariaDB JDBC driver dependency (`org.mariadb.jdbc:mariadb-java-client`)

### Config Defaults (generated on first run)
```json
{
  "storage": {
    "type": "JSON",
    "host": "localhost",
    "port": 3306,
    "database": "savs_economy",
    "user": "root",
    "password": "password",
    "tablePrefix": "savs_eco_",
    "poolSize": 10,
    "connectionTimeout": 30000,
    "idleTimeout": 600000
  }
}
```

---

## Implementation Tasks

### 1. Update Storage Type Enum & Config Validation
**File:** `src/main/java/savage/commoneconomy/config/EconomyConfig.java`
- Add `MARIADB` alias to `StorageType` enum for clarity
- Add config validation in `ConfigManager.loadMain()` to validate MySQL connection params when type is MYSQL/MARIADB

### 2. Enhance SqlStorage for Production MySQL
**File:** `src/main/java/savage/commoneconomy/storage/SqlStorage.java`

**Changes needed:**
- **Connection pool tuning**: Add `maximumPoolSize`, `minimumIdle`, `connectionTimeout`, `idleTimeout`, `maxLifetime` from config
- **MySQL-specific optimizations**: 
  - `cachePrepStmts=true`, `prepStmtCacheSize=250`, `prepStmtCacheSqlLimit=2048`
  - `useServerPrepStmts=true`, `rewriteBatchedStatements=true`
  - `useLocalSessionState=true`, `elideSetAutoCommits=true`
- **SSL support**: Add `useSSL`, `verifyServerCertificate`, `requireSSL` config options
- **Character set**: Ensure `characterEncoding=utf8mb4` and `connectionCollation=utf8mb4_unicode_ci`
- **Auto-reconnect**: Add `autoReconnect=true`, `maxReconnects=3`

### 3. Add MySQL-Specific Config Fields
**File:** `src/main/java/savage/commoneconomy/config/EconomyConfig.java` → `StorageConfig`
```java
public boolean useSSL = false;
public boolean verifyServerCertificate = false;
public boolean requireSSL = false;
public String characterEncoding = "utf8mb4";
public String connectionCollation = "utf8mb4_unicode_ci";
public int maxLifetime = 1800000; // 30 minutes
public int minimumIdle = 2;
```

### 4. Migration Tool: JSON → MySQL
**New File:** `src/main/java/savage/commoneconomy/storage/MigrationTool.java`
- CLI command or admin command to migrate existing `balances.json` to MySQL
- Steps:
  1. Load all accounts from JSON storage
  2. For each account, insert into MySQL with `ON DUPLICATE KEY UPDATE`
  3. Report success/failure counts
  4. Optionally backup JSON before migration

**Admin Command:** `/economy migrate json-to-mysql`

### 5. Health Check & Connection Validation
**File:** `src/main/java/savage/commoneconomy/storage/SqlStorage.java`
- Add `healthCheck()` method using `connection.isValid(5)`
- Add startup validation: test connection on init, log clear error if fails
- Add periodic validation in background (optional)

### 6. Configuration Documentation
**File:** `src/main/resources/config.json` (generated on first run) - ensure comments/docs for MySQL settings
**Or:** Update `README.md` with MySQL setup guide

---

## Configuration Example (config.json)

```json
{
  "storage": {
    "type": "MYSQL",
    "host": "localhost",
    "port": 3306,
    "database": "savs_economy",
    "user": "economy_user",
    "password": "secure_password",
    "tablePrefix": "savs_eco_",
    "poolSize": 20,
    "minimumIdle": 5,
    "connectionTimeout": 30000,
    "idleTimeout": 600000,
    "maxLifetime": 1800000,
    "useSSL": false,
    "verifyServerCertificate": false,
    "requireSSL": false,
    "characterEncoding": "utf8mb4",
    "connectionCollation": "utf8mb4_unicode_ci"
  }
}
```

---

## MySQL Database Setup

```sql
CREATE DATABASE savs_economy CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'economy_user'@'%' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON savs_economy.* TO 'economy_user'@'%';
FLUSH PRIVILEGES;
```

Table will be auto-created: `savs_eco_balances` with columns:
- `uuid` VARCHAR(36) PRIMARY KEY
- `name` VARCHAR(255)
- `balance` DECIMAL(30, 2)
- `version` BIGINT DEFAULT 0

---

## Migration Procedure

1. **Backup**: Copy `config/savs-common-economy/balances.json` to safe location
2. **Stop server**
3. **Edit config.json**: Change `"type": "MYSQL"` and fill in MySQL credentials
4. **Start server**: Tables auto-created
5. **Run migration**: Execute `/economy migrate json-to-mysql` (new command)
6. **Verify**: Check `/baltop` and player balances match
7. **Optional**: Remove JSON backup after confirmation

---

## Code Changes Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `EconomyConfig.java` | Modify | Add MySQL-specific config fields |
| `ConfigManager.java` | Modify | Validate MySQL config on load |
| `SqlStorage.java` | Modify | Enhanced connection pool, SSL, charset, health check |
| `MigrationTool.java` | New | JSON → MySQL migration utility |
| `AdminEconomyCommands.java` | Modify | Add `/economy migrate` command |

---

## Testing Checklist

- [ ] Fresh MySQL install: server starts, tables created, balances work
- [ ] Existing JSON data: migration command works, all balances preserved
- [ ] Connection loss: graceful error handling, reconnection works
- [ ] Pool exhaustion: max pool size respected, timeouts work
- [ ] SSL connections: works with `useSSL=true` and proper certs
- [ ] Large balances: DECIMAL(30,2) handles large numbers correctly
- [ ] Concurrent access: optimistic locking (version) prevents race conditions
- [ ] Cross-server sync: Redis pub/sub still works with MySQL storage

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| MariaDB driver incompatibility with MySQL 8.0+ | Test with target MySQL version; driver 3.5.7 supports MySQL 8.0 |
| Connection leaks | HikariCP leak detection; add `leakDetectionThreshold` |
| Charset issues (emojis in names) | Enforce `utf8mb4` in JDBC URL and table creation |
| Migration data loss | Dry-run mode; backup JSON first; transactional inserts |
| Pool exhaustion under load | Monitor `HikariPoolMXBean`; tune `poolSize`/`maxLifetime` |

---

## Out of Scope
- PostgreSQL-specific features (already supported)
- SQLite (already supported)
- Redis persistence (Redis is for pub/sub only, not primary storage)
- Backup/restore automation (external tooling recommended)
