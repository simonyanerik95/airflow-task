# Apache Airflow Helm Chart for Production

This repository contains the customized `values-prod.yaml` file for deploying Apache Airflow in a Kubernetes environment using Helm. The configuration is optimized for a production environment and includes customizations such as the use of KubernetesExecutor, external PostgreSQL, secure access via TLS, and integration with AWS Secrets Manager.

The setup is based on the [Apache Airflow Production Guide](https://airflow.apache.org/docs/helm-chart/stable/production-guide.html), which was used as a reference for modifying the default `values.yaml` to suit the needs of a production environment.

## Key Features

- **KubernetesExecutor**: Enables distributed task execution using Kubernetes pods, providing scalability and isolation for Airflow tasks.
- **External PostgreSQL**: Uses an external PostgreSQL database (e.g., AWS RDS) for the Airflow metadata database to ensure stability, performance, and persistence.
- **PgBouncer Enabled**: Connection pooling with PgBouncer is enabled to efficiently manage database connections.
- **AWS Secrets Manager Integration via External Secrets**: Airflow secrets, connections, and variables are securely managed using External Secrets, which retrieves them from AWS Secrets Manager.
- **TLS and Secure Ingress**: Secure external access to the Airflow web UI using Nginx ingress with Cert-Manager handling TLS certificates.
- **DAG Synchronization**: Continuous synchronization of DAGs from a private GitHub repository using GitSync with SSH authentication.
- **Persistent Logs**: Logs are persisted using an existing Persistent Volume Claim (PVC) to retain logs across restarts.
- **DNS Configuration with Route53 and External DNS**: Route53 DNS integration allows external access to the Airflow UI with automatic DNS record creation using External DNS.

## Prerequisites

- EKS cluster up and running
- Helm 3.x
- Nginx ingress controller
- External PostgreSQL (for instance AWS RDS) for metadata database
- AWS Secrets Manager for secrets (via External Secrets)
- Route53 DNS for domain management
- Install **Cert-Manager** to handle automatic TLS certificates in your Kubernetes cluster
- Install **External Secrets** for syncing secrets from AWS Secrets Manager into your Kubernetes cluster
- **External DNS** is required to automatically create and manage DNS records in Route53. Make sure your EKS cluster has the necessary permissions to access Route53.

## Installation

1. Clone the repository:
    ```bash
    git clone https://github.com/simonyanerik95/airflow-task.git
    cd airflow
    ```

2. Configure the following secrets in your Kubernetes cluster:

   - **Webserver Secret Key**:
   - **PostgreSQL Metadata Database Secret**:
   - **Git Sync SSH Key**:

3. Deploy Airflow using Helm:
    ```bash
    helm repo add apache-airflow https://airflow.apache.org
    helm install airflow apache-airflow/airflow -f values-prod.yaml
    ```

4. Verify the deployment:
    ```bash
    kubectl get pods -n airflow
    ```

## Configuration Details

### Why KubernetesExecutor?

We chose the KubernetesExecutor because it allows Airflow tasks to be run as individual pods in a Kubernetes cluster. This enables:

- **Scalability**: Dynamically scales the number of workers based on the load.
- **Isolation**: Each task runs in its own pod, providing better isolation and fault tolerance.
- **Resource Efficiency**: Allows for fine-grained resource management (CPU, Memory) per task.

### Worker Logs in Airflow Console

Worker logs are configured to appear in the Airflow web UI using Airflow's default logging configuration. Logs are stored in a persistent volume claim (`opsfleet-airflow-logs-pvc`) and are accessible from the Airflow console. By retaining logs in a PVC, logs are available across pod restarts, ensuring visibility of both current and past logs.

### DAG Deployment via GitSync

We have chosen to deploy DAGs to Airflow using GitSync. This method allows DAGs to be automatically synchronized from a GitHub repository every 60 seconds. GitSync provides:

- **Version Control**: DAGs are versioned and managed within the Git repository, allowing for easy tracking of changes.
- **Simplicity**: Developers can push changes directly to the GitHub repository, and DAGs will automatically update in Airflow.
- **Security**: GitSync uses SSH authentication for secure access to the GitHub repository.
- **Consistency**: All Airflow workers will have the latest DAGs synchronized in real-time.

### Executor Configuration

- **KubernetesExecutor** is used to launch tasks in Kubernetes pods, offering scalability and flexibility.
- `allowPodLaunching: true` allows the creation of new pods to execute tasks dynamically.

### Database Configuration

- **External PostgreSQL** is recommended for production. The built-in PostgreSQL is disabled in this setup.
- The database connection string is securely managed through a Kubernetes secret (`opsfleet-db-secret`).

### PgBouncer Configuration

- PgBouncer is enabled to pool database connections, reducing the load on the PostgreSQL database.
  - `maxClientConn: 100`: Sets the maximum number of client connections to PgBouncer.
  - `metadataPoolSize: 10`: Pool size for connections to the metadata database.
  - `resultBackendPoolSize: 5`: Pool size for result backend database connections.

### Ingress Configuration

- Nginx ingress controller is used to expose the Airflow web UI with TLS enabled.
- **Cert-Manager** is used for automatic certificate provisioning for TLS.
- Set your hostname in the `hosts` section under `ingress.web.hosts`.

### Persistent Logs

- Logs are persisted using an existing Persistent Volume Claim (`opsfleet-airflow-logs-pvc`), ensuring logs are available even after pod restarts.

### Security Contexts

- Pods are configured with non-root users (`runAsUser: 50000`) for enhanced security.
- Privilege escalation is disabled (`allowPrivilegeEscalation: false`) for the containers.

### AWS Secrets Manager Integration

- Airflow is configured to use AWS Secrets Manager to store connections and variables securely using External Secrets.

### DNS Configuration with External DNS

- **External DNS** is installed to automatically create and manage DNS records in Route53, ensuring that Airflow's web UI is reachable through a domain name managed in Route53.
