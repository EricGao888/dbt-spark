# dbt-spark for Serverless Spark

This document covers the Serverless Spark (SS) specific features and usage of dbt-spark.

## Overview

This fork of dbt-spark adds support for connecting to Aliyun EMR Serverless Spark via Kyuubi Gateway, enabling dbt users to run data transformations on serverless Spark infrastructure.

## Prerequisites

### Install dbt-core and dbt-spark

```bash
# Install dbt-core
pip install dbt-core==1.5.9

# Install dbt-spark from source (this repo)
pip install -e .

# Install PyHive dependencies for thrift connection
pip install pyhive thrift thrift-sasl sasl pure-sasl
```

### Verify Installation

```bash
dbt --version
# Should show: dbt-core 1.5.9, spark 1.5.2
```

## Connection Methods

### Thrift Connection (Recommended for Kyuubi Gateway)

Create a `profiles.yml` with the thrift connection method:

```yaml
spark_thrift:
  target: dev
  outputs:
    dev:
      type: spark
      method: thrift
      host: localhost        # Spark Thrift Server or Kyuubi Gateway host
      port: 10000            # Thrift server port
      user: dbt              # Username
      schema: your_database  # Target schema/database
      connect_retries: 3
      connect_timeout: 10
      retry_all: true
```

### Connecting to Kyuubi Gateway

To connect to a Kyuubi Gateway endpoint, the key difference from a standard thrift connection is the `scheme: http` parameter. Kyuubi Gateway uses HTTP protocol over port 443, so you must specify `scheme: http` in your profile:

```yaml
spark_thrift:
  target: dev
  outputs:
    dev:
      type: spark
      method: thrift
      host: your-kyuubi-endpoint          # Kyuubi Gateway endpoint
      port: 443                            # 443 for public, 80 for internal
      user: your-username                  # Token name (non-DLF) or RAM user/role (DLF)
      password: your-token                 # Authentication token
      scheme: http                         # Required for Kyuubi Gateway
      schema: default                      # Target database/schema
      connect_retries: 3
      connect_timeout: 60                  # Serverless Spark may take time to start
      retry_all: true
```

> **Important**: The `scheme: http` field is a custom extension added in this fork. Without it, dbt-spark will attempt a raw TCP socket connection which will fail against Kyuubi Gateway's HTTP-based thrift endpoint.

## Example: Submit SQL to Serverless Spark via dbt

This section provides a complete step-by-step example of using dbt to submit SQL queries to Aliyun EMR Serverless Spark through Kyuubi Gateway.

### Step 1: Set Up Credentials

Create a config file (e.g., `ss-config`) in the project root with your Kyuubi Gateway credentials. This file should be listed in `.gitignore` to avoid committing secrets:

```bash
# ss-config (do NOT commit this file)
KYUUBI_ENDPOINT='your-gateway-endpoint'
KYUUBI_TOKEN='your-auth-token'
KYUUBI_USERNAME='your-username'
KYUUBI_PORT='443'
KYUUBI_USE_DLF='false'
```

### Step 2: Create a dbt Project

Create a minimal dbt project directory:

```
my_dbt_project/
├── dbt_project.yml
├── profiles.yml
└── models/
    ├── my_first_model.sql
    └── schema.yml
```

**dbt_project.yml**:
```yaml
name: 'my_dbt_project'
version: '1.0.0'
config-version: 2
profile: 'spark_thrift'
model-paths: ["models"]
```

**profiles.yml** (use your actual credentials from `ss-config`):
```yaml
spark_thrift:
  target: dev
  outputs:
    dev:
      type: spark
      method: thrift
      host: your-kyuubi-endpoint          # from KYUUBI_ENDPOINT
      port: 443                            # from KYUUBI_PORT
      user: your-username                  # from KYUUBI_USERNAME
      password: your-token                 # from KYUUBI_TOKEN
      scheme: http
      schema: default
      connect_retries: 3
      connect_timeout: 60
      retry_all: true
```

**models/my_first_model.sql**:
```sql
{{ config(materialized='view') }}

SELECT 1 as id, 'hello_dbt_spark' as message
```

**models/schema.yml**:
```yaml
version: 2
models:
  - name: my_first_model
    description: "A simple model to verify dbt-spark Kyuubi connectivity"
```

### Step 3: Verify Connection

```bash
cd my_dbt_project
dbt debug --profiles-dir .
```

Expected output:
```
Configuration:
  profiles.yml file [OK found and valid]
  dbt_project.yml file [OK found and valid]
Connection:
  host: your-kyuubi-endpoint
  port: 443
  schema: default
  Registered adapter: spark=1.5.2
  Connection test: [OK connection ok]

All checks passed!
```

### Step 4: Run Models

```bash
dbt run --profiles-dir .
```

Expected output:
```
Running with dbt=1.5.9
Found 1 model, 0 tests, ...

1 of 1 START sql view model default.my_first_model ............ [RUN]
1 of 1 OK created sql view model default.my_first_model ....... [OK in 1.74s]

Completed successfully
Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

> **Note**: The first run may take 2-3 minutes as Serverless Spark needs to start up. Subsequent runs will be faster if the Spark session is still active.

### Step 5: Verify Results (Optional)

You can verify the created view using PyHive directly:

```python
from pyhive import hive

conn = hive.connect(
    host='your-kyuubi-endpoint',
    port=443,
    scheme='http',
    username='your-username',
    password='your-token'
)
cursor = conn.cursor()
cursor.execute("SELECT * FROM default.my_first_model")
print(cursor.fetchall())  # [(1, 'hello_dbt_spark')]
cursor.close()
conn.close()
```

## Serverless Spark Kyuubi Gateway

### Environment Variables

For scripts that use environment variables (e.g., `examples/kyuubi_pyhive_example.py`):

```bash
# Set environment variables
export KYUUBI_ENDPOINT='your-gateway-endpoint'
export KYUUBI_TOKEN='your-auth-token'
export KYUUBI_USERNAME='your-username'

# Optional
export KYUUBI_PORT='443'        # Default: 443
export KYUUBI_USE_DLF='false'   # Default: false
```

### PyHive Direct Connection Example

```bash
# Run the PyHive example script
python examples/kyuubi_pyhive_example.py
```

See `examples/kyuubi_pyhive_example.py` for a complete working example of connecting to Kyuubi Gateway using PyHive directly.

### dbt Thrift Example

A ready-to-use dbt project example is available at `examples/dbt_thrift_example/`:

```bash
cd examples/dbt_thrift_example

# Update profiles.yml with your Kyuubi Gateway credentials

# Verify connection
dbt debug --profiles-dir .

# Run models
dbt run --profiles-dir .
```

See `examples/dbt_thrift_example/README.md` for detailed instructions.

## Reference

- [Aliyun EMR Serverless Spark Kyuubi Gateway Management](https://help.aliyun.com/zh/emr/emr-serverless-spark/user-guide/manage-kyuubi-gateways)
- [dbt-spark Documentation](https://docs.getdbt.com/docs/profile-spark)
- [dbt Installation Guide](https://docs.getdbt.com/docs/installation)
