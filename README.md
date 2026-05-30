<div align="center">

# 🌐 dhg-rateauto-tf-gke-routing

### Terraform · GKE Gateway API · HTTPS Routing · SSL Termination
### DHG Rate Automation Platform — `dhg-vaccine-rateauto-nonpord`

[![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.4-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io)
[![GKE Gateway API](https://img.shields.io/badge/GKE-Gateway_API-4285F4?logo=google-cloud&logoColor=white)](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api)
[![Google Provider](https://img.shields.io/badge/Google_Provider-~%3E5.0-34A853?logo=google&logoColor=white)](https://registry.terraform.io/providers/hashicorp/google/latest)
[![HTTPS](https://img.shields.io/badge/Protocol-HTTPS-00C853?logo=letsencrypt&logoColor=white)](https://cloud.google.com/load-balancing/docs/ssl-certificates)
[![WIF](https://img.shields.io/badge/Auth-Workload_Identity_Federation-FF6D00?logo=googlecloud)](https://cloud.google.com/iam/docs/workload-identity-federation)

---

*Provisions GKE Gateway API resources — Static IP, HTTPS Gateway, HTTPRoutes, and SSL certificates — to route external traffic to the DHG Vaccine Fee dashboard running on GKE Autopilot.*

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Why Gateway API Over Ingress](#-why-gateway-api-over-ingress)
- [Architecture](#-architecture)
- [Traffic Flow](#-traffic-flow)
- [Repository Structure](#-repository-structure)
- [Prerequisites](#-prerequisites)
- [Resources Created](#-resources-created)
- [Routing Design](#-routing-design)
- [Path-Based Routing](#-path-based-routing)
- [SSL & HTTPS](#-ssl--https)
- [Variables Reference](#-variables-reference)
- [Environments](#-environments)
- [Usage](#-usage)
- [CI/CD Pipeline](#-cicd-pipeline)
- [Security](#-security)
- [Live Endpoints](#-live-endpoints)
- [Provider Versions](#-provider-versions)
- [Related Repositories](#-related-repositories)

---

## 🌐 Overview

This repository is the **third and final infrastructure layer** of the DHG Rate Automation platform. It provisions the network routing layer that connects the public internet to the private GKE Autopilot workloads — using the modern **GKE Gateway API** instead of the legacy Kubernetes Ingress.

It configures:
- 🔒 **HTTPS-only** access with automatic SSL certificate management
- 🌍 A **static external IP** (`8.232.155.177`) bound to the domain `dev.gcpcloudhub.shop`
- 🔀 **Path-based routing** — different URL prefixes route to different backend services
- 🚦 **HTTP → HTTPS redirect** — all plain HTTP traffic is automatically upgraded

This repo is deployed **after** the GKE cluster is running (`dhg-rateauto-tf-gke`) and **before** the application CI/CD pipelines push container images.

### 🔑 Key Facts

| Property | Value |
|---|---|
| 🌍 Domain | `dev.gcpcloudhub.shop` |
| 🔒 Protocol | HTTPS (TLS 1.2+) |
| 📍 Static IP | `8.232.155.177` |
| 🗺️ Frontend path | `/vaccinefee-ui` |
| 🗺️ Backend path | `/vaccinefee/api` |
| 🌎 Region | Global (Google-managed load balancer) |
| 🔄 Routing method | GKE Gateway API (not Kubernetes Ingress) |

---

## ⚡ Why Gateway API Over Ingress

The GKE Gateway API is the **next-generation** replacement for Kubernetes Ingress. Here's why we chose it:

| Feature | Kubernetes Ingress | GKE Gateway API ✅ |
|---|---|---|
| **Standard** | Kubernetes | Kubernetes SIG-Network standard |
| **Routing rules** | Basic host/path | Rich: header, weight, method-based |
| **Role separation** | Single resource | Split: Gateway (infra) + HTTPRoute (app) |
| **HTTPS redirect** | Requires annotation hacks | Native support |
| **Traffic splitting** | Not supported | Native canary/A-B testing |
| **Load balancer type** | Google HTTP(S) LB | Global External HTTP(S) LB v2 |
| **Certificate management** | Manual or cert-manager | Google-managed certificates |
| **Future-proof** | Being deprecated | Active development standard |
| **Observability** | Limited | Full Cloud Monitoring integration |

For a production healthcare platform, Gateway API provides the expressiveness, security, and observability we need without workarounds.

---

## 🏛️ Architecture

```
                              Internet
                                 │
                                 │ HTTPS (443)
                                 ▼
              ┌──────────────────────────────────────┐
              │     Global External HTTPS Load        │
              │     Balancer (Google-managed)         │
              │                                       │
              │     Static IP: 8.232.155.177          │
              │     Domain:    dev.gcpcloudhub.shop   │
              │     SSL Cert:  Google-managed         │
              └──────────────────┬───────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      GKE Gateway         │
                    │  (gke-l7-global-external │
                    │   -managed-mc Class)     │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                                      │
              ▼                                      ▼
  ┌───────────────────────┐            ┌───────────────────────┐
  │      HTTPRoute         │            │      HTTPRoute         │
  │  path: /vaccinefee-ui  │            │  path: /vaccinefee/api │
  │  → Frontend Service   │            │  → Backend Service    │
  └───────────┬───────────┘            └───────────┬───────────┘
              │                                    │
              ▼                                    ▼
  ┌───────────────────────┐            ┌───────────────────────┐
  │  K8s Service           │            │  K8s Service           │
  │  dhg-vaccinefee-ui     │            │  dhg-vaccinefee-api   │
  │  ClusterIP :8080       │            │  ClusterIP :8080      │
  └───────────┬───────────┘            └───────────┬───────────┘
              │                                    │
              ▼                                    ▼
  ┌───────────────────────┐            ┌───────────────────────┐
  │  Pod: Frontend         │            │  Pod: Backend          │
  │  React + nginx         │            │  FastAPI (Python)     │
  │  :8080                 │            │  :8080                │
  └───────────────────────┘            └───────────────────────┘
              │                                    │
              └─────────────────┬──────────────────┘
                                ▼
                    ┌────────────────────┐
                    │  Cloud SQL (PSC)   │
                    │  PostgreSQL        │
                    │  10.10.0.3:5432   │
                    └────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │  HTTP → HTTPS Redirect (HTTPRoute)                   │
  │  All port 80 traffic → 301 redirect to HTTPS        │
  └─────────────────────────────────────────────────────┘
```

---

## 🔀 Traffic Flow

Here is the complete request lifecycle from a user's browser to the application:

```
Step 1:  User opens https://dev.gcpcloudhub.shop/vaccinefee-ui
         │
Step 2:  DNS resolves dev.gcpcloudhub.shop → 8.232.155.177 (Static IP)
         │
Step 3:  Google's Global Load Balancer receives the request
         Terminates TLS using Google-managed SSL certificate
         │
Step 4:  GKE Gateway evaluates the request
         Matches path prefix /vaccinefee-ui
         │
Step 5:  HTTPRoute forwards to Kubernetes Service dhg-vaccinefee-ui
         on port 8080 (ClusterIP — no public IP)
         │
Step 6:  Service load balances to one of the React+nginx Pods
         Pod serves the React SPA (Single Page Application)
         │
Step 7:  React app makes API calls to /vaccinefee/api/*
         These go back through the same Gateway → Backend HTTPRoute
         │
Step 8:  FastAPI backend handles the request
         Queries PostgreSQL via PSC (private IP 10.10.0.3)
         Returns JSON response
         │
Step 9:  Response travels back: Pod → Service → Gateway → LB → User

Total round-trip: ~50-200ms typical for India region
```

---

## 📁 Repository Structure

```
dhg-rateauto-tf-gke-routing/
│
├── 📁 .github/
│   └── 📁 workflows/
│       └── 📄 terraform.yml         # CI/CD: plan on PR, apply on merge
│
├── 📁 environments/
│   ├── 📄 dev.tfvars                # Dev routing config (domain, IP, paths)
│   ├── 📄 test.tfvars               # Test routing config
│   └── 📄 stage.tfvars              # Stage routing config
│
├── 📄 main.tf                       # Gateway, HTTPRoutes, Static IP resources
├── 📄 variables.tf                  # All input variable definitions
├── 📄 outputs.tf                    # Gateway IP, URLs as outputs
├── 📄 providers.tf                  # Google + Kubernetes provider config
├── 📄 terraform.tf                  # GCS backend configuration
├── 📄 versions.tf                   # Terraform + provider version constraints
└── 📄 README.md                     # This file
```

---

## ✅ Prerequisites

| Requirement | Details |
|---|---|
| 🏗️ **VPC deployed** | `dhg-rateauto-tf-vpc` must be applied first |
| ☸️ **GKE cluster running** | `dhg-rateauto-tf-gke` must be applied second |
| 🚀 **App deployed** | K8s Services must exist in namespace |
| 🔧 **Terraform** | `>= 1.4` |
| ☁️ **Google Provider** | `~> 5.0` |
| ☸️ **Kubernetes Provider** | `~> 2.0` |
| 🔌 **APIs enabled** | `container.googleapis.com`, `compute.googleapis.com`, `certificatemanager.googleapis.com` |
| 🔐 **IAM permissions** | `roles/compute.networkAdmin`, `roles/container.developer` |
| 🌐 **Domain DNS** | `dev.gcpcloudhub.shop` A record → `8.232.155.177` |

Enable required APIs:

```bash
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  certificatemanager.googleapis.com \
  --project=dhg-vaccine-rateauto-nonpord
```

---

## 📦 Resources Created

| # | Resource | Type | Description |
|---|---|---|---|
| 1 | `google_compute_global_address` | GCP | Static external IP (`8.232.155.177`) |
| 2 | `kubernetes_manifest` (Gateway) | K8s | GKE Gateway — global HTTPS load balancer |
| 3 | `kubernetes_manifest` (HTTPRoute UI) | K8s | Routes `/vaccinefee-ui` → Frontend Service |
| 4 | `kubernetes_manifest` (HTTPRoute API) | K8s | Routes `/vaccinefee/api` → Backend Service |
| 5 | `kubernetes_manifest` (HTTPRoute redirect) | K8s | HTTP → HTTPS 301 redirect |
| 6 | `google_compute_managed_ssl_certificate` | GCP | Google-managed SSL cert for the domain |

**Total: 6 resources** per environment.

---

## 🔀 Routing Design

### Gateway Class

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: dhg-vaccinefee-gateway
  namespace: dhg-rateauto-dev-namespace
spec:
  gatewayClassName: gke-l7-global-external-managed-mc
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: dhg-ssl-cert
    - name: http
      protocol: HTTP
      port: 80
```

**`gke-l7-global-external-managed-mc`** is the GKE-managed GatewayClass that provisions a **Global External HTTP(S) Load Balancer** — the same enterprise-grade load balancer used by Google's own products. It provides:
- ✅ Anycast IP routing (closest Google edge POP)
- ✅ DDoS protection at the network layer
- ✅ Automatic SSL certificate renewal
- ✅ HTTP/2 and QUIC support
- ✅ Cloud CDN integration (if enabled)

---

## 🗺️ Path-Based Routing

Two separate `HTTPRoute` resources handle the routing:

### Frontend Route — React Dashboard

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dhg-vaccinefee-ui-route
  namespace: dhg-rateauto-dev-namespace
spec:
  parentRefs:
    - name: dhg-vaccinefee-gateway
  hostnames:
    - "dev.gcpcloudhub.shop"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /vaccinefee-ui
      backendRefs:
        - name: dhg-vaccinefee-ui
          port: 8080
```

### Backend Route — FastAPI

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dhg-vaccinefee-api-route
  namespace: dhg-rateauto-dev-namespace
spec:
  parentRefs:
    - name: dhg-vaccinefee-gateway
  hostnames:
    - "dev.gcpcloudhub.shop"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /vaccinefee/api
      backendRefs:
        - name: dhg-vaccinefee-api
          port: 8080
```

### HTTP → HTTPS Redirect

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dhg-http-redirect
  namespace: dhg-rateauto-dev-namespace
spec:
  parentRefs:
    - name: dhg-vaccinefee-gateway
      sectionName: http
  rules:
    - filters:
        - type: RequestRedirect
          requestRedirect:
            scheme: https
            statusCode: 301
```

### URL Routing Summary

| Incoming Request | Route | Backend | Service Port |
|---|---|---|---|
| `https://dev.gcpcloudhub.shop/vaccinefee-ui/*` | UI HTTPRoute | `dhg-vaccinefee-ui` | 8080 |
| `https://dev.gcpcloudhub.shop/vaccinefee/api/*` | API HTTPRoute | `dhg-vaccinefee-api` | 8080 |
| `https://dev.gcpcloudhub.shop/vaccinefee/api/docs` | API HTTPRoute | Swagger UI | 8080 |
| `http://dev.gcpcloudhub.shop/*` | Redirect HTTPRoute | → 301 HTTPS | — |

---

## 🔒 SSL & HTTPS

### Google-Managed SSL Certificate

```hcl
resource "google_compute_managed_ssl_certificate" "dhg_ssl" {
  name = "dhg-vaccinefee-ssl-cert"

  managed {
    domains = ["dev.gcpcloudhub.shop"]
  }
}
```

Google-managed certificates are **fully automatic**:
- ✅ **Provisioned automatically** — no manual CSR generation
- ✅ **Renewed automatically** — 30 days before expiry
- ✅ **Free** — no certificate authority costs
- ✅ **Domain validated** — Google verifies DNS ownership
- ✅ **TLS 1.2+** — older TLS versions blocked automatically

### Certificate Provisioning Timeline

After `terraform apply`, certificate provisioning typically takes:

```
0 min   → Certificate resource created in GCP
2 min   → Google begins DNS validation
5-15 min → Certificate provisioned and active
15 min  → HTTPS traffic starts flowing
```

> ⚠️ The DNS A record (`dev.gcpcloudhub.shop → 8.232.155.177`) must exist **before** applying this Terraform. Google cannot validate the domain without it.

### Static IP Reservation

```hcl
resource "google_compute_global_address" "dhg_static_ip" {
  name         = "dhg-vaccinefee-static-ip"
  address_type = "EXTERNAL"
  ip_version   = "IPV4"
}
```

Using a **reserved static IP** rather than ephemeral:
- ✅ IP never changes across Gateway recreations
- ✅ DNS record stays valid permanently
- ✅ SSL certificate linked to a stable endpoint
- ✅ Can be referenced in allowlists and firewall rules

---

## 📊 Variables Reference

### 🏗️ Core

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `project_id` | `string` | — | ✅ | GCP project ID |
| `region` | `string` | `us-central1` | No | GCP region |
| `stage` | `string` | — | ✅ | Environment: `dev`, `test`, `stage`, `prod` |
| `cluster_name` | `string` | — | ✅ | GKE cluster name to deploy routes into |
| `namespace` | `string` | — | ✅ | Kubernetes namespace for all resources |

### 🌐 Networking

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `domain` | `string` | — | ✅ | Domain name for HTTPS (e.g. `dev.gcpcloudhub.shop`) |
| `static_ip_name` | `string` | — | ✅ | Name for the reserved global static IP |
| `gateway_name` | `string` | — | ✅ | Name for the GKE Gateway resource |

### 🔀 Routing

| Variable | Type | Default | Required | Description |
|---|---|---|---|---|
| `frontend_service_name` | `string` | — | ✅ | K8s Service name for the React frontend |
| `frontend_service_port` | `number` | `8080` | No | Frontend service port |
| `frontend_path_prefix` | `string` | `/vaccinefee-ui` | No | URL path prefix for frontend routes |
| `backend_service_name` | `string` | — | ✅ | K8s Service name for the FastAPI backend |
| `backend_service_port` | `number` | `8080` | No | Backend service port |
| `backend_path_prefix` | `string` | `/vaccinefee/api` | No | URL path prefix for API routes |

### 🏷️ Labels

| Variable | Type | Default | Description |
|---|---|---|---|
| `resource_labels` | `map(string)` | `{}` | GCP labels applied to all resources |

---

## 🌍 Environments

### Example `environments/dev.tfvars`

```hcl
# ── Core ──────────────────────────────────────────────────
project_id   = "dhg-vaccine-rateauto-nonpord"
region       = "us-central1"
stage        = "dev"
cluster_name = "dhg-rateauto-dev-gke-cluster"
namespace    = "dhg-rateauto-dev-namespace"

# ── Domain & IP ───────────────────────────────────────────
domain         = "dev.gcpcloudhub.shop"
static_ip_name = "dhg-vaccinefee-dev-static-ip"
gateway_name   = "dhg-vaccinefee-dev-gateway"

# ── Routing ───────────────────────────────────────────────
frontend_service_name = "dhg-vaccinefee-ui"
frontend_service_port = 8080
frontend_path_prefix  = "/vaccinefee-ui"

backend_service_name  = "dhg-vaccinefee-api"
backend_service_port  = 8080
backend_path_prefix   = "/vaccinefee/api"

# ── Labels ────────────────────────────────────────────────
resource_labels = {
  environment = "dev"
  team        = "platform"
  managed-by  = "terraform"
}
```

### Environment URL Mapping

| Environment | Domain | Static IP | Frontend URL | API URL |
|---|---|---|---|---|
| **Dev** | `dev.gcpcloudhub.shop` | `8.232.155.177` | `/vaccinefee-ui` | `/vaccinefee/api` |
| **Test** | `test.gcpcloudhub.shop` | TBD | `/vaccinefee-ui` | `/vaccinefee/api` |
| **Stage** | `stage.gcpcloudhub.shop` | TBD | `/vaccinefee-ui` | `/vaccinefee/api` |

---

## 🚀 Usage

### 🖥️ Local Deployment

```bash
# 1. Clone the repo
git clone https://github.com/bikram-singh/dhg-rateauto-tf-gke-routing.git
cd dhg-rateauto-tf-gke-routing

# 2. Get GKE credentials (required for Kubernetes provider)
gcloud container clusters get-credentials dhg-rateauto-dev-gke-cluster \
  --region us-central1 \
  --project dhg-vaccine-rateauto-nonpord

# 3. Authenticate with GCP
gcloud auth application-default login

# 4. Initialise Terraform
terraform init

# 5. Plan
terraform plan -var-file=environments/dev.tfvars -out=tfplan

# 6. Apply
terraform apply -auto-approve -input=false tfplan
```

### ✅ Verify Routing is Working

```bash
# Check Gateway status
kubectl get gateway dhg-vaccinefee-dev-gateway \
  -n dhg-rateauto-dev-namespace

# Check HTTPRoutes
kubectl get httproute -n dhg-rateauto-dev-namespace

# Check static IP was assigned
kubectl get gateway dhg-vaccinefee-dev-gateway \
  -n dhg-rateauto-dev-namespace \
  -o jsonpath='{.status.addresses[0].value}'

# Test HTTPS endpoint
curl -I https://dev.gcpcloudhub.shop/vaccinefee-ui

# Test API endpoint
curl https://dev.gcpcloudhub.shop/vaccinefee/api/health

# Test HTTP redirect
curl -I http://dev.gcpcloudhub.shop/vaccinefee-ui
# Should return: HTTP/1.1 301 Moved Permanently
# Location: https://dev.gcpcloudhub.shop/vaccinefee-ui
```

### 🗑️ Destroy

```bash
# Destroy routing only (does not affect GKE cluster or VPC)
terraform destroy -var-file=environments/dev.tfvars
```

> **Safe to destroy:** Removing the Gateway only removes the load balancer and routing rules. The GKE cluster, pods, and data are unaffected.

---

## ⚙️ CI/CD Pipeline

### 🔄 Pipeline Flow

```
┌────────────────────────────────────────────────────────┐
│               On Pull Request → main                    │
│                                                         │
│   terraform fmt    →   terraform init   →   terraform  │
│   -check               -backend-config     validate    │
│                                             +           │
│                                         terraform plan  │
│                                         (PR comment)   │
└────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────┐
│               On Push to main                           │
│                                                         │
│   terraform init  →  terraform plan  →  terraform      │
│                       -out=tfplan       apply          │
│                                         -auto-approve  │
└────────────────────────────────────────────────────────┘
```

### 🔐 WIF Authentication

```yaml
# .github/workflows/terraform.yml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: >-
      projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/
      dhg-rateauto-wif-pool/providers/github-oidc
    service_account: >-
      dhg-routing-tf-sa@dhg-vaccine-rateauto-nonpord.iam.gserviceaccount.com

# Also configure kubectl access
- name: Get GKE credentials
  uses: google-github-actions/get-gke-credentials@v2
  with:
    cluster_name: dhg-rateauto-dev-gke-cluster
    location: us-central1
    project_id: dhg-vaccine-rateauto-nonpord
```

### 🔑 Required IAM Roles

| Role | Purpose |
|---|---|
| `roles/compute.networkAdmin` | Reserve and manage static IP |
| `roles/compute.loadBalancerAdmin` | Manage load balancer resources |
| `roles/container.developer` | Deploy K8s resources (Gateway, HTTPRoutes) |
| `roles/certificatemanager.editor` | Manage SSL certificates |
| `roles/storage.objectAdmin` | Read/write Terraform state in GCS |

---

## 🔒 Security

### 🛡️ Security Features

| Feature | Implementation |
|---|---|
| **HTTPS enforced** | HTTP automatically redirects to HTTPS (301) |
| **TLS 1.2+** | Older TLS versions rejected by Google LB |
| **Google-managed cert** | Auto-renewed, domain-validated |
| **No public node IPs** | All routing goes through the Gateway — nodes are private |
| **Static IP control** | Reserved IP prevents hijacking |
| **WIF only** | No JSON keys stored — CI/CD uses short-lived OIDC tokens |
| **Namespace isolation** | All K8s resources in `dhg-rateauto-dev-namespace` |

### 🔐 SSL/TLS Configuration

The Google Global External Load Balancer enforces:
- ✅ TLS 1.2 minimum (TLS 1.0, 1.1 rejected)
- ✅ Modern cipher suites only
- ✅ HSTS support (HTTP Strict Transport Security)
- ✅ Automatic certificate renewal (30 days before expiry)
- ✅ SNI (Server Name Indication) support

### 🌐 Network Path

```
User → Google Edge (anycast) → TLS termination → Gateway
                                                      ↓
                                              Private GKE VPC
                                                      ↓
                                              Pod (no public IP)
```

No traffic ever reaches the GKE nodes directly from the internet — everything is channeled through the managed load balancer.

---

## 🖥️ Live Endpoints

| Endpoint | Description | URL |
|---|---|---|
| 🏠 **Frontend Dashboard** | React vaccine pricing UI | `https://dev.gcpcloudhub.shop/vaccinefee-ui` |
| ⚡ **Backend API** | FastAPI REST endpoints | `https://dev.gcpcloudhub.shop/vaccinefee/api` |
| 📖 **Swagger UI** | Interactive API documentation | `https://dev.gcpcloudhub.shop/vaccinefee/api/docs` |
| 🔁 **HTTP Redirect** | Auto-upgrades to HTTPS | `http://dev.gcpcloudhub.shop/*` → `https://` |

---

## 📌 Provider Versions

```hcl
# versions.tf
terraform {
  required_version = ">= 1.4"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

# providers.tf
provider "google" {
  project = var.project_id
  region  = var.region
}

provider "kubernetes" {
  host  = "https://${data.google_container_cluster.gke.endpoint}"
  token = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(
    data.google_container_cluster.gke.master_auth[0].cluster_ca_certificate
  )
}
```

The **Kubernetes provider** authenticates using the GKE cluster endpoint and Google access token obtained via WIF — no kubeconfig file or service account keys needed.

---

## 🔗 Related Repositories

| Repository | Purpose | Deploy Order |
|---|---|---|
| [`dhg-rateauto-tf-vpc`](https://github.com/bikram-singh/dhg-rateauto-tf-vpc) | VPC, Subnet, NAT, Firewall | 1️⃣ First |
| [`dhg-rateauto-tf-gke`](https://github.com/bikram-singh/dhg-rateauto-tf-gke) | GKE Autopilot Cluster | 2️⃣ Second |
| [`dhg-rateauto-tf-gke-routing`](https://github.com/bikram-singh/dhg-rateauto-tf-gke-routing) | **This repo** — Gateway API, HTTPS, Routing | 3️⃣ Third |
| [`dhg-rateauto-tf-gcs-buckets`](https://github.com/bikram-singh/dhg-rateauto-tf-gcs-buckets) | GCS Bucket Provisioning | 4️⃣ Independent |
| [`dhg-rateauto-api-backend`](https://github.com/bikram-singh/dhg-rateauto-api-backend) | FastAPI Backend | 5️⃣ App layer |
| [`dhg-rateauto-ui-frontend`](https://github.com/bikram-singh/dhg-rateauto-ui-frontend) | React Frontend Dashboard | 5️⃣ App layer |

---

<div align="center">

**Maintained by Bikram Singh**
`dhg-vaccine-rateauto-nonpord` · `us-central1` · GKE Gateway API

</div>
