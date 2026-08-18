# Nextcloud Enterprise Helm Chart (Nextcloud + Collabora + Redis + MinIO + ClamAV + Notify Push + HPA)

A production-grade Umbrella Helm Chart and Kustomize package for deploying Nextcloud with Collabora Online Office, Redis session/locking, MinIO S3 object storage, Notify Push WebSockets, Preview Pre-generator, ClamAV Real-time Antivirus, and Horizontal Pod Autoscaling.

---

## 🏛️ Infrastructure Architecture (Mermaid Diagram)

```mermaid
flowchart TB
    %% ==========================================
    %% CLIENT & TRAFFIC ENTRY
    %% ==========================================
    User(["🌐 Client Apps & Browsers\n(Web, Desktop, iOS, Android)"]):::clientNode

    %% ==========================================
    %% INGRESS & EDGE ROUTING
    %% ==========================================
    subgraph IngressGateway ["  🛡️ Traefik Ingress Gateway (Edge TLS / Router)  "]
        direction TB
        Traefik["Traefik Ingress Controller\n(Port 80/443 + Let's Encrypt TLS)"]:::ingressNode
        
        subgraph Middlewares [" Traefik Security & Routing Middlewares "]
            SecHeaders["🔒 A+ Security Headers\n• HSTS (31536000s, preload)\n• X-Frame: SAMEORIGIN\n• X-Content-Type: nosniff"]:::midNode
            Rewrites["🔀 Endpoint Rewrites\n• /.well-known/(caldav|carddav)\n• /.well-known/(webfinger|nodeinfo)"]:::midNode
        end
    end

    %% ==========================================
    %% CORE SERVICES
    %% ==========================================
    subgraph KubernetesCluster ["  ☁️ Kubernetes Cluster (Namespace: nextcloud-system)  "]
        
        subgraph AppTier [" 📱 Application & Microservices Layer "]
            NC["💻 Nextcloud Core (PHP 8.5 / Apache)\n• OPcache 256MB / 30,000 files\n• In-Memory APCu & Redis Cache\n• Theme: Unity Drive"]:::ncNode
            Collabora["📄 Collabora Office (CODE)\n• Jailed Bind Mounts (SYS_ADMIN)\n• Warm Workers (num_prespawn=2)\n• Hardware Concurrency (2 Threads)"]:::officeNode
            NotifyPush["⚡ Notify Push Daemon (Rust)\n• Persistent WebSocket Stream\n• Instant 0ms Sync & Notifications"]:::pushNode
            ClamAV["🛡️ ClamAV Antivirus Daemon\n• Real-Time File Stream Scanning\n• Auto-Updated Virus Signatures"]:::secNode
            PreviewCron["🖼️ Preview Generator (CronJob)\n• Pre-renders High-Res Images (15m)\n• 60 FPS Fluid Gallery Scrolling"]:::cronNode
        end

        subgraph StorageTier [" 💾 Persistent Storage & Caching Layer "]
            Redis["⚡ Redis Master Cache\n• Distributed Session & File Locking\n• Real-Time Pub/Sub Message Bus"]:::redisNode
            Postgres["🐘 PostgreSQL 17 Primary\n• Relational Application Metadata\n• 8Gi Longhorn Replicated Volume"]:::dbNode
            MinIO["📦 MinIO S3 Object Storage\n• Primary File Storage (nextcloud-data)\n• 20Gi Longhorn Replicated Volume"]:::s3Node
        end

    end

    %% ==========================================
    %% TRAFFIC ROUTING EDGES
    %% ==========================================
    User ==>|"HTTPS / TLS 1.3"| Traefik
    Traefik --- SecHeaders
    Traefik --- Rewrites

    Traefik -->|"Path: /"| NC
    Traefik -->|"Path: /push (WS Upgrade)"| NotifyPush
    Traefik -->|"Host: office.sengporkeat.com"| Collabora
    Traefik -->|"Host: minio.sengporkeat.com"| MinIO

    %% ==========================================
    %% SERVICE-TO-SERVICE INTERACTIONS
    %% ==========================================
    NC <-->|"TCP 6379 (Locks / Cache)"| Redis
    NC <-->|"TCP 5432 (SQL)"| Postgres
    NC <-->|"S3 API (Port 9000)"| MinIO
    NC <-->|"WOPI Protocol (Port 9980)"| Collabora
    NC -->|"TCP 3310 (Real-Time File Scan)"| ClamAV
    NotifyPush <-->|"Pub/Sub Stream"| Redis
    NotifyPush <-->|"Auth & Queries"| Postgres
    PreviewCron -.->|"Executes 'preview:pre-generate'"| NC

    %% ==========================================
    %% COLOR SCHEMES & STYLES
    %% ==========================================
    classDef clientNode fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc;
    classDef ingressNode fill:#0284c7,stroke:#bae6fd,stroke-width:2px,color:#ffffff;
    classDef midNode fill:#0369a1,stroke:#7dd3fc,stroke-width:1px,color:#ffffff;
    classDef ncNode fill:#7c3aed,stroke:#ddd6fe,stroke-width:2px,color:#ffffff;
    classDef officeNode fill:#059669,stroke:#a7f3d0,stroke-width:2px,color:#ffffff;
    classDef pushNode fill:#d97706,stroke:#fde68a,stroke-width:2px,color:#ffffff;
    classDef secNode fill:#dc2626,stroke:#fecaca,stroke-width:2px,color:#ffffff;
    classDef cronNode fill:#4f46e5,stroke:#c7d2fe,stroke-width:2px,color:#ffffff;
    classDef redisNode fill:#b91c1c,stroke:#fca5a5,stroke-width:2px,color:#ffffff;
    classDef dbNode fill:#1d4ed8,stroke:#bfdbfe,stroke-width:2px,color:#ffffff;
    classDef s3Node fill:#0f766e,stroke:#99f6e4,stroke-width:2px,color:#ffffff;
```

