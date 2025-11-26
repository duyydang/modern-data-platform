# Enterprise On-Premise Data Lakehouse Architecture

**Version:** 2.0  
**Target Environment:** Kubernetes (On-Premise / Private Cloud)  
**Architecture Pattern:** Lakehouse (Compute/Storage Separation)  
**Deployment Strategy:** Phased Implementation

---

<img width="2335" height="1915" alt="Data Architect Onprem1" src="https://github.com/user-attachments/assets/5ecdb73b-8660-41b5-a101-c60a93878136" />

## Infrastructure Requirements

### Minimum Production Cluster

**Phase 1 (MVP - 6 months):**
- **Control Plane:** 3 nodes (Master nodes)
  - 4 vCPU, 16GB RAM each
  - For K8s control plane, etcd
  
- **Data Plane:** 5 nodes (Worker nodes)
  - **Node Type A (Storage):** 2 nodes
    - 8 vCPU, 32GB RAM, 4TB HDD (MinIO)
  - **Node Type B (Compute):** 3 nodes
    - 16 vCPU, 64GB RAM, 500GB SSD (Trino workers)

**Phase 2 (Scale - 12 months):**
- Add 3 more compute nodes (Type B)
- Add 2 more storage nodes (Type A)

**Phase 3 (Advanced - 18 months):**
- Add 2 dedicated Spark nodes: 32 vCPU, 128GB RAM each
- Add 1 Kafka node: 8 vCPU, 32GB RAM, 1TB SSD

### Network Requirements
- **Internal:** 10Gbps between nodes (minimum)
- **Storage Network:** Dedicated VLAN for MinIO traffic (recommended)
- **Load Balancer:** MetalLB or HAProxy for service exposure

---

## PHASE 1: Foundation (Months 1-6) - MVP

**Goal:** Establish secure, queryable data lake with basic ETL

### 1.1 Core Infrastructure Layer

#### Kubernetes Management
**Component:** Rancher or Lens (UI) + kubectl

- **What it does (Simple explanation):** Think of this as the "control panel" for your entire data platform. Just like you use a file explorer to see files on your computer, Rancher/Lens lets you see all your applications running in the cluster.

- **Function:** Cluster management, monitoring, troubleshooting

- **Real-world analogy:** Imagine you're managing an apartment building. Rancher is like the building manager's office where you can:
  - See which apartments (pods) are occupied
  - Check if the heating (services) is working
  - View maintenance logs
  - Restart broken appliances

- **Deployment:** Single instance, access via browser

- **Why we need it:** Without this, you'd need to type complex commands in a terminal to check if things are running. Rancher gives you buttons, graphs, and dashboards instead.

- **Alternatives:** OpenLens (free), K9s (terminal-based for power users)

---

#### Ingress Controller
**Component:** Nginx Ingress Controller or Traefik

- **What it does (Simple explanation):** This is the "front door" to your data platform. When you type `https://trino.yourcompany.com` in your browser, the Ingress Controller decides "Oh, they want Trino" and routes you there.

- **Function:** Routes external traffic to services (Trino UI, Airflow UI, etc.)

- **Real-world analogy:** Think of a hotel receptionist:
  - Guest asks "Where's the restaurant?" → Receptionist directs them to Floor 3
  - Guest asks "Where's the gym?" → Receptionist directs them to Floor 5
  - Ingress asks "Who wants trino.company.com?" → Routes to Trino service
  - Ingress asks "Who wants airflow.company.com?" → Routes to Airflow service

- **Deployment:** 
  ```yaml
  # 2 replicas for HA (if one breaks, the other handles traffic)
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
  ```

- **Why we need it:** Without this, you'd need to remember port numbers like `http://10.20.30.40:8080` for every service. With Ingress, you get clean URLs like `https://trino.yourcompany.com`.

- **Alternatives:** HAProxy, Istio Gateway (more complex)

---

#### Certificate Management
**Component:** cert-manager

- **What it does (Simple explanation):** Automatically creates and renews the "padlock" certificates that make your websites show `https://` instead of `http://`. 

- **Function:** Automatic SSL/TLS certificate provisioning

- **Real-world analogy:** Like renewing your car registration:
  - **Without cert-manager:** You have to manually renew certificates every 90 days (forget once = website shows "Not Secure" warning)
  - **With cert-manager:** It's like automatic renewal—you set it up once, and it handles everything forever

- **Deployment:** Helm install with Let's Encrypt (or internal CA)

- **Why we need it:** 
  - Security: Encrypts data between user's browser and your platform
  - Trust: Modern browsers show warnings if sites don't use HTTPS
  - Compliance: Many regulations require encrypted connections

