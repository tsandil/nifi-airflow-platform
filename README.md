# NiFi + Airflow Platform on Kubernetes

End-to-end data platform infrastructure for learning and testing production patterns with Apache Airflow, NiFi, and PostgreSQL on Kubernetes.

## What This Is

A local Kubernetes (Kind) cluster running a complete ELT pipeline stack:
- **Apache Airflow 2.7.1** - Workflow orchestration
- **Apache NiFi 1.23.2** - Data flow processing
- **PostgreSQL 14** - Pipeline state management
- **Custom Package**: [`airflow-nifi-pipeline-utils`](https://test.pypi.org/project/airflow-nifi-pipeline-utils/) - My package for coordinating Airflow + NiFi workflows

## Architecture
```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│   Airflow   │────▶│    NiFi     │────▶│  PostgreSQL  │
│  (airflow)  │     │   (nifi)    │     │  (postgres)  │
└─────────────┘     └─────────────┘     └──────────────┘
  Orchestration      Data Processing     State Tracking
  localhost:8081     localhost:8080      pipeline_info DB
```

**Namespaces:**
- `airflow` - Airflow components (scheduler, webserver, triggerer)
- `nifi` - NiFi single-node cluster
- `postgres` - PostgreSQL with `pipeline_info` schema

## Current Status

**Infrastructure Complete**
- Kind cluster running with 3 namespaces
- All services deployed and healthy
- Package installed in Airflow
- Ready for DAG development

## What I Learned

### Kubernetes & Infrastructure
- Kind cluster configuration with port mappings
- Namespace design for production patterns
- StatefulSets vs Deployments (when to use each)
- PersistentVolumeClaims and storage classes
- Service discovery (ClusterIP, NodePort, Headless)
- Debugging pod startup issues (Pending, CrashLoopBackOff)
- ConfigMaps for application configuration

### Helm
- Helm chart customization via values.yaml
- Chart versioning (matching app versions)
- Troubleshooting failed installations
- Managing dependencies (cert-manager for nifikop)

### Service-Specific
- Airflow LocalExecutor setup
- External PostgreSQL configuration for Airflow metadata
- NiFi deployment without operator (StatefulSet approach)
- PostgreSQL init scripts via ConfigMaps
- Installing Python packages from TestPyPI in Airflow

### Debugging Skills
- Reading pod events and logs
- Understanding init containers
- Storage class compatibility (ReadWriteOnce vs ReadWriteMany)
- Connection troubleshooting between services

## Prerequisites

- Docker Desktop
- Kind (Kubernetes in Docker)
- kubectl
- Helm 3
- macOS (tested on M3 Pro)

## Setup Instructions

### 1. Create Kind Cluster
```bash
kind create cluster --config kind/cluster-config.yaml
```

### 2. Install cert-manager (required for NiFi)
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

### 3. Deploy PostgreSQL
```bash
kubectl create namespace postgres
kubectl apply -f kubernetes/postgres/
```

Verify:
```bash
kubectl get pods -n postgres
kubectl exec -n postgres postgres-0 -- psql -U postgres -d pipeline_info -c "\dt state.*"
```

### 4. Deploy NiFi
```bash
kubectl create namespace nifi
kubectl apply -f kubernetes/nifi/
```

Access NiFi: http://localhost:8080/nifi (via port-forward if needed)

### 5. Deploy Airflow
```bash
kubectl create namespace airflow

# Create airflow database
kubectl exec -n postgres postgres-0 -- psql -U postgres -c "CREATE DATABASE airflow;"

# Install Airflow
helm repo add apache-airflow https://airflow.apache.org
helm install airflow apache-airflow/airflow \
  --namespace airflow \
  --version 1.11.0 \
  --values kubernetes/airflow/values.yaml \
  --timeout 10m

# Install custom package
kubectl exec -n airflow airflow-scheduler-0 -c scheduler -- pip install \
  --index-url https://test.pypi.org/simple/ \
  --extra-index-url https://pypi.org/simple \
  airflow-nifi-pipeline-utils==1.1.2
```

Access Airflow:
```bash
kubectl port-forward -n airflow svc/airflow-webserver 8081:8080
```
Then go to: http://localhost:8081 (admin/admin)

### 6. Configure Airflow Connection

In Airflow UI: Admin → Connections → Add
- **Connection Id**: `pipeline_postgres`
- **Connection Type**: `Postgres`
- **Host**: `postgres.postgres.svc.cluster.local`
- **Schema**: `pipeline_info`
- **Login**: `postgres`
- **Password**: `postgres`
- **Port**: `5432`

## Accessing Services

| Service | URL | Credentials |
|---------|-----|-------------|
| Airflow UI | http://localhost:8081 | admin / admin |
| NiFi UI | http://localhost:8080/nifi | (no auth) |
| PostgreSQL | postgres.postgres.svc.cluster.local:5432 | postgres / postgres |

## Verification
```bash
# Check all pods
kubectl get pods --all-namespaces

# Verify package installation
kubectl exec -n airflow airflow-scheduler-0 -c scheduler -- pip list | grep airflow_nifi

# Check PostgreSQL schema
kubectl exec -n postgres postgres-0 -- psql -U postgres -d pipeline_info -c "\dt state.*"
```

## Next Steps

- [ ] Create test DAG using `airflow-nifi-pipeline-utils`
- [ ] Configure NiFi ListenHTTP processor
- [ ] Test end-to-end pipeline flow
- [ ] Add monitoring/logging

## Project Structure
```
nifi-airflow-platform/
├── kind/
│   └── cluster-config.yaml          # Kind cluster definition
├── kubernetes/
│   ├── postgres/
│   │   ├── 01-configmap.yaml        # Init scripts with pipeline_info DDL
│   │   ├── 02-statefulset.yaml      # PostgreSQL deployment
│   │   └── 03-service.yaml          # PostgreSQL service
│   ├── nifi/
│   │   ├── 01-configmap.yaml        # NiFi configuration
│   │   ├── 02-statefulset.yaml      # NiFi deployment
│   │   ├── 03-service-headless.yaml # Internal service
│   │   └── 04-service-nodeport.yaml # External access
│   └── airflow/
│       └── values.yaml              # Helm chart customization
└── README.md
```

## Resource Usage

Optimized for MacBook M3:
- PostgreSQL: ~500MB RAM, 5GB storage
- NiFi: ~1-1.5GB RAM, 5GB storage  
- Airflow: ~1.5GB RAM total (scheduler + webserver + triggerer), 5GB storage
- Total: ~3-3.5GB RAM, 15GB disk

## Cleanup
```bash
# Delete everything
kind delete cluster --name nifi-airflow-dev

# Or selectively
helm uninstall airflow -n airflow
kubectl delete namespace nifi postgres airflow
```

## Troubleshooting

### Pods stuck in Pending
- Check PVC status: `kubectl get pvc -n <namespace>`
- Check events: `kubectl get events -n <namespace> --sort-by='.lastTimestamp'`

### Database connection errors
- Verify database exists: `kubectl exec -n postgres postgres-0 -- psql -U postgres -l`
- Check service DNS: `kubectl get svc -n postgres`

### Package not found in Airflow
- Manually install: See step 5 above
- Check logs: `kubectl logs -n airflow airflow-scheduler-0 -c scheduler`

## Learning Resources

- [Airflow Documentation](https://airflow.apache.org/docs/)
- [NiFi Documentation](https://nifi.apache.org/docs.html)
- [Kind Documentation](https://kind.sigs.k8s.io/)
- [My Package on TestPyPI](https://test.pypi.org/project/airflow-nifi-pipeline-utils/)

## License

MIT