# Autonomous Infrastructure and Architecture

Project scanned from: `D:\CSTADPreUniversityTraining\ITP\a8sfinallProject`

This document describes the infrastructure and architecture of the Autonomous project based on the project folders and configuration files in the workspace.

## 1. Infrastructure Description

Autonomous is built as a self-service platform that runs on Kubernetes and is managed with a GitOps operating model. The project is split into multiple folders that represent the product application, platform services, database platform, infrastructure automation, monitoring, security, and network access.

At the application layer, the platform has a Next.js frontend and admin interface. The main frontend is in `a8s-frontend`, and the admin/documentation-facing apps are in `a8s-admin` and `a8s-documentation`. These apps are containerized with Docker and expose the user interface, API routes, authentication callbacks, dashboard pages, project pages, database deployment pages, monitoring pages, logs, image scanning, and GitHub integration screens.

The control-plane backend lives in `a8s-backend`. It is a Spring Boot application written in Java. It provides secured REST and WebSocket APIs, handles authentication-aware actions, manages projects, provisions databases, runs backup and restore operations, integrates with Kubernetes, talks to MinIO object storage, connects to Git providers, triggers Jenkins/SonarQube workflows, and exposes monitoring and notification services.

The persistence layer has two important parts. First, the platform itself uses PostgreSQL from the `a8s-db` Helm chart for application data. Second, user-facing managed database deployments are handled by the database cluster platform in `ab-cluster`, `db-cluster`, `database-cluster`, and `db-cluster-gitops`. The database platform supports PostgreSQL, MySQL, MongoDB, Redis, and Cassandra through Kubernetes operators and Helm charts. Persistent storage is configured with Longhorn-backed PVCs, and database backups are designed to use MinIO/S3-compatible object storage.

The infrastructure layer is Kubernetes-first. The `a8s-infra` folder contains automation for Kubernetes setup, ingress creation, CSI drivers, metrics server, Cilium networking, and Ansible/Kubespray-style infrastructure tasks. The `ingressSetting` folder documents a Traefik TCP gateway that runs as a host-networked DaemonSet on Kubernetes nodes and exposes database protocols through ports such as `15432`, `13306`, `16379`, `17017`, and `19042`.

GitOps is handled by Argo CD. The `db-cluster-gitops` repo contains an `ApplicationSet` that watches user database values and syncs the `ab-cluster/db-cluster` Helm chart into user namespaces. The backend creates or updates per-user values files, Git commits and pushes them, and Argo CD creates, updates, or prunes the Kubernetes resources automatically.

Observability is handled through Prometheus, Grafana, Thanos, Loki, Alloy, and alerting rules. The `multi-cluster-monitoring` folder describes Prometheus plus Thanos sidecars in source clusters, with centralized Thanos Query, Store Gateway, Grafana, and MinIO in the main monitoring cluster. The `namespace-per-user-log` folder adds namespace-level quotas, RBAC, log streaming, Loki/Alloy configuration, and quota alerts. The `thanos-client` folder contains manifests for exposing or connecting Thanos components across clusters.

Security and operations are also represented in the workspace. `trivy` contains container/image vulnerability scanning support. `Wireguard-setup` documents VPN/private network access. `share-lib-defetchdojo` provides shared Jenkins library support for DefectDojo-style security integration. CI/CD automation appears in GitHub workflow files, Jenkins files, and the Jenkins setup image in the root project folder.

In short, Autonomous runs as a Kubernetes-hosted platform composed of frontend containers, a Spring Boot control plane, PostgreSQL platform data storage, managed database clusters, GitOps delivery, object storage, observability services, ingress/TCP gateways, and CI/CD/security tooling.

## 2. Architecture Flow Description

The user starts in the Autonomous frontend. Protected dashboard paths are guarded by Better Auth session logic, and authentication is connected to Keycloak and OAuth providers such as Google or GitHub. After login, the frontend and Next.js API routes call the Spring Boot backend through secured REST or WebSocket APIs.

The backend acts as the main control plane. It receives requests for workspace setup, project creation, monolithic or microservice deployments, database deployments, backup operations, logs, metrics, image scanning, notifications, profile management, and admin functions. The backend enforces authentication and ownership, persists platform state, and coordinates the external systems required to perform the requested action.

For application deployment flows, the backend integrates with Git providers, Jenkins automation, Kubernetes, Helm, SonarQube, and image scanning services. A user can import or create a project from the frontend, the backend prepares deployment metadata, CI/CD builds and scans container images, and Kubernetes receives the runtime workloads. Runtime status, logs, metrics, releases, domains, and operations are then surfaced back to the frontend.

For database deployment flows, the user chooses a database engine and configuration from the UI. The backend converts the request into a user-specific GitOps values file. It commits and pushes that file to the GitOps repository. Argo CD ApplicationSet detects the new or changed path, renders the `ab-cluster/db-cluster` Helm chart, creates the user namespace if required, and deploys the selected database engine with its operator, services, secrets, persistent volumes, monitoring, and optional backup settings.