- **How it works:**
  1. You deploy Trino → cert-manager notices
  2. cert-manager asks Let's Encrypt "Can I get a certificate for trino.company.com?"
  3. Let's Encrypt verifies you own the domain
  4. Certificate is issued and installed automatically
  5. Before expiry (90 days), cert-manager renews it automatically

- **Alternatives:** Manual certificate management (not recommended), Commercial CAs like DigiCert

---

### 1.2 Security & Identity Layer

#### Component: Keycloak
- **Deployment:**
  ```yaml
  Replicas: 2 (HA mode)
  CPU: 2 cores per pod
  Memory: 4GB per pod
  Database: PostgreSQL 14+ (3-node cluster via Bitnami)
  ```
- **PostgreSQL for Keycloak:**
  - Primary: 1 node (4 vCPU, 8GB RAM)
  - Standby: 1 node (same specs)
  - Backup: pg_basebackup daily to MinIO

---

### 1.3 Storage Layer

#### Component: MinIO
- **Deployment via MinIO Operator:**
  ```yaml
  Tenant: "datalake"
  Servers: 4 (minimum for erasure coding)
  Volumes per Server: 4 (16 total volumes)
  Storage Class: local-path (or Longhorn)
  Capacity: 1TB per volume = 16TB raw (8TB usable with EC:4)
  ```
- **Bucket Strategy:**
  - `bronze/` - Raw ingested data
  - `silver/` - Cleaned data
  - `gold/` - Business-ready tables
  - `backups/` - Database dumps, configs

---

### 1.4 Metadata & Catalog Layer

#### Component: Project Nessie
- **Deployment:**
  ```yaml
  Replicas: 2
  CPU: 1 core per pod
  Memory: 2GB per pod
  Database: PostgreSQL (can share with Keycloak or separate)
  ```
- **Configuration:**
  - Create default branch: `main`
  - Enable authentication via Keycloak

---

### 1.5 Query Engine (Serving Layer)

#### Component: Trino
- **Deployment:**
  ```yaml
  Coordinator:
    replicas: 1
    cpu: 4 cores
    memory: 16GB
  
  Workers:
    replicas: 3 (start), scale to 6 later
    cpu: 8 cores each
    memory: 32GB each
  ```
- **Catalogs to Configure:**
  - `iceberg` → points to Nessie
  - `postgresql` → for testing/migration (optional)
- **Access Control:**
  - Enable file-based access control initially
  - Example rule: `SELECT on iceberg.gold.* for role:analyst`

---

### 1.6 Orchestration Layer

#### Component: Apache Airflow
- **Deployment (Official Helm Chart):**
  ```yaml
  Executor: KubernetesExecutor
  Webserver: 2 replicas (2 vCPU, 4GB each)
  Scheduler: 2 replicas (2 vCPU, 4GB each)
  Database: PostgreSQL (3rd instance or shared)
  Redis: For CeleryExecutor (if using) - 1 replica
  ```
- **Initial DAGs:**
  - Health check DAG (test all connections)
  - Daily table maintenance DAG (placeholder)

---

### 1.7 Observability Layer (NEW - Critical!)

#### Component: Prometheus + Grafana Stack (kube-prometheus-stack)
- **Function:** Metrics collection, alerting, dashboards
- **Deployment:**
  ```yaml
  Prometheus:
    replicas: 2
    retention: 15 days
    storage: 50GB per replica
  
  Grafana:
    replicas: 2
    dashboards: Pre-installed for K8s, MinIO, Trino
  
  Alertmanager:
    replicas: 2
    routes: Email/Slack for critical alerts
  ```
- **Key Metrics to Track:**
  - MinIO: Disk usage, request latency
  - Trino: Query queue length, memory pressure
  - Kubernetes: Node CPU/Memory, pod restarts
- **Why:** You cannot run production without knowing when systems are failing

#### Component: Loki (Log Aggregation)
- **Function:** Centralized logging (like ELK but lighter)
- **Deployment:**
  ```yaml
  Loki: 1 replica (2 vCPU, 4GB RAM)
  Promtail: DaemonSet (runs on every node)
  Storage: MinIO (write logs to S3 bucket)
  ```
- **Why:** Instead of SSH-ing into pods, query logs via Grafana

---

### 1.8 Backup & Disaster Recovery (NEW)

#### Component: Velero

- **What it does (Simple explanation):** Velero is like a "Save Game" button for your entire data platform. If something breaks catastrophically, you can reload a previous save point.

- **Function:** Backup entire K8s namespaces (configs + data)

