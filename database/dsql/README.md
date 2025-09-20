# AWS DSQL

## Connection Formats

AWS DSQL driver supports two connection formats:

### DSQL-Specific Format (Recommended)
`dsql://cluster-endpoint/database?password=your-password&param1=value1`

This format is specifically designed for AWS DSQL and only requires:
- **Cluster endpoint**: Your DSQL cluster endpoint hostname
- **Password**: Your database password (passed as query parameter)
- **Database**: The database name (optional, defaults to 'postgres')

### Traditional URL Format
`dsql://user:password@cluster-endpoint:5432/database?param1=value1`

This format follows the standard database URL pattern for compatibility.

## Connection Parameters

| URL Query | WithInstance Config | Description |
|-----------|---------------------|-------------|
| `password` | - | Database password (DSQL-specific format only) |
| `x-migrations-table` | `MigrationsTable` | Name of the migrations table |
| `x-migrations-table-quoted` | `MigrationsTableQuoted` | Whether the migration table name is quoted |
| `x-statement-timeout` | `StatementTimeout` | Abort any statement that takes more than the specified number of milliseconds |
| `x-multi-statement` | `MultiStatementEnabled` | Enable multi-statement support |
| `x-multi-statement-max-size` | `MultiStatementMaxSize` | Maximum size of multi-statement blocks in bytes |
| `x-lock-table` | `LockTable` | Name of the table used for locking (DSQL only supports table-based locking) |
| `sslmode` | - | SSL mode (defaults to 'require' for DSQL-specific format) |

## AWS DSQL Limitations

AWS DSQL is PostgreSQL-compatible but has several limitations compared to full PostgreSQL:

1. **No Advisory Locks**: The DSQL driver only supports table-based locking. Advisory locks (`pg_advisory_lock`) are not supported.
2. **Limited SQL Feature Set**: DSQL supports a subset of PostgreSQL functionality. See [AWS DSQL documentation](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-unsupported-features.html) for details.

## Connection Notes

- **DSQL-specific format** (recommended): Uses cluster endpoint directly with password as query parameter
- **Traditional format**: Follows standard database URL pattern for compatibility  
- Uses PGX v4 driver for the underlying connection
- Automatically enforces SSL (sslmode=require) for security
- The driver automatically forces table-based locking regardless of any lock strategy parameter
- DSQL clusters use port 5432 by default and user 'root'

## Examples

### DSQL-Specific Format (Recommended)
```go
import (
    "database/sql"
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/dsql"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func main() {
    // Using the DSQL driver directly
    p := &dsql.DSQL{}
    d, err := p.Open("dsql://my-cluster.abc123.us-east-1.dsql.amazonaws.com/mydb?password=mypassword")
    if err != nil {
        // handle error
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file:///migrations",
        "dsql", 
        d)
    if err != nil {
        // handle error
    }

    m.Up()
}
```

### Traditional Format
```go
func main() {
    // Using sql.Open with postgres driver first, then WithInstance
    db, err := sql.Open("pgx/v4", "postgres://root:password@dsql-endpoint:5432/database?sslmode=require")
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

    m.Up()
}
```

## CLI Usage

### DSQL-Specific Format (Recommended)
```bash
# Using cluster endpoint and password
migrate -source file://path/to/migrations -database "dsql://my-cluster.abc123.us-east-1.dsql.amazonaws.com/mydb?password=mypassword" up

# With additional parameters
migrate -source file://path/to/migrations -database "dsql://my-cluster.abc123.us-east-1.dsql.amazonaws.com/mydb?password=mypassword&x-migrations-table=my_migrations" up
```

### Traditional Format
```bash
migrate -source file://path/to/migrations -database "dsql://root:password@dsql-cluster.us-east-1.dsql.amazonaws.com:5432/database?sslmode=require" up
```

## Testing

This driver was tested using:
1. **Unit tests**: Comprehensive test suite using PostgreSQL containers (since DSQL is PostgreSQL-compatible)
2. **Integration tests**: All database operations including locking, migrations, and error handling
3. **CLI integration**: Verified with migrate command-line tool

### End-to-End Testing with Real DSQL

To enable end-to-end testing with actual AWS DSQL clusters, you would need:

1. **AWS Credentials**: IAM role or user with DSQL permissions
2. **DSQL Cluster**: A test cluster that can be created/destroyed
3. **Environment Variables**: 
   ```bash
   export AWS_REGION=us-east-1
   export DSQL_CLUSTER_ENDPOINT=your-test-cluster.region.dsql.amazonaws.com
   export DSQL_PASSWORD=your-test-password
   ```

The current test suite uses PostgreSQL containers to simulate DSQL behavior, which provides comprehensive coverage of the driver functionality without requiring AWS resources.