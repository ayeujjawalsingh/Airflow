# Apache Airflow: S3 File Processing – Two Approaches

This guide demonstrates two ways to build an Apache Airflow DAG that performs a multi-step file processing workflow using AWS S3:

- ✅ Approach 1: Pure Python-based DAG
- ✅ Approach 2: YAML-config-driven DAG

---

## ✅ Use Case Summary

This DAG downloads a file from S3, processes it using a Python function, moves the output, and logs every step using Airflow operators.

---

## 🚀 Approach 1: Pure Python-based DAG

### 📂 File: `s3_pipeline_dag.py`

```python
from airflow import DAG
from airflow.operators.dummy import DummyOperator
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from datetime import datetime

def download_from_s3(bucket_name, key, local_path, **kwargs):
    hook = S3Hook(aws_conn_id='aws_default')
    hook.get_key(key, bucket_name).download_file(local_path)

def process_file(local_path, **kwargs):
    with open(local_path, 'r') as f:
        lines = f.readlines()
    with open(local_path.replace(".txt", "_processed.txt"), 'w') as f:
        f.write(f"Processed {len(lines)} lines")

default_args = {'owner': 'airflow', 'start_date': datetime(2024, 1, 1)}
with DAG(dag_id='s3_file_processing_pipeline', default_args=default_args, schedule_interval=None, catchup=False) as dag:
    start = DummyOperator(task_id='start')
    download = PythonOperator(task_id='download_file', python_callable=download_from_s3,
        op_kwargs={'bucket_name': 'your-bucket', 'key': 'input.txt', 'local_path': '/tmp/sample.txt'})
    process = PythonOperator(task_id='process_file', python_callable=process_file,
        op_kwargs={'local_path': '/tmp/sample.txt'})
    move = BashOperator(task_id='move_file', bash_command='mv /tmp/sample_processed.txt /tmp/archive.txt')
    end = DummyOperator(task_id='end')

    start >> download >> process >> move >> end
```

---

## 🔧 Approach 2: YAML-driven DAG (Config + Loader Script)

### 📄 File: `configs/s3_dag_config.yaml`

```yaml
dag:
  dag_id: "s3_yaml_pipeline"
  schedule_interval: "@daily"
  start_date: "2024-01-01"
  catchup: false
  tags: ["yaml", "aws"]

tasks:
  - task_id: "start"
    operator: "DummyOperator"
  - task_id: "download_file"
    operator: "PythonOperator"
    callable: "download_from_s3"
    params:
      bucket_name: "your-bucket"
      key: "input.txt"
      local_path: "/tmp/sample.txt"
  - task_id: "process_file"
    operator: "PythonOperator"
    callable: "process_file"
    params:
      local_path: "/tmp/sample.txt"
  - task_id: "move_file"
    operator: "BashOperator"
    bash_command: "mv /tmp/sample_processed.txt /tmp/archive.txt"
  - task_id: "end"
    operator: "DummyOperator"

dependencies:
  - ["start", "download_file"]
  - ["download_file", "process_file"]
  - ["process_file", "move_file"]
  - ["move_file", "end"]
```

### 🐍 File: `s3_pipeline_from_yaml.py`

```python
import yaml
from pathlib import Path
from airflow import DAG
from airflow.operators.dummy import DummyOperator
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.utils.dates import datetime
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

def download_from_s3(bucket_name, key, local_path, **kwargs):
    hook = S3Hook(aws_conn_id='aws_default')
    hook.get_key(key, bucket_name).download_file(local_path)

def process_file(local_path, **kwargs):
    with open(local_path, 'r') as f:
        lines = f.readlines()
    with open(local_path.replace('.txt', '_processed.txt'), 'w') as f:
        f.write(f"Processed {len(lines)} lines")

CONFIG_PATH = Path(__file__).parent / "configs" / "s3_dag_config.yaml"
with open(CONFIG_PATH, 'r') as file:
    config = yaml.safe_load(file)

dag_conf = config['dag']
dag = DAG(dag_id=dag_conf['dag_id'], start_date=datetime.fromisoformat(dag_conf['start_date']),
          schedule_interval=dag_conf['schedule_interval'], catchup=dag_conf.get('catchup', False),
          tags=dag_conf.get('tags', []))

task_map = {}
for task_def in config['tasks']:
    task_id = task_def['task_id']
    if task_def['operator'] == 'DummyOperator':
        task = DummyOperator(task_id=task_id, dag=dag)
    elif task_def['operator'] == 'PythonOperator':
        task = PythonOperator(task_id=task_id, python_callable=globals()[task_def['callable']],
                              op_kwargs=task_def.get('params', {}), dag=dag)
    elif task_def['operator'] == 'BashOperator':
        task = BashOperator(task_id=task_id, bash_command=task_def['bash_command'], dag=dag)
    task_map[task_id] = task

for dep in config['dependencies']:
    task_map[dep[0]] >> task_map[dep[1]]
```

---

## ✅ Comparison

| Feature                  | Python-Based         | YAML-Driven         |
|--------------------------|----------------------|---------------------|
| Code in DAG              | Fully in Python      | Defined in YAML     |
| Separation of config     | ❌                   | ✅ Clean separation |
| Reusable for teams       | ❌ Less reusable     | ✅ Highly reusable  |
| Easier to version config | ❌                   | ✅ Yes              |
| Flexibility (custom logic)| ✅ Full Python power | ✅ with wrappers    |

---

## ✅ Requirements

Install necessary packages:

```bash
pip install apache-airflow apache-airflow-providers-amazon pyyaml
```

Make sure AWS credentials (`aws_default`) are configured in Airflow.

---

## ✅ Conclusion

- Use **Python DAGs** when workflows are logic-heavy and custom.
- Use **YAML-based DAGs** when workflow structure is repeatable and config-driven.