- **Real-world analogy:** Think of time travel for your system:
  - **Monday 9 AM:** Everything works perfectly, Velero takes a "snapshot"
  - **Tuesday 10 AM:** Someone accidentally deletes the production database
  - **Tuesday 10:05 AM:** You press "Restore from Monday" → System goes back to Monday's state

- **Deployment:**
  ```yaml
  Velero Server: 1 replica
  Backup Location: MinIO S3 bucket (separate from data)
  Schedule: Daily full backup at 2 AM
  Retention: 30 days
  ```

- **What to Backup:**
  - All PostgreSQL databases (via pg_dump in CronJob)
  - Keycloak realm configs (all user accounts and permissions)
  - Airflow DAGs (your workflow definitions)
  - Trino catalog configs (connection settings)

- **Why we need it:** 
  - Hardware failure: Server catches fire → Restore to new hardware
  - Human error: Junior engineer runs `DELETE FROM users` → Restore last backup
  - Ransomware: Hackers encrypt your data → Restore clean backup
  - Testing: Want to test a risky upgrade? Take backup first!

- **Recovery Time Objective (RTO):** How long to restore
  - With Velero: ~30 minutes for full platform
  - Without backup: Days or weeks to rebuild from scratch

- **Important:** A backup you never test is just wishful thinking. Run restore drills quarterly!

- **Alternatives:** Kasten K10 (commercial), Stash

---

#### PostgreSQL Backup Strategy

- **What it does (Simple explanation):** Three-layer protection for your databases (the "crown jewels" of your platform)

- **Tool:** pgBackRest or Barman

- **Real-world analogy:** Think of protecting your phone:
  - **Full backup** = Complete phone backup to computer (weekly)
  - **Incremental backup** = Only backup new photos since last backup (daily)
  - **WAL archiving** = Real-time sync to cloud (continuous)

- **Schedule:**
  - **Full backup:** Weekly (Sunday 2 AM)
    - Takes a complete snapshot of the database
    - Size: Could be 50GB-500GB depending on data
    - Takes 1-2 hours
  
  - **Incremental backup:** Daily (Every day 2 AM)
    - Only backs up what changed since last full backup
    - Size: Usually 1-10GB
    - Takes 5-15 minutes
  
  - **WAL archiving:** Continuous (every few seconds)
    - Write-Ahead Logs = "transaction diary"
    - Captures every INSERT/UPDATE/DELETE in real-time
    - Size: Small files, 16MB each

- **RPO Target:** 15 minutes (via WAL)
  - **RPO** = Recovery Point Objective = "How much data can we afford to lose?"
  - With 15-minute RPO: If disaster strikes at 3:00 PM, you can recover to 2:45 PM

- **Recovery scenarios:**
  - **Scenario 1: Accidental table drop at 3:15 PM**
    - Restore from morning backup (8 AM)
    - Replay WAL files from 8 AM → 3:14 PM
    - Result: Only lose 1 minute of data
  
  - **Scenario 2: Complete server failure**
    - Provision new server
    - Restore last full backup
    - Replay incremental + WAL
    - Time: 1-2 hours

- **Storage location:** Write backups to MinIO (different bucket from production data)

- **Why we need THREE layers:**
  - Full backup alone: Expensive, slow
  - WAL alone: Can't restore if base is corrupted
  - Together: Fast, cheap, reliable

- **Testing backups:**
  ```bash
  # Monthly drill: Restore to a test environment
  1. Create new Postgres instance "test-restore"
  2. Restore last night's backup
  3. Verify data looks correct
  4. Document time taken
  5. Delete test instance
  ```

- **Alternatives:** pg_dump (simple but locks database), Patroni (built-in backup), Cloud provider snapshots

---

### 1.9 Secrets Management (NEW)

#### Component: Sealed Secrets or External Secrets Operator

- **What it does (Simple explanation):** Keeps passwords and API keys safe, even when stored in Git (version control).

- **Function:** Encrypt secrets in Git (GitOps safe)

- **Real-world analogy:** Think of a secret message system:
  - **Bad way:** Write password on sticky note, stick to monitor (everyone can see it)
  - **Kubernetes Secret (better):** Lock password in safe, but the safe key is under the doormat
  - **Sealed Secret (best):** Encrypt password with a code only the safe can decrypt. Even if someone steals the encrypted text, they can't read it.

- **The Problem Without This:**
  ```yaml
  # database-secret.yaml (stored in Git)
  apiVersion: v1
  kind: Secret
  data:
    password: cGFzc3dvcmQxMjM=  # This is just base64, NOT encrypted!
  ```
  - Anyone with Git access can decode this: `echo "cGFzc3dvcmQxMjM=" | base64 -d` → "password123"
  - Former employees still have the Git history with passwords

