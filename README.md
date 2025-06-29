# Learn Airflow: An Overview

Apache Airflow is an open-source platform designed to help data professionals efficiently create, schedule, and monitor data tasks and workflows. It's built entirely in Python, offering robust integration capabilities, high scalability, and a rich user interface. Being open-source, it's a cost-effective solution for orchestrating data pipelines.

---

## 🔧 1. Setup & Installation

### 1.1 Local Installation

```bash
python3 -m venv .venv
source .venv/bin/activate

# Install Airflow with common providers
export AIRFLOW_VERSION=2.7.3
pip install "apache-airflow[amazon,postgres]==${AIRFLOW_VERSION}" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-3.8.txt"

# Initialize database and start services
airflow db init
airflow scheduler &
airflow webserver --port 8080
```

- Access the UI at: `http://localhost:8080`

### 1.2 Docker Installation (Recommended)

**Directory Structure:**

```
Airflow_Docker/
├── dags/
├── logs/
├── plugins/
├── requirements.txt
└── docker-compose.yml
```

**Sample **``**:**

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - pgdata:/var/lib/postgresql/data

  airflow:
    image: apache/airflow:2.7.3
    depends_on:
      - postgres
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    volumes:
      - ./dags:/opt/airflow/dags
      - ./logs:/opt/airflow/logs
      - ./plugins:/opt/airflow/plugins
      - ./requirements.txt:/requirements.txt:ro
    ports:
      - "8080:8080"
    command: webserver

volumes:
  pgdata:
```

**Launch Commands:**

```bash
docker compose up -d  # Start webserver

docker compose exec airflow airflow scheduler  # Run scheduler
```

**requirements.txt Example:**

```text
apache-airflow-providers-amazon
boto3
awscli
```

---

## 🧱 2. Key Airflow Concepts

### 2.1 DAG (Directed Acyclic Graph)

- **Directed:** Tasks have a specified, one-way dependency.
- **Acyclic:** No loops allowed, ensuring logical flow.
- **Graph:** Nodes are tasks, edges are dependencies.

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

dag = DAG('sample_dag', start_date=datetime(2025,6,29), schedule_interval='@daily')

step1 = BashOperator(task_id='step1', bash_command='echo step1', dag=dag)
step2 = BashOperator(task_id='step2', bash_command='echo step2', dag=dag)

step1 >> step2  # Sets dependency
```

### 2.2 Tasks & Operators

- **Task:** An instance of an operator.
- **Operator:** Blueprint of work (Python, Bash, SQL, etc.).
- **Types:**
  - `PythonOperator`, `BashOperator`, `S3ToRedshiftOperator`, `DummyOperator`, `BranchPythonOperator`, `SubDagOperator`

### 2.3 Parameters, Dependencies, and Schedules

- Parameters: Configure behavior via kwargs.
- Dependencies: Set with `>>`, `<<`, `.set_upstream()`, `.set_downstream()`, or `chain()`
- Schedules: Define execution frequency, e.g., `@hourly`, `@daily`

---

## 🏗️ 3. Architecture Overview

- **Webserver:** User interface
- **Scheduler:** Triggers task execution
- **Executor:** Chooses where/how to run tasks
- **Database:** Stores Airflow metadata
- **Workers:** Perform task execution

`airflow.cfg` controls configuration (executors, DB URI, logging).

---

## 🧠 4. Deeper Dive into Components

### 4.1 Sensors

Wait for external condition before executing downstream task.

```python
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor

wait_file = S3KeySensor(
  task_id='wait_for_file',
  bucket_name='my_bucket',
  bucket_key='incoming/data.csv',
  aws_conn_id='aws_default',
  poke_interval=30,
  timeout=300,
  mode='reschedule'
)
```

- **poke\_interval**: Time between condition checks
- **timeout**: Max wait time
- **mode**: `poke` (blocks) vs `reschedule` (defers)

### 4.2 Hooks

Connect to external systems (S3, Postgres, Redshift, etc.)

```python
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

hook = S3Hook(aws_conn_id='aws_default')
keys = hook.list_keys(bucket_name='my_bucket')
```

### 4.3 XCom (Cross Communication)

Pass data between tasks:

```python
# Push:
ti.xcom_push(key='file_name', value='data.csv')

# Pull:
filename = ti.xcom_pull(task_ids='task_id', key='file_name')
```

### 4.4 Variables & Connections

- **Variables:** Key-value store
- **Connections:** Centralized config to external services (DBs, APIs)

### 4.5 Datasets (Cross-DAG Triggers)

Trigger DAGs based on data readiness, not time.

```python
# Producer
PythonOperator(task_id='update_data', outlets=[Dataset('s3://my-data/file.csv')])

# Consumer
@dag(schedule=[Dataset('s3://my-data/file.csv')])
def downstream_dag(): ...
```

---

## 🚀 5. Example: Full Workflow (Sensor + Hook)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from datetime import datetime

def process_file(bucket, key):
    hook = S3Hook()
    path = hook.download_file(key, bucket_name=bucket, local_path='/tmp/data.csv')
    print("Downloaded to", path)

with DAG('s3_pipeline', start_date=datetime(2025,6,29), schedule_interval='@daily') as dag:
    wait = S3KeySensor(
        task_id='wait_for_s3',
        bucket_name='my_bucket',
        bucket_key='data.csv',
        aws_conn_id='aws_default',
        mode='reschedule'
    )
    process = PythonOperator(
        task_id='process_file',
        python_callable=process_file,
        op_kwargs={'bucket': 'my_bucket', 'key': 'data.csv'}
    )
    wait >> process
```

---

## 🧼 6. Best Practices

- Use `mode='reschedule'` for sensors
- Keep business logic outside DAG definitions
- Use `@task` and `@dag` decorators (TaskFlow API)
- Avoid large XCom payloads (use S3, GCS, etc. for large data)
- Use `.env` and secrets for environment-specific configs

---

## 📚 7. Resources

- [Airflow Docs](https://airflow.apache.org/docs/)
- [Awesome Airflow GitHub](https://github.com/apache/airflow)
- [Provider Packages](https://github.com/apache/airflow/tree/main/airflow/providers)

---

💡 *Use Airflow to build scalable, reliable, and elegant data pipelines.*

