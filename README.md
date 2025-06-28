# Apache Airflow – Comprehensive Guide

## What is Apache Airflow?

Apache Airflow is an open-source **workflow orchestration** platform used to author, schedule, and monitor complex workflows as code.  You define workflows as Directed Acyclic Graphs (DAGs) in Python, where each node is a task and edges define dependencies.  Airflow then manages the execution order and retries of tasks, providing a centralized UI to track progress and logs.  This makes it ideal for data pipelines (ETL/ELT), batch jobs, report generation, ML model training, and other automated processes.

* **Orchestration:** Airflow “orchestrates” data workflows by sequencing tasks with dependencies and triggers, ensuring they run in the correct order. In essence, it schedules and executes tasks (like Python scripts, queries, or command-line jobs) when their upstream tasks succeed, handling retries and alerts as needed. This lets you treat your entire data pipeline as code, version-controlled and repeatable.

## Key Features

Airflow’s design emphasizes **flexibility, visibility, organization,** and **scalability**:

* **Organization:** Workflows are defined in Python DAG files, which makes them modular and maintainable. You can group or tag DAGs for easy filtering, and Airflow automatically organizes tasks by their dependencies.  Using standard Python (and Jinja templating), you have full control over loops, parameters, and dynamic task generation.

* **Visibility:** Airflow’s rich web UI provides **full visibility** into pipelines. It shows the status of each DAG run and task instance, with built-in graphs and dashboards.  For example, the Home Page shows system health and recent run history, while the DAG graph and grid views let you inspect task states over time. Logs and task metadata are accessible through the UI, aiding debugging and audit.

* **Flexibility:** Airflow is highly extensible. Since workflows are code, you can write custom **operators**, **hooks**, and even plugins to integrate with virtually any system.  There are 1,500+ community providers (operator/hook packages) for AWS, Google Cloud, Azure, and many other tools. This lets you easily include tasks like running a Bash command, calling a Python function, or invoking an AWS SageMaker job in your DAG.

* **Scalability:** Airflow’s modular architecture uses a **scheduler**, **executor**, and **worker** model to scale.  It can orchestrate “an arbitrary number of workers,” so you can run many tasks in parallel. For example, the Celery or Kubernetes executors distribute tasks across multiple worker nodes for horizontal scaling.  Even for a small setup, you can use the LocalExecutor (parallel on one machine) and later upgrade to a distributed setup as demand grows.

## Limitations and When *Not* to Use Airflow

While powerful, Airflow is not a one-size-fits-all solution. Key limitations include:

* **Batch-only (not streaming):** Airflow is designed for batch workflows with clear start/end points, not continuous streaming. It uses time-based scheduling (or trigger-based via sensors) rather than processing endless event streams.  For real-time or micro-batch use cases, event-driven frameworks (Kafka streams, etc.) are more appropriate.

* **Requires coding skills:** Workflows are defined in code, so you need Python (and developer) expertise. Airflow is largely inaccessible to non-programmers or those expecting a low-code interface.  All logic and dependencies must be scripted, which gives flexibility but comes with a learning curve.

* **Limited built-in versioning:** Airflow does not natively version DAG files or automatically track changes across deployments. This makes auditing or rolling back pipeline changes harder. You can mitigate this via good source control practices, but out-of-the-box the system treats pipelines as code snapshots.

* **Complexity and maintenance:** A full Airflow setup has many components (scheduler, web server, metadata DB, executor, workers, etc.).  Running all these reliably requires ops work (database management, Python environments, networking). Smaller projects might find simpler tools (cron, scripted pipelines) easier.  Finally, Airflow DAGs can become brittle when very large or when many interdependent tasks change frequently.

In summary, use Airflow when you need **rich scheduling, dependency management, and monitoring of batch workflows**. Avoid it for simple one-off jobs, real-time event processing, or for users lacking developer support.

## Installing Apache Airflow (Hands-on)

To install Airflow locally or on a server, follow these general steps:

1. **Prepare Python environment:** Ensure you have Python 3.9–3.12 installed. Create a virtual environment (recommended) and activate it.
2. **Set `$AIRFLOW_HOME` (optional):** By default, Airflow uses `~/airflow`. You can change this by setting `export AIRFLOW_HOME=~/my_airflow` before installation.
3. **Install via pip with constraints:** Use the official constraint file for compatibility. For example:

   ```bash
   AIRFLOW_VERSION=3.0.2
   PYTHON_VERSION="$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"
   CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

   pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
   ```

   This ensures you install a tested set of Airflow dependencies.
4. **Initialize and run:** Once installed, initialize the metadata DB and start Airflow:

   ```bash
   airflow standalone
   ```

   The `airflow standalone` command will set up the database, create an admin user, and launch the webserver and scheduler.
5. **Access the UI:** Open `http://localhost:8080` in your browser and log in with the credentials shown in the terminal. You should see the default “example” DAGs, which you can enable and run to test your setup.

For production or Docker-based installs, you would set up a robust database (PostgreSQL/MySQL) instead of SQLite, and run the scheduler, workers, and webserver as separate services. But the above quick start is enough to get hands-on experience with Airflow.

## Apache Airflow Web UI Walkthrough

The Airflow Web UI provides a graphical way to monitor and manage workflows.  Important views include:

* **Home Page:** The default landing page shows system-wide status. It includes health indicators (e.g. “scheduler running,” database healthy) and summary charts of DAG/Task success rates over selectable time ranges. It also lists recently failing tasks or triggered DAGs, so you can spot issues quickly.
  &#x20;*Figure: Airflow Home Page (dark theme) showing system health and recent DAG/task statistics.* The Home Page provides an at-a-glance overview of the environment, with health indicators and links to recent activity.

* **DAG List View:** Clicking the **“DAGs”** tab lists all defined DAGs. Each row shows the DAG ID, schedule interval, most recent run status, and a mini-history chart of past runs. You can pause or unpause DAGs here, add tags for grouping, and use the search/filter bar to find specific workflows.
  &#x20;*Figure: Airflow DAGs List View (dark theme) with filters and actions.* The DAG List View displays every pipeline and its status, and lets you search, filter or sort them.

* **DAG Details (Grid/Graph Views):** Clicking a DAG opens its **Details page**. Here you’ll find multiple tabs:

  * *Grid View:* Shows a matrix of tasks (rows) by DAG runs (columns).  Each cell is colored by task state (success, failed, etc.), providing a quick overview of past runs. You can click a cell to view logs or even clear/mark tasks.
  * *Graph View:* Visualizes the DAG as a flowchart of task nodes and arrows for dependencies. This helps you understand the execution order and easily identify where failures occur.
  * *Code:* Lets you view the DAG’s Python source code (latest version).
  * *Tasks/Events/Run History:* Additional tabs list all tasks with metadata, events (version changes or triggers), and a table of all DAG runs with their status and duration.
    &#x20;*Figure: Airflow Graph View (dark theme) of a DAG’s structure.* The Graph View shows tasks and dependencies, making it easy to trace a workflow and investigate failures.

Overall, the UI provides complete visibility into workflows: you can trigger DAGs manually, rerun failed tasks, and drill into logs and metadata without leaving the browser.

## Components & Terminology

Airflow consists of several components, each with a specific role:

* **DAG (Directed Acyclic Graph):** Represents a workflow. It is defined in Python and lists all tasks and their dependencies. A DAG has no cycles, ensuring a clear execution order.

* **Task:** The basic unit of work in Airflow. Tasks are instances of operators (or sensors) and are arranged in the DAG with upstream/downstream links. For example, a task could be “run a SQL query” or “call a REST API.”

* **Task Instance:** One run of a Task for a specific DAG run (execution date). Each Task Instance has a state (queued, running, success, failed, etc.) that Airflow tracks. When you look at a past DAG run, you are seeing a collection of Task Instances.