- **The Solution With Sealed Secrets:**
  ```yaml
  # database-sealed-secret.yaml (stored in Git - SAFE!)
  apiVersion: bitnami.com/v1alpha1
  kind: SealedSecret
  spec:
    encryptedData:
      password: AgBqk2x... (1000 characters of gibberish)
  ```
  - This gibberish can ONLY be decrypted by your Kubernetes cluster
  - Safe to commit to public GitHub!

- **How it works:**
  1. **Setup:** Install Sealed Secrets Controller in cluster
     - It generates a **private key** (stays in cluster forever, never leaves)
     - It generates a **public key** (you download this)
  
  2. **Creating a secret:**
     ```bash
     # Encrypt password using public key
     echo -n "password123" | kubeseal --raw \
       --from-file=/dev/stdin \
       --scope=cluster-wide > encrypted-password.txt
     ```
  
  3. **Deployment:**
     - You commit `encrypted-password.txt` to Git
     - Deploy to Kubernetes
     - Sealed Secrets Controller sees it and decrypts using its private key
     - Creates a normal Kubernetes Secret that apps can use

- **Deployment:**
  ```yaml
  Sealed Secrets Controller: 1 replica
  Public Key: Stored in Git (for encrypting)
  Private Key: K8s secret (for decrypting) - BACKUP THIS!
  ```

- **What happens if you lose the private key?**
  - You lose the ability to decrypt old secrets
  - You must create new secrets and reconfigure everything
  - **Solution:** Backup the private key in a vault (physical safe, password manager)

- **Why we need it:** 
  - GitOps workflows: Store infrastructure as code safely
  - Compliance: Auditors want to see "secrets at rest encryption"
  - Rotation: When you change a password, commit the new encrypted version to Git

- **Alternative: External Secrets Operator (ESO)**
  - **What it does:** Syncs secrets from an external vault (like HashiCorp Vault, AWS Secrets Manager) into Kubernetes
  - **Use case:** Large organizations with centralized secret management
  - **Example flow:**
    1. DBA creates database password in HashiCorp Vault
    2. ESO watches Vault every 5 minutes
    3. When password changes, ESO updates the Kubernetes Secret automatically
    4. Pods restart and pick up new password

- **When to use which:**
  - **Sealed Secrets:** Small/medium teams, simple setup, secrets in Git
  - **External Secrets Operator + Vault:** Large enterprises, centralized secret management, compliance requirements

- **Security Best Practices:**
  - Rotate database passwords quarterly
  - Use different credentials for dev/staging/production
  - Never commit plain-text secrets to Git (use git-secrets tool to prevent this)
  - Audit secret access logs monthly

---

## PHASE 2: Advanced Processing (Months 7-12)

**Goal:** Enable complex ETL, transformations, and streaming

### 2.1 Compute Engine

#### Component: Apache Spark
- **Deployment (Spark Operator):**
  ```yaml
  Spark Driver:
    cpu: 4 cores
    memory: 8GB
  
  Spark Executors:
    count: 6 (dynamic allocation)
    cpu: 4 cores each
    memory: 16GB each
  ```
- **Custom Docker Image:**
  - Base: `apache/spark:3.5.0`
  - Add: `iceberg-spark-runtime-3.5`, `nessie-spark-extensions`
  - Add: AWS SDK JARs (for S3/MinIO)

#### Component: dbt Core
- **Deployment:** Run as Airflow task (KubernetesPodOperator)
- **Configuration:**
  ```yaml
  profiles.yml:
    target: prod
    outputs:
      prod:
        type: trino
        host: trino-coordinator
        catalog: iceberg
        schema: gold
  ```
- **Project Structure:**
  ```
  models/
    staging/     -- SQL: SELECT * FROM bronze.*
    marts/       -- SQL: Business logic
    tests/       -- Data quality tests
  ```

---

### 2.2 Real-Time Ingestion

#### Component: Kafka + Strimzi Operator
- **Deployment:**
  ```yaml
  Kafka Brokers: 3 replicas (minimum for production)
  CPU: 4 cores per broker
  Memory: 8GB per broker
  Storage: 500GB per broker (SSD preferred)
  Zookeeper: 3 replicas (managed by Strimzi)
  ```
- **Kafka Connect:**
  ```yaml
  Connectors:
    - Debezium MySQL Source (capture changes)
    - Debezium PostgreSQL Source
    - MinIO Sink Connector (write to S3)
  Workers: 2 replicas
  ```