---

## 🌟 Key Features & Enterprise Capabilities

* **⚡ Ultra-Low Latency Notify Push (WebSockets)**: Dedicated Rust WebSocket microservice (`icewind1991/notify_push`) reducing background HTTP requests by 80% with instantaneous UI updates across all clients.
* **🖼️ Preview Pre-generator**: Background Kubernetes CronJob pre-rendering photo/document previews to deliver 60 FPS smooth scrolling in galleries.
* **🛡️ ClamAV Real-Time Antivirus Protection**: Live stream inspection of every uploaded file rejecting malware and ransomware before storing to disk.
* **📄 High-Performance Collabora Office**: Fully integrated with warm process pre-spawning (`num_prespawn_children=2`), Linux jail bind-mounting, and suppressed server audit warnings.
* **🔒 A+ Enterprise Security**: Pre-configured Traefik middleware enforcing 1-year HSTS preloading, anti-clickjacking (`SAMEORIGIN`), XSS filtering, MIME sniffing protection, and WebDAV/WebFinger rewrites.
* **🚀 Turbocharged PHP & Memory Caching**: OPcache (256MB / 30,000 files), APCu in-memory cache, and Redis distributed session locking.
* **💾 MinIO S3 Object Storage**: Full S3 primary storage with automated bucket lifecycle management.
* **📈 Elastic Auto-Scaling (HPA)**: Kubernetes autoscalers for both Nextcloud web and Collabora Office workloads.

---

## 📁 Repository & Directory Layout

```text
nextcloud-helm-chart/
├── README.md                           # Documentation & architecture diagrams
├── charts/
│   └── nextcloud-helmchart/
│       ├── Chart.yaml                  # Umbrella chart metadata & upstream dependencies
│       ├── values.yaml                 # Master configuration (MinIO, HPA, Caching, SSL, ClamAV)
│       ├── assets/                     # Custom branding (logo.png, background.png, favicon.png)
│       └── templates/
│           ├── _helpers.tpl            # Template naming helpers & labels
│           ├── rbac.yaml               # ServiceAccount, Role & RoleBindings for background jobs
│           ├── security-headers.yaml   # Traefik Middlewares (HSTS A+, WebDAV, WebFinger)
│           ├── notify-push-deployment.yaml # High-performance Rust WebSocket daemon & IngressRoute
│           ├── preview-generator-cronjob.yaml # 15-minute background preview pre-rendering CronJob
│           ├── clamav-deployment.yaml  # ClamAV daemon microservice & ClusterIP service
│           ├── collabora-deployment.yaml # Collabora Online Office deployment & service
│           ├── collabora-hpa.yaml      # Collabora Autoscaler (Min: 1, Max: 4)
│           ├── minio-deployment.yaml   # MinIO Dual-Mode Deployment, PVC, Service & Ingress
│           ├── performance-configmap.yaml # APCu, Redis, & PHP OPcache configurations
│           └── theming-job.yaml        # Post-install hook automating apps, themes & branding
│
└── kustomize/
    ├── base/
    │   └── kustomization.yaml          # Base Kustomize manifest referencing the Helm chart
    └── overlays/
        ├── production/
        │   ├── kustomization.yaml      # Production overlay
        │   └── custom-values.yaml      # Production overrides (Postgres, Redis, MinIO, ClamAV)
        └── dev/
            ├── kustomization.yaml      # Dev overlay
            └── custom-values.yaml      # Lightweight Dev overrides (SQLite, single replica)
```

---

## 🚀 Deployment & Upgrades

### Deploy / Upgrade Production:
```bash
git pull
helm upgrade --install nextcloud ./charts/nextcloud-helmchart \
  --namespace nextcloud-system \
  --create-namespace \
  -f ./kustomize/overlays/production/custom-values.yaml
```

---

## 🔍 Verification & Health Checks

```bash
# Check all running pods, services, and cronjobs
kubectl get pods,svc,cronjob,ingress -n nextcloud-system

# Verify Nextcloud Core Integrity
kubectl exec -n nextcloud-system $(kubectl get pod -n nextcloud-system -l app.kubernetes.io/name=nextcloud -o jsonpath='{.items[0].metadata.name}') -c nextcloud -- su -s /bin/bash www-data -c "php occ integrity:check-core"

# Test ClamAV Antivirus Daemon
kubectl logs -n nextcloud-system pod/$(kubectl get pod -n nextcloud-system -l app.kubernetes.io/name=clamav -o jsonpath='{.items[0].metadata.name}') --tail=20

# Trigger Manual Preview Pre-generation
kubectl create job --from=cronjob/nextcloud-preview-generator preview-manual -n nextcloud-system
```