* **Scheduler:** A daemon that parses DAG files, determines which tasks should run (based on schedule/tags/dependencies), and submits those tasks to an executor. The scheduler is responsible for populating Task Instances and sending them to queues when their dependencies are met.

* **Executor:** The component that **runs** tasks. It determines how tasks are actually executed.  Common executors include:

  * *Sequential Executor:* Runs one task at a time (mostly for testing).
  * *Local Executor:* Runs multiple tasks in parallel on the same machine.
  * *Celery Executor:* Dispatches tasks to a pool of external worker processes or machines via a message queue like RabbitMQ.
  * *Kubernetes Executor:* Launches each task as a pod in a Kubernetes cluster, automatically scaling to zero when idle.
    Whichever executor you choose, it pulls tasks from queues and runs them, then reports results back to the metadata DB.

* **Worker:** When using Celery or Kubernetes executors, workers are the processes or pods that pick up tasks from the queue and execute them. They perform the actual work (running code, copying files, etc.) and then update the task status in the database.

* **Web Server:** Hosts the Airflow UI. This component serves the dashboard and API calls to trigger DAGs or fetch logs. It reads from the metadata database to display DAGs, tasks, and run history.  Users interact with the system primarily through this web interface.

* **Metadata Database:** A SQL database (Postgres, MySQL, etc.) where Airflow stores all state: DAG definitions (serialized), task instance statuses, user accounts, variables, etc. The scheduler, webserver, and workers all read and write here. (By default, Airflow creates an `airflow.db` SQLite file for quick tests, but production should use a robust DB.)

* **Operator:** A template for a task. Operators abstract a unit of work (e.g. *BashOperator*, *PythonOperator*, *S3ToRedshiftOperator*, etc.). When you “instantiate” an operator in a DAG file, it creates a Task. Operators use Hooks under the hood to connect to external systems.

* **Hook:** A reusable interface to an external system or service. For example, a `PostgresHook` handles connecting to a Postgres database. Hooks fetch credentials from Airflow Connections and provide methods to run queries or upload files. Operators typically rely on hooks to perform their work.

* **Sensor:** A special type of operator that waits for a condition. For example, `FileSensor` waits for a file to appear, or `TimeDeltaSensor` waits for a time period. Sensors keep a task alive until the condition is true. In Airflow 2.x you can use **Deferrable Sensors** to free up worker slots while waiting (these rely on the **Triggerer** component).

* **Triggerer:** An optional, asyncio-based component that handles deferrable operators (sensors). It listens for events or conditions and only wakes a worker when a sensor’s condition is met. If you don’t use deferrable tasks, the triggerer isn’t needed.

* **Queue:** A named buffer in the executor setup. By default there is a `default` queue.  When the scheduler enqueues a Task Instance, it places it on a queue (e.g. “default”).  Executors/Workers watch these queues and pull tasks off them in order to run.

* **Plugins:** A way to extend Airflow with custom features. You can drop Python files in a plugins folder to add new operators, hooks, UI elements, or global functions.  The scheduler and webserver automatically load plugins on start.

* **Logs:** Every task run emits logs (stdout/stderr). Airflow stores logs (by default, on the local file system) and exposes them in the UI.  You can view a task’s logs via the UI or retrieve them from the logs directory on disk.  (Providers can add alternate log backends to send logs to S3, ElasticSearch, etc.)

* **airflow\.cfg:** The main configuration file for Airflow. On first run, Airflow creates an `airflow.cfg` in `$AIRFLOW_HOME` with default settings. You can edit this file or set the same options via environment variables (`AIRFLOW__SECTION__KEY`).

* **Trigger:** In Airflow parlance, a *trigger* can mean two things: (1) an external event that causes a DAG run (e.g. the “Trigger DAG” button or the REST API can kick off a DAG outside its schedule), or (2) the internal trigger mechanism like *Trigger Rules* that control task execution. Either way, triggers are how Airflow fires workflows.