For external database access, the database service can be exposed through a shared Traefik TCP gateway or HAProxy-style external TCP proxy. DNS points to Kubernetes node public IPs, firewall rules allow selected database ports, Traefik routes by TCP entry point and SNI, and traffic is forwarded to the correct database service inside the user's namespace.

For observability, workloads and database clusters expose metrics to Prometheus. Thanos sidecars make Prometheus data queryable across clusters. Thanos Query gives Grafana a single query endpoint for multi-cluster dashboards, while Thanos Store Gateway reads historical blocks from MinIO. Logs can flow through Alloy into Loki. Alerts and namespace quota rules provide operational feedback, and the backend/frontend can display monitoring and notification data.

The main architecture pattern is therefore:

1. User action in Next.js frontend.
2. Authentication and session validation through Better Auth, Keycloak, and OAuth.
3. Backend control-plane API receives the request.
4. Backend persists state and coordinates platform services.
5. CI/CD, GitOps, Kubernetes, Helm, operators, and storage execute the real infrastructure change.
6. Observability and status data flow back through backend APIs and frontend dashboards.

## 3. Infrastructure Diagram

```mermaid
flowchart TB
    user["Users / Admins"]

    subgraph frontend["Frontend Layer"]
        fe["Autonomous Web App<br/>Next.js / React"]
        admin["A8S Admin App<br/>Next.js"]
        proxy["Next.js API Routes<br/>BFF / Proxy Layer"]
        authui["Better Auth Session<br/>Management"]
    end

    subgraph access["Access Management"]
        keycloak["Keycloak<br/>OIDC Provider"]
        oauth["Social OAuth Providers<br/>Google / GitHub"]
    end

    subgraph core["Core Service Layer"]
        backend["A8S Core Backend<br/>Spring Boot / Java"]
        modules["Control-Plane Modules<br/>Projects / Workspaces / Deployments<br/>Single DB / DB Cluster / Backups"]
        integrations["Platform Integrations<br/>Git Providers / Jenkins / SonarQube<br/>Kubernetes / Helm / MinIO"]
    end

    subgraph cicd["CI/CD and Security"]
        github["GitHub Repositories<br/>Source and GitOps"]
        jenkins["Jenkins Automation<br/>Build / Deploy"]
        scanner["Trivy / DefectDojo<br/>Image and Security Scans"]
        registry["Container Registry<br/>Application Images"]
    end

    subgraph persistence["Persistence and Storage"]
        appdb["Platform PostgreSQL<br/>a8s-db Helm Chart"]
        minio["MinIO / S3 Storage<br/>Backups and Metrics Blocks"]
        pvc["Longhorn PVCs<br/>Database Data Volumes"]
    end

    subgraph infra["Kubernetes Infrastructure"]
        appcluster["Kubernetes Application Cluster<br/>Frontend / Backend / Platform Apps"]
        dbcluster["Kubernetes Database Cluster<br/>Managed DB Engines"]
        ingress["Ingress and TCP Gateway<br/>Nginx / Traefik / HAProxy"]
        operators["Database Operators<br/>CNPG / Percona / Redis / K8ssandra"]
        network["Network and Access<br/>Cilium / WireGuard / Firewall / DNS"]
    end

    subgraph gitops["GitOps Engine"]
        gitopsrepo["db-cluster-gitops<br/>Per-user values.yaml"]
        argocd["Argo CD ApplicationSet<br/>Continuous Sync"]
        helm["Helm Charts<br/>ab-cluster / db-cluster"]
    end

    subgraph obs["Observability"]
        prom["Prometheus<br/>Metrics Collection"]
        thanos["Thanos Query / Sidecar / Store<br/>Multi-cluster Metrics"]
        grafana["Grafana<br/>Dashboards"]
        logs["Loki / Alloy<br/>Log Streaming"]
        alerts["Alert Rules<br/>Quota and Platform Alerts"]
    end

    user --> fe
    user --> admin
    fe --> proxy
    admin --> proxy
    proxy --> authui
    authui --> keycloak
    authui --> oauth
    proxy --> backend

    backend --> modules
    modules --> integrations
    backend --> appdb
    backend --> minio
    integrations --> github
    integrations --> jenkins
    jenkins --> scanner
    jenkins --> registry
    registry --> appcluster

    backend --> appcluster
    backend --> gitopsrepo
    gitopsrepo --> argocd
    argocd --> helm
    helm --> dbcluster
    dbcluster --> operators
    operators --> pvc
    dbcluster --> ingress
    ingress --> network

    appcluster --> prom
    dbcluster --> prom
    prom --> thanos
    thanos --> grafana
    thanos --> minio
    appcluster --> logs
    dbcluster --> logs
    logs --> alerts
    prom --> alerts
    alerts --> backend

    classDef layer fill:#f7fbff,stroke:#6b8cff,stroke-width:1.5px,color:#0f172a
    classDef core fill:#ecfdf5,stroke:#22c55e,stroke-width:1.5px,color:#064e3b
    classDef infra fill:#eef2ff,stroke:#6366f1,stroke-width:1.5px,color:#1e1b4b
    classDef ops fill:#fff7ed,stroke:#f97316,stroke-width:1.5px,color:#7c2d12
    classDef store fill:#fef2f2,stroke:#ef4444,stroke-width:1.5px,color:#7f1d1d

    class fe,admin,proxy,authui,keycloak,oauth layer
    class backend,modules,integrations core
    class appcluster,dbcluster,ingress,operators,network,argocd,helm,gitopsrepo infra
    class github,jenkins,scanner,registry,prom,thanos,grafana,logs,alerts ops
    class appdb,minio,pvc store
```