#### Component: Spark Structured Streaming Jobs
- **Function:** Read from Kafka → Write to Iceberg (via Nessie)
- **Deployment:** Long-running Spark jobs (via Spark Operator)
- **Example Job:**
  ```python
  spark.readStream.format("kafka") \
    .option("subscribe", "db.users") \
    .load() \
    .writeStream.format("iceberg") \
    .option("path", "s3a://bronze/users") \
    .option("checkpointLocation", "s3a://checkpoints/users") \
    .start()
  ```

---

## PHASE 3: Governance & Advanced Analytics (Months 13-18)

### 3.1 Data Catalog

#### Component: OpenMetadata
- **Deployment:**
  ```yaml
  OpenMetadata Server: 2 replicas (4 vCPU, 8GB each)
  ElasticSearch: 3 nodes (for search index)
  MySQL: 1 instance (metadata storage)
  Airflow: Built-in (for metadata ingestion)
  ```
- **Ingestion Connectors:**
  - Trino: Scan all tables, columns, stats
  - Nessie: Pull commit history (lineage)
  - Airflow: Task-level lineage
  - dbt: Model dependencies (via dbt manifest.json)

---

### 3.2 Data Quality

#### Component: Great Expectations

- **What it does (Simple explanation):** Automated "spell-checker" for your data. It checks if data follows rules you define, like "age must be between 0-120" or "email must contain @".

- **Function:** Automated data testing and validation

- **Real-world analogy:** Think of airport security:
  - **Without Great Expectations:** Passengers board planes. Maybe someone brought a weapon? You find out when it's too late.
  - **With Great Expectations:** Every passenger goes through metal detector. If something's wrong, alarm sounds BEFORE they board.

- **Deployment:** Run as Airflow tasks or Spark jobs

- **Example Checks:**
  ```python
  # Check 1: Critical fields must never be empty
  expect_column_values_to_not_be_null("user_id")
  expect_column_values_to_not_be_null("transaction_date")
  
  # Check 2: Values must be in valid range
  expect_column_values_to_be_between("age", 0, 120)
  expect_column_values_to_be_between("price", 0.01, 1000000)
  
  # Check 3: Values must match pattern
  expect_column_values_to_match_regex("email", ".*@.*\\..*")
  
  # Check 4: Unique constraint
  expect_column_values_to_be_unique("order_id")
  
  # Check 5: Foreign key relationship
  expect_column_values_to_be_in_set("country_code", ["US", "CA", "MX"])
  ```

- **Real-world use case:**
  ```
  Scenario: E-commerce company
  
  Monday: Data pipeline runs, loads 10,000 orders
  Great Expectations checks:
    ✓ All orders have customer_id
    ✓ All prices are positive
    ✗ ALERT: 500 orders have order_date in the future!
    ✗ ALERT: 50 orders have negative quantities!
  
  Action: Pipeline stops, data team investigates
  Root cause: Bug in source system introduced yesterday
  Fix: Correct the source code, re-run pipeline
  Result: Bad data never reaches production reports
  ```

- **Integration:** 
  - Results stored in PostgreSQL database
  - Displayed in OpenMetadata (data catalog)
  - Airflow DAG fails if expectations not met

- **How it prevents disasters:**
  - **Finance dashboard** shows revenue = -$1 million → Expectation catches negative values
  - **User report** says 200-year-old customers → Expectation catches age > 120
  - **Compliance** requires all PII to be masked → Expectation checks for SSN patterns

- **Configuration example:**
  ```yaml
  # expectations/orders_table.yaml
  expectations:
    - expectation_type: expect_table_row_count_to_be_between
      kwargs:
        min_value: 1000  # We expect at least 1000 orders per day
        max_value: 100000
    
    - expectation_type: expect_column_mean_to_be_between
      column: price
      kwargs:
        min_value: 10  # Average order value should be $10-$500
        max_value: 500
  ```

- **When to implement:** 
  - Start in Phase 1: Add basic checks (not null, positive values)
  - Expand in Phase 3: Add complex statistical checks

- **Alternatives:** 
  - dbt tests (built into dbt, simpler but less powerful)
  - Soda Core (similar to Great Expectations)
  - Custom SQL checks (manual, harder to maintain)

---

#### Component: Apache Ranger (Optional - Advanced)

- **What it does (Simple explanation):** The "security guard" that decides who can see what data, down to the column and row level.