* **Providers:** Modular packages that extend Airflow’s core. The “core” Airflow provides the scheduler and basic operators. **Provider packages** contain additional operators, hooks, sensors, and even UI pieces for specific services (e.g. an AWS provider, Google provider, Databricks provider). Installing a provider (e.g. `apache-airflow-providers-amazon`) adds support for that platform.

## Airflow on AWS (EC2 / Cloud Setup)

Airflow works well on cloud platforms like AWS.  You can run Airflow on an EC2 instance just as on-premise, or use managed services:

* **EC2 Installation:** Launch an EC2 instance (e.g. Amazon Linux or Ubuntu) with Python 3. Install Airflow via `pip` as above, configure a robust metadata database (Amazon RDS is common), and open port 8080 for the web UI.  You might also use AWS S3 or EFS for sharing DAG files between nodes. EC2 gives you full control (you can install Celery, use ElasticCache, etc.), but requires you to manage the servers.

* **Managed Workflows (MWAA):** AWS offers **Managed Workflows for Apache Airflow** which is a fully managed service. It takes care of servers, scaling, upgrades, and integrates with IAM and S3.  MWAA supports most Airflow features and is simply an Airflow environment you configure in the AWS console (you still write DAG code in Python).

* **Cloud Integrations:** Airflow has built-in AWS operators and hooks. For example, there are operators for **S3, Athena, EMR, Glue, ECS, SageMaker, Redshift** and more.  These come from the `apache-airflow-providers-amazon` package.  In fact, Airflow advertises “plug-and-play operators” for AWS and other clouds, making it easy to orchestrate tasks across AWS services.  For instance, you might have a DAG that copies files in S3, runs a Redshift query, triggers an EMR Spark job, and then notifies via SNS, all with pre-built Airflow operators.

## Advanced Use Cases

Airflow’s flexibility enables many complex scenarios:

* **Data Pipelines (ETL/ELT):** Orchestrate multi-stage data processing. Airflow can extract data from databases/APIs, transform it (e.g. via Spark or Python code), and load it into data warehouses. It handles tasks like waiting for upstream loads, running parallel jobs, and merging results.

* **Machine Learning Pipelines:** Schedule the entire ML workflow: data preprocessing, model training, evaluation, and deployment. You can trigger AWS SageMaker training jobs or GCP AI Platform jobs from Airflow, then wait for completion before moving to the next step.

* **Dynamic Task Mapping:** Airflow 2.x supports dynamic task generation (Task Mapping), allowing you to create tasks at runtime based on input data. For example, you could read a list of files or table partitions and spawn tasks to process each in parallel.

* **Event-driven (deferrable) tasks:** Using the Triggerer and Sensors, Airflow can effectively “pause” workflows until certain conditions (file arrival, external job completion) are met, then continue execution without holding up a worker.

* **Kubernetes / Cloud Native Workflows:** Run Airflow in Kubernetes (with the KubernetesExecutor or Helm Charts). Tasks themselves can launch Kubernetes Jobs or pods, enabling scaling across a cluster. This is useful for large-scale data processing where each task may need its own container environment.

* **DevOps / Infrastructure:** Some use Airflow to automate infrastructure tasks (backups, log rotations, or even CI/CD pipelines). For example, you could have a nightly DAG that takes EBS snapshots, transfers configs, or deploys applications.

* **Data Warehousing (ELT):** Many teams use Airflow to schedule data warehouse loads (e.g., moving raw data into a staging area, then executing SQL to populate fact tables on a schedule).

Airflow is also commonly integrated with **other tools**. For example, you can call DBT (data build tool) commands from Airflow tasks for transformation workflows, or trigger Apache Spark jobs. The key is that Airflow’s extensible operators let you “orchestrate anything you can script or API-call”.

## Summary

Apache Airflow is a powerful tool for **workflow orchestration**. It excels at organizing complex, scheduled pipelines with clear visibility and extensibility. However, it shines primarily in batch/data workflows and requires developer effort to maintain. By understanding its components (DAGs, Scheduler, Executor, etc.) and UI, you can use Airflow to automate a wide variety of data engineering and ML tasks reliably.