## 4. Architecture Diagram

```mermaid
flowchart TB
    user["User"]

    subgraph ui["Product Experience"]
        pages["Next.js Pages<br/>Dashboard / Projects / Deployments<br/>Databases / Logs / Monitoring"]
        routes["Next.js API Routes<br/>Frontend BFF"]
        session["Better Auth Session<br/>Protected Routes"]
    end

    subgraph identity["Identity"]
        keycloak["Keycloak"]
        oauth["OAuth Providers<br/>Google / GitHub"]
    end

    subgraph api["Backend Control Plane"]
        backend["Spring Boot Backend<br/>Autonomous Core API"]
        security["Security / Ownership / Rate Limit"]
        domain["Domain Services"]
        ws["WebSocket / Live Updates"]
    end

    subgraph domains["Backend Feature Domains"]
        workspaces["Workspaces and Entitlements"]
        projects["Projects<br/>Monolithic and Microservice"]
        dbdeploy["Database Deployments<br/>Single DB and DB Cluster"]
        backup["Backup / Restore"]
        console["Database Console APIs"]
        monitoring["Monitoring / Alerts"]
        notifications["Notifications"]
        payments["Payments<br/>Stripe / Bakong KHQR"]
    end

    subgraph state["State and Artifacts"]
        postgres["Platform PostgreSQL<br/>Users / Projects / Deployments"]
        minio["MinIO<br/>Backup Artifacts"]
        git["Git Repositories<br/>Source and GitOps Values"]
        images["Container Registry<br/>Built Images"]
    end

    subgraph delivery["Delivery and Runtime"]
        jenkins["Jenkins Pipelines"]
        sonar["SonarQube"]
        trivy["Trivy Scanner"]
        k8s["Kubernetes API"]
        helm["Helm Releases"]
        argocd["Argo CD ApplicationSet"]
    end

    subgraph runtime["Running Platform"]
        apps["Application Workloads<br/>Frontend / Backend / User Apps"]
        dbs["Managed Databases<br/>PostgreSQL / MySQL / MongoDB<br/>Redis / Cassandra"]
        gateway["Ingress / TCP Gateway<br/>HTTP and Database Access"]
        storage["Longhorn Volumes"]
    end

    subgraph observe["Feedback Loop"]
        prom["Prometheus"]
        thanos["Thanos"]
        grafana["Grafana"]
        loki["Loki / Alloy"]
    end

    user --> pages
    pages --> session
    session --> keycloak
    session --> oauth
    pages --> routes
    routes --> backend
    backend --> security
    security --> domain
    domain --> ws
    ws --> pages

    domain --> workspaces
    domain --> projects
    domain --> dbdeploy
    domain --> backup
    domain --> console
    domain --> monitoring
    domain --> notifications
    domain --> payments

    workspaces --> postgres
    projects --> postgres
    dbdeploy --> postgres
    backup --> minio
    console --> dbs
    payments --> postgres

    projects --> git
    projects --> jenkins
    jenkins --> sonar
    jenkins --> trivy
    jenkins --> images
    images --> k8s

    dbdeploy --> git
    git --> argocd
    argocd --> helm
    helm --> k8s
    k8s --> apps
    k8s --> dbs
    dbs --> storage
    apps --> gateway
    dbs --> gateway

    apps --> prom
    dbs --> prom
    prom --> thanos
    thanos --> grafana
    apps --> loki
    dbs --> loki
    grafana --> routes
    loki --> routes
    monitoring --> prom
    notifications --> pages

    classDef ui fill:#eef6ff,stroke:#3b82f6,stroke-width:1.5px,color:#0f172a
    classDef backend fill:#ecfdf5,stroke:#22c55e,stroke-width:1.5px,color:#064e3b
    classDef domain fill:#f0fdf4,stroke:#16a34a,stroke-width:1.2px,color:#14532d
    classDef state fill:#fff1f2,stroke:#e11d48,stroke-width:1.5px,color:#881337
    classDef delivery fill:#fff7ed,stroke:#f97316,stroke-width:1.5px,color:#7c2d12
    classDef runtime fill:#eef2ff,stroke:#6366f1,stroke-width:1.5px,color:#1e1b4b
    classDef observe fill:#f5f3ff,stroke:#8b5cf6,stroke-width:1.5px,color:#3b0764

    class pages,routes,session,keycloak,oauth ui
    class backend,security,domain,ws backend
    class workspaces,projects,dbdeploy,backup,console,monitoring,notifications,payments domain
    class postgres,minio,git,images state
    class jenkins,sonar,trivy,k8s,helm,argocd delivery
    class apps,dbs,gateway,storage runtime
    class prom,thanos,grafana,loki observe
```

