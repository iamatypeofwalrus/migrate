# AWS DSQL

`dsql://user:password@host:port/dbname?param1=value1&param2=value2`

| URL Query    | WithInstance Config | Description |
|-------------|-------|-------------|
| `x-migrations-table` | `MigrationsTable` | Name of the migrations table |
| `x-migrations-table-quoted` | `MigrationsTableQuoted` | Whether the migration table name is quoted |
| `x-statement-timeout` | `StatementTimeout` | Abort any statement that takes more than the specified number of milliseconds |
| `x-multi-statement` | `MultiStatementEnabled` | Enable multi-statement support |
| `x-multi-statement-max-size` | `MultiStatementMaxSize` | Maximum size of multi-statement blocks in bytes |
| `x-lock-table` | `LockTable` | Name of the table used for locking (DSQL only supports table-based locking) |

## AWS DSQL Limitations

AWS DSQL is PostgreSQL-compatible but has several limitations compared to full PostgreSQL:

1. **No Advisory Locks**: The DSQL driver only supports table-based locking. Advisory locks (`pg_advisory_lock`) are not supported.
2. **Limited SQL Feature Set**: DSQL supports a subset of PostgreSQL functionality. See [AWS DSQL documentation](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-unsupported-features.html) for details.

## Connection Notes

- The driver accepts `dsql://` URLs but internally converts them to `postgres://` for connection since DSQL is PostgreSQL-compatible
- Uses PGX v4 driver for the underlying connection
- All PostgreSQL connection parameters are supported
- The driver automatically forces table-based locking regardless of the `x-lock-strategy` parameter

## Example

```go
import (
    "database/sql"
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/dsql"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func main() {
    db, err := sql.Open("pgx/v4", "postgres://user:password@dsql-endpoint:5432/database?sslmode=require")
    if err != nil {
        // handle error
    }

    driver, err := dsql.WithInstance(db, &dsql.Config{})
    if err != nil {
        // handle error
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file:///migrations",
        "dsql", 
        driver)
    if err != nil {
        // handle error
    }

    m.Up() // or m.Step(2) if you want to explicitly set the number of migrations to run
}
```

## CLI Usage

```bash
migrate -source file://path/to/migrations -database dsql://user:password@dsql-endpoint:5432/database?sslmode=require up
```