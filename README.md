# Nextcloud Enterprise Helm Chart (Nextcloud + Collabora + Redis + MinIO + HPA)

A production-ready Umbrella Helm Chart and Kustomize package for deploying Nextcloud with Collabora Online Office, Redis session caching, MinIO S3 object storage, and Horizontal Pod Autoscaling (HPA: 2 to 6 replicas).

---

## Architecture Overview

* **Nextcloud Web / API**: Auto-scaling (Min: 2, Max: 6 replicas, Target CPU: 70%, Target RAM: 80%)
* **Collabora Online Office**: Auto-scaling (Min: 2, Max: 6 replicas, Target CPU: 75%)
* **Redis Cache**: In-memory session cache and transactional lock sharing across all Nextcloud pods
* **MinIO S3**: Dual-mode Object Storage (Embedded automatic deployment or External/Hosted connection)
* **Database Options**: Support for PostgreSQL, MariaDB/MySQL, and SQLite
* **Longhorn CSI**: Replicated storage for Database and application configuration
* **Traefik Ingress**: Layer 7 routing with automated Let's Encrypt TLS/HTTPS certificates

---

## MinIO S3 Object Storage (Dual Mode Selection)

You can choose whether Helm should deploy a brand-new MinIO instance or connect to an existing hosted S3 / MinIO cluster:

### Mode 1: Connect to Existing / External MinIO (Default)
Set `installEmbedded: false` and point to your endpoint:
```yaml
minio:
  enabled: true
  installEmbedded: false
  endpoint: "minio.minio-system.svc.cluster.local" # Internal or external URL e.g. s3.amazonaws.com
  port: 9000
  bucket: "nextcloud-data"
  accessKey: "admin"
  secretKey: "Admin@1234"
  useSsl: false
```

---

### Mode 2: Auto-Deploy MinIO (Embedded in Chart)
Set `installEmbedded: true` and the chart will deploy MinIO Deployment, PVC, Service, and Ingress automatically:
```yaml
minio:
  enabled: true
  installEmbedded: true
  image:
    repository: "quay.io/minio/aistor/minio"
    tag: "RELEASE.2026-07-24T16-43-31Z"
  storageSize: 20Gi
  storageClass: "longhorn"
  accessKey: "admin"
  secretKey: "Admin@1234"
  consoleDomain: "minio.sengporkeat.com"
  s3Domain: "s3.sengporkeat.com"
```

---

## Database Configuration Matrix (PostgreSQL vs MariaDB vs SQLite)

### Option 1: PostgreSQL (Recommended for Production & HPA)
```yaml
database:
  type: "postgresql"
  username: "admin"
  password: "Admin@1234"
  database: "nextcloud"
  storageSize: 8Gi
```

### Option 2: MariaDB / MySQL
```yaml
database:
  type: "mariadb"
  username: "admin"
  password: "Admin@1234"
  rootPassword: "RootPassword@1234"
  database: "nextcloud"
  storageSize: 8Gi
```

### Option 3: SQLite (Zero External Pods / Embedded)
```yaml
database:
  type: "sqlite"
```
*(No username, password, or storageSize required. SQLite is for single-replica/testing).*

---

## Directory Structure

```text
nextcloud-helmchart/
├── README.md                           # Documentation & deployment instructions
├── .gitignore                          # Clean repository ignore file
│
├── charts/
│   └── nextcloud-helmchart/
│       ├── Chart.yaml                  # Chart metadata & upstream dependencies
│       ├── values.yaml                 # Master configuration (MinIO, HPA 2-6, Ingress, SSL)
│       ├── .helmignore                 # Excludes system/temporary files from Helm packaging
│       ├── assets/                     # Custom branding files (logo.png, background.png, favicon.png)
│       └── templates/
│           ├── _helpers.tpl            # Template naming helpers & labels
│           ├── nextcloud-hpa.yaml      # Nextcloud Autoscaler (Min: 2, Max: 6)
│           ├── collabora-deployment.yaml # Collabora Online Office deployment & service
│           ├── collabora-hpa.yaml      # Collabora Autoscaler (Min: 2, Max: 6)
│           ├── collabora-ingress.yaml  # Collabora Ingress (office.sengporkeat.com + Let's Encrypt)
│           ├── collabora-configmap.yaml # Auto-connect Nextcloud to Collabora WOPI
│           ├── minio-deployment.yaml   # MinIO Dual-Mode Deployment, PVC, Service & Ingress
│           ├── clean-experience-configmap.yaml # Suppresses ads, first-run wizard & empty skeleton
│           └── theming-configmap.yaml  # Applies custom branding colors & embedded images
│
└── kustomize/
    ├── base/
    │   └── kustomization.yaml          # Base Kustomize manifest referencing the Helm chart
    └── overlays/production/
        ├── kustomization.yaml          # Production overlay
        └── custom-values.yaml          # Easily customizable environment overrides
```

---

## Deployment Commands

### Deploy with Helm
```bash
# 1. Download Helm chart dependencies (First time only)
helm dependency build ./charts/nextcloud-helmchart

# 2. Deploy or upgrade the full stack
helm upgrade --install nextcloud ./charts/nextcloud-helmchart \
  --namespace nextcloud-system \
  --create-namespace
```

### Deploy with Kustomize
```bash
kubectl apply -k ./kustomize/overlays/production/ --enable-helm
```

---

## Verification Commands

```bash
# Check all pods are running
kubectl get pods -n nextcloud-system

# Check HPA autoscalers
kubectl get hpa -n nextcloud-system

# Check Ingress & SSL certificates
kubectl get ingress,certificate -n nextcloud-system
```

---

## Teardown / Uninstall

```bash
helm uninstall nextcloud -n nextcloud-system
```