- **Function:** Fine-grained access control (more advanced than Trino's built-in rules)

- **Real-world analogy:** Think of a hospital's patient records system:
  - **Simple access control (Trino built-in):**
    - Doctors can see patient_records table ✓
    - Nurses can see patient_records table ✓
  
  - **Ranger's fine-grained control:**
    - Doctors see ALL columns (including sensitive diagnosis)
    - Nurses see most columns BUT NOT mental_health_diagnosis
    - Billing department sees name, insurance BUT SSN is masked as XXX-XX-1234
    - Interns (age < 25) can ONLY see their own department's patients (row-level filter)

- **When to Use:** If you need:
  1. **Column masking:** Show only last 4 digits of credit card
  2. **Row-level security:** Users see only their department's data
  3. **Dynamic policies:** "Contractors can't access data from last 30 days"
  4. **Audit trail:** "Who accessed this patient's records?" (required for HIPAA)

- **Deployment:**
  ```yaml
  Ranger Admin: 2 replicas (4 vCPU, 8GB each)
  Ranger Database: PostgreSQL
  Plugins: Install in Trino, Spark, Kafka
  ```

- **Example policies:**
  ```
  Policy 1: Column Masking
  Table: customers
  Column: credit_card
  Rule: MASK_SHOW_LAST_4 for role:analyst
  Result: Analyst sees 1234-5678-9012-3456 as ****-****-****-3456
  
  Policy 2: Row-Level Filter
  Table: sales
  Rule: country = {USER.country} for role:regional_manager
  Result: US manager only sees US sales, UK manager only sees UK sales
  
  Policy 3: Time-Based Access
  Table: financial_reports
  Rule: DENY if current_date < report_date + 30 days AND role:contractor
  Result: Contractors can't see reports until 30 days after publication
  ```

- **Warning:** Complex to operate, requires dedicated admin
  - **Pros:** Meets strict compliance requirements (GDPR, HIPAA, SOX)
  - **Cons:** 
    - Performance overhead (every query checked against policies)
    - Configuration complexity (hundreds of policies to manage)
    - Debugging difficulty (query fails, is it Ranger or Trino?)

- **Decision tree:**
  ```
  Do you need column masking? 
    NO → Use Trino's built-in access control
    YES → Continue...
  
  Do you need row-level security?
    NO → Use Trino's built-in access control
    YES → Continue...
  
  Do you have regulations requiring audit trails?
    NO → Use Trino's built-in access control
    YES → Deploy Ranger
  ```

- **Only add if Trino's access control is insufficient:**
  - Trino can handle: Role-based access, table/schema permissions
  - Trino cannot handle: Column masking, dynamic row filtering, detailed audit logs

- **Alternatives:**
  - Trino's built-in file-based access control (simpler, 80% of use cases)
  - Application-level security (enforce rules in your app, not database)
  - Separate tables (create `customers_pii` and `customers_public` tables)

---

### 3.3 Machine Learning Platform (Optional)

#### Component: MLflow

- **What it does (Simple explanation):** A "lab notebook" for data scientists. It tracks every experiment: which algorithm they tried, what parameters they used, and how accurate the results were.

- **Function:** ML experiment tracking, model registry, model versioning

- **Real-world analogy:** Think of developing a recipe:
  - **Without MLflow:**
    - Chef tries 50 variations of chocolate cake
    - Notes scattered on paper, some lost
    - Can't remember which version was best
    - Can't recreate the winning recipe
  
  - **With MLflow:**
    - Every recipe attempt logged automatically
    - Ingredients (parameters), oven time (training), taste score (accuracy) all recorded
    - Search: "Show me all cakes with >4.5 star rating"
    - Click "Recipe #23" → Get exact instructions to recreate

- **Deployment:**
  ```yaml
  MLflow Server: 1 replica (2 vCPU, 4GB)
  Backend Store: PostgreSQL (stores experiment metadata)
  Artifact Store: MinIO (stores trained models, graphs, logs)
  ```

- **Key Features:**

  1. **Experiment Tracking:**
  ```python
  import mlflow
  
  # Data scientist runs this code
  with mlflow.start_run():
      # Log what they tried
      mlflow.log_param("learning_rate", 0.01)
      mlflow.log_param("num_trees", 100)
      
      # Train model...
      accuracy = train_model()
      
      # Log results
      mlflow.log_metric("accuracy", accuracy)  # 0.87
      mlflow.log_artifact("confusion_matrix.png")
  ```
  - MLflow UI shows: "Run #47 had 87% accuracy with learning_rate=0.01"

  2. **Model Registry:**
  ```python
  # Promote best model to production
  mlflow.register_model(
      model_uri="runs:/abc123/model",
      name="fraud_detection_model"
  )
  
  # Version history:
  # v1 (2024-01): 85% accuracy - ARCHIVED
  # v2 (2024-03): 87% accuracy - PRODUCTION
  # v3 (2024-06): 89% accuracy - STAGING (testing)
  ```

  3. **Model Serving:**
  ```bash
  # Deploy model as REST API
  mlflow models serve -m "models:/fraud_detection_model/Production"
  
  # Application sends request:
  curl http://mlflow-server/predict -d '{"transaction_amount": 5000}'
  # Response: {"fraud_probability": 0.92}
  ```

- **Integration:** 
  - Data scientists read training data from Iceberg tables via Spark
  - Train models (TensorFlow, PyTorch, scikit-learn)
  - Register in MLflow
  - Production apps query MLflow API for predictions

- **Real-world workflow:**
  ```
  Week 1: Data scientist trains 100 model variations
    MLflow logs: Algorithms, hyperparameters, accuracy scores
  
  Week 2: Team reviews MLflow UI
    Best model: Random Forest, 92% accuracy
    Decision: Move to staging
  
  Week 3: A/B test in staging environment
    90% traffic → old model (88% accuracy)
    10% traffic → new model (92% accuracy)
    MLflow tracks: Prediction latency, error rates
  
  Week 4: Promote to production
    Update model version in MLflow: "Production"
    All apps automatically use new model
  ```

- **Why we need it:**
  - **Reproducibility:** "How did we build model v5?" → Check MLflow logs
  - **Collaboration:** Team can see each other's experiments
  - **Governance:** "Which model is in production?" → Check registry
  - **Debugging:** Model accuracy dropped? Compare with previous versions

- **When to add:**
  - You have data scientists on the team
  - You're building predictive models (fraud detection, recommendations, forecasting)
  - You need to deploy models to production applications

- **Don't add if:**
  - Your analytics is purely descriptive (dashboards, reports)
  - No ML use cases on roadmap

- **Alternatives:** 
  - Weights & Biases (W&B) - SaaS, better for deep learning
  - Neptune.ai - SaaS, good for teams
  - Kubeflow - Full ML platform (heavier, includes training orchestration)

---

#### Component: JupyterHub (Optional)

- **What it does (Simple explanation):** Gives each data scientist their own "workspace" where they can write code, run queries, and create charts in a web browser.

- **Function:** Multi-user interactive notebooks for exploratory data analysis

- **Real-world analogy:** Think of a computer lab at university:
  - **Without JupyterHub:** 
    - Each student needs to install Python, libraries, databases on their laptop
    - Works on Windows, breaks on Mac
    - "It works on my machine!" problem
  
  - **With JupyterHub:**
    - Students open browser, get identical environment
    - All tools pre-installed and configured
    - Access from anywhere (office, home, coffee shop)

- **Deployment:**
  ```yaml
  JupyterHub: 1 replica (hub server)
  User Pods: Spawn on-demand when user logs in
  Authentication: Keycloak (OIDC) - same login as Trino/Airflow
  
  User Notebook Specifications:
    Small: 2 vCPU, 4GB RAM (for exploration)
    Medium: 4 vCPU, 8GB RAM (for light ML)
    Large: 8 vCPU, 16GB RAM (for heavy Spark jobs)
  ```

- **What's pre-installed in notebooks:**
  - **Python libraries:** pandas, numpy, scikit-learn, matplotlib
  - **Database connectors:** 
    - Trino connector: Query data lake directly
    - Spark connector: Run distributed processing
  - **ML tools:** TensorFlow, PyTorch, XGBoost
  - **MLflow client:** Log experiments from notebook

- **User workflow:**
  ```python
  # Data scientist opens notebook in browser
  
  # 1. Connect to Trino
  from trino import dbapi
  conn = dbapi.connect(host='trino', catalog='iceberg')
  
  # 2. Query data
  df = pd.read_sql("SELECT * FROM gold.customer_purchases LIMIT 10000", conn)
  
  # 3. Explore data
  df.describe()
  df.plot(kind='scatter', x='age', y='purchase_amount')
  
  # 4. Train model
  from sklearn.ensemble import RandomForestClassifier
  model = RandomForestClassifier()
  model.fit(X_train, y_train)
  
  # 5. Log to MLflow
  mlflow.log_metric("accuracy", model.score(X_test, y_test))
  ```

- **Resource management:**
  - User selects "Small" environment → Gets 4GB RAM pod
  - After 1 hour of inactivity → Pod shuts down automatically (save resources)
  - User logs back in → Pod starts again in 30 seconds

- **Why we need it:**
  - **Standardization:** Everyone uses same Python version, same libraries
  - **Cost-effective:** Users share cluster resources instead of individual workstations
  - **Collaboration:** Share notebooks via Git or JupyterHub's built-in sharing
  - **Security:** Data never leaves the platform (no downloading to laptops)

- **When to add:**
  - You have data scientists or analysts who code in Python/R
  - You want to enable self-service analytics
  - You need to enforce security (data cannot be downloaded)

- **Don't add if:**
  - Analysts only use BI tools (Tableau, Power BI)
  - Small team (<5 people) who are comfortable with local setups

- **Alternatives:**
  - VSCode Server (for users who prefer IDE over notebooks)
  - RStudio Server (if your team uses R instead of Python)
  - Google Colab / Databricks (cloud SaaS options)

- **Best practices:**
  - Set resource quotas per user (prevent one person hogging all CPU)
  - Enable Git integration (save notebooks to version control)
  - Create shared folder for team datasets
  - Regular cleanup: Delete notebooks older than 90 days

---

## Resource Summary by Phase

| Phase | Nodes | Total vCPU | Total RAM | Storage | Team Size |
|-------|-------|------------|-----------|---------|-----------|
| Phase 1 (MVP) | 8 | 104 | 384GB | 4TB | 2-3 engineers |
| Phase 2 (Processing) | 13 | 232 | 768GB | 9TB | 4-5 engineers |
| Phase 3 (Governance) | 16 | 296 | 1TB | 12TB | 5-7 engineers |

---

## Critical Success Factors

### Team Skills Required
- **Phase 1:** Kubernetes basics, SQL, Python
- **Phase 2:** Spark development, streaming concepts
- **Phase 3:** Data governance, ML engineering (optional)

### Operational Runbooks Needed
1. **Disaster Recovery:** How to restore from Velero backup (TEST THIS!)
2. **Scale-Up:** Adding new Trino workers during high load
3. **Incident Response:** Prometheus alert → investigation steps
4. **Security Patching:** Rolling update strategy for each component

### Cost Optimization Tips
- Use **spot instances** for Spark executor nodes (if cloud)
- Enable **MinIO lifecycle policies** (auto-delete bronze data after 90 days)
- Configure **Trino query limits** (prevent runaway queries)
- Use **Horizontal Pod Autoscaler (HPA)** for Trino workers

---

## Phased Deployment Timeline

### Month 1-2: Infrastructure Setup
- [ ] K8s cluster provisioning
- [ ] Install Rancher, Nginx Ingress, cert-manager
- [ ] Deploy Prometheus + Grafana
- [ ] Deploy PostgreSQL clusters

### Month 3-4: Core Lakehouse
- [ ] Deploy MinIO + create buckets
- [ ] Deploy Nessie
- [ ] Deploy Trino + connect to Nessie
- [ ] Deploy Keycloak + configure SSO
- [ ] First query: `SELECT * FROM iceberg.bronze.test_table`

### Month 5-6: Orchestration
- [ ] Deploy Airflow
- [ ] Create first DAG: Manual CSV upload to MinIO → Iceberg table
- [ ] Set up Velero backups
- [ ] Load testing (simulate 100 concurrent Trino queries)

### Month 7-9: Advanced ETL
- [ ] Deploy Spark Operator
- [ ] Migrate heavy transformations from Trino to Spark
- [ ] Deploy dbt + first models

### Month 10-12: Streaming
- [ ] Deploy Kafka + Strimzi
- [ ] Set up Debezium CDC from 1 source database
- [ ] First streaming pipeline: MySQL → Kafka → Iceberg

### Month 13-15: Governance
- [ ] Deploy OpenMetadata
- [ ] Configure lineage ingestion
- [ ] Deploy Great Expectations
- [ ] First data quality dashboard

### Month 16-18: Optimization
- [ ] Tune Iceberg compaction schedules
- [ ] Implement data retention policies
- [ ] Create custom Grafana dashboards
- [ ] Conduct disaster recovery drill

---

## When to Add Each Component

| Component | Add When... | Don't Add If... |
|-----------|-------------|-----------------|
| Kafka | You need sub-second data latency | Hourly batch is acceptable |
| Spark | Trino queries timeout (complex joins) | Queries finish in < 1 minute |
| dbt | Analysts complain about table creation | Only engineers write SQL |
| OpenMetadata | Users can't find tables ("Where's user data?") | < 50 tables in system |
| Great Expectations | Data quality incidents occur weekly | Manual checks are manageable |
| Ranger | Trino's access control is insufficient | Simple role-based access works |
| MLflow | Data scientists ask "Where's model v3?" | No ML workloads |

---

## Final Recommendations

1. **Start Small:** Phase 1 is enough for 80% of use cases. Don't over-engineer early.
2. **Automate Everything:** Use GitOps (ArgoCD/FluxCD) to deploy configs from Git.
3. **Test Backups Monthly:** The day you need a restore is not the day to learn Velero.
4. **Document as You Go:** Future you (or your replacement) will thank you.
5. **Hire for Operations:** This stack needs 24/7 monitoring. Plan for on-call rotation.

**Good luck! You're building something impressive.** 🚀
