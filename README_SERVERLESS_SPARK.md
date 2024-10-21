## Alibaba Cloud EMR Serverless Spark Adapter

### Installation

#### From Source Code

* code repo - https://github.com/EricGao888/dbt-spark.git
* code branch - adapter-support-ss-v1.8.0

```shell
python -m venv dbt-env-ss
source dbt-env-ss/bin/activate

pip install dbt-core

# cd to the root of dbt-spark source code root folder
pip install .
```

### Example Profile

```yaml
dbt_spark_test:
  outputs:
    local:
      connect_retries: 2
      connect_timeout: 60
      host: emr-spark-gateway-cn-hangzhou.data.aliyun.com # any non-empty string
      method: serverless_spark
      retry_all: False
      schema: order_dw
      type: spark
      ak: <your-access-key-id> # for security, you could use env var `export AK=<your-access-key-id>` instead of configuring this
      sk: <your-access-key-secret> # for security, you could use env var `export SK=<your-access-key-secret>` instead of configuring this
      region: <your-workspace-region-id> # e.g. cn-hangzhou
      workspace_id: <your-workspace-id> # e.g. w-72704d9fb078****
      server_side_parameters: # global spark configurations
        "spark.driver.memory": "4g"
        "spark.driver.cores": "1"
        "spark.executor.memory": "4g"
        "spark.executor.cores": "1"
        "spark.executor.instances": "2"
        "spark.dynamicAllocation.enabled": "true"

  target: local
```

### Supported Features
* Submitting jobs to **Alibaba Cloud EMR Serverless Spark**
* Logging `stderr` when job fails
* Concatenating `alter table column comments` statements in one serverless spark job
* Generating job names from sql comments automatically
* Configuring `spark-confs` in `profile.yml`
