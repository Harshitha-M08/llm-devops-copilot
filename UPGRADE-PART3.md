# AI-Driven Hybrid Kubernetes System - Google Cloud Deployment Guide (Part 3)

## 📋 Overview

This guide provides **step-by-step instructions** for deploying the entire AI-Driven Hybrid Kubernetes System on **Google Cloud Platform (GCP)**. By the end of this guide, you will have a fully functional, production-ready system running on GCP with:

- **GKE (Google Kubernetes Engine)** cluster for container orchestration
- **Cloud SQL (PostgreSQL)** for relational database
- **Memorystore (Redis)** for caching
- **Cloud Storage** for backups and static assets
- **Cloud Load Balancer** for traffic distribution
- **Cloud Monitoring** for observability
- **Cloud Build** for CI/CD
- **Artifact Registry** for container images

---

## 🎯 Prerequisites

### Required Tools & Accounts

1. **Google Cloud Account**
   - [ ] Create account at https://cloud.google.com/
   - [ ] Enable billing (you'll get $300 free credits for 90 days)
   - [ ] Create a new project (e.g., `ai-devops-system`)

2. **Install Required Tools on Your Local Machine**

   **Google Cloud SDK (gcloud CLI)**
   ```bash
   # Windows (PowerShell)
   (New-Object Net.WebClient).DownloadFile("https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe", "$env:Temp\GoogleCloudSDKInstaller.exe")
   & $env:Temp\GoogleCloudSDKInstaller.exe

   # Verify installation
   gcloud version
   ```

   **kubectl** (Kubernetes CLI)
   ```bash
   gcloud components install kubectl
   ```

   **Helm** (Kubernetes package manager)
   ```bash
   # Windows (using Chocolatey)
   choco install kubernetes-helm

   # Or download from https://github.com/helm/helm/releases
   # Verify
   helm version
   ```

   **Docker Desktop**
   - Download from https://www.docker.com/products/docker-desktop
   - Install and start Docker Desktop

   **Terraform** (Infrastructure as Code)
   ```bash
   # Windows (using Chocolatey)
   choco install terraform

   # Verify
   terraform version
   ```

   **Git Bash** (for running shell scripts on Windows)
   - Download from https://git-scm.com/download/win

3. **API Keys**
   - [ ] OpenAI API key: https://platform.openai.com/api-keys
   - [ ] Anthropic API key: https://console.anthropic.com/
   - [ ] SMTP credentials for email (Gmail, SendGrid, etc.)

---

## 📁 PHASE 1: GCP PROJECT SETUP

### 1.1 Initialize GCP Project

```bash
# Login to Google Cloud
gcloud auth login

# Set your project ID (replace with your project ID)
export PROJECT_ID="ai-devops-system"
gcloud config set project $PROJECT_ID

# Enable required APIs
gcloud services enable \
  container.googleapis.com \
  compute.googleapis.com \
  sql-component.googleapis.com \
  sqladmin.googleapis.com \
  redis.googleapis.com \
  storage-api.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  cloudresourcemanager.googleapis.com \
  servicenetworking.googleapis.com

# Set default region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Verify configuration
gcloud config list
```

### 1.2 Create Service Account for Terraform

```bash
# Create service account
gcloud iam service-accounts create terraform-sa \
  --display-name="Terraform Service Account"

# Grant necessary roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:terraform-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/editor"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:terraform-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/container.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:terraform-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Create and download key
gcloud iam service-accounts keys create ~/terraform-key.json \
  --iam-account=terraform-sa@${PROJECT_ID}.iam.gserviceaccount.com

# Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS=~/terraform-key.json
```

### 1.3 Create Artifact Registry for Docker Images

```bash
# Create Docker repository
gcloud artifacts repositories create ai-system-images \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images for AI System"

# Configure Docker to use gcloud as credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# Verify
gcloud artifacts repositories list
```

---

## 📁 PHASE 2: TERRAFORM INFRASTRUCTURE SETUP

### 2.1 Create Terraform Configuration for GCP

#### File: `infrastructure/terraform/gcp/provider.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }

  # Optional: Store state in Cloud Storage
  backend "gcs" {
    bucket = "ai-system-terraform-state"  # Create this bucket first
    prefix = "terraform/state"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

data "google_client_config" "default" {}

provider "kubernetes" {
  host                   = "https://${google_container_cluster.primary.endpoint}"
  token                  = data.google_client_config.default.access_token
  cluster_ca_certificate = base64decode(google_container_cluster.primary.master_auth[0].cluster_ca_certificate)
}
```

#### File: `infrastructure/terraform/gcp/variables.tf`

```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
  default     = "ai-devops-system"
}

variable "region" {
  description = "GCP Region"
  type        = string
  default     = "us-central1"
}

variable "zone" {
  description = "GCP Zone"
  type        = string
  default     = "us-central1-a"
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
  default     = "prod"
}

# GKE Cluster Configuration
variable "gke_cluster_name" {
  description = "GKE Cluster Name"
  type        = string
  default     = "ai-system-cluster"
}

variable "gke_node_count" {
  description = "Number of nodes per zone"
  type        = number
  default     = 3
}

variable "gke_machine_type" {
  description = "Machine type for GKE nodes"
  type        = string
  default     = "e2-standard-4"  # 4 vCPUs, 16 GB RAM
}

variable "gke_disk_size_gb" {
  description = "Disk size for GKE nodes (GB)"
  type        = number
  default     = 100
}

# Database Configuration
variable "db_instance_name" {
  description = "Cloud SQL Instance Name"
  type        = string
  default     = "ai-system-postgres"
}

variable "db_tier" {
  description = "Cloud SQL instance tier"
  type        = string
  default     = "db-custom-2-7680"  # 2 vCPUs, 7.5 GB RAM
}

variable "db_name" {
  description = "Database name"
  type        = string
  default     = "devops_db"
}

variable "db_user" {
  description = "Database user"
  type        = string
  default     = "devops"
}

# Redis Configuration
variable "redis_instance_name" {
  description = "Memorystore Redis instance name"
  type        = string
  default     = "ai-system-redis"
}

variable "redis_memory_size_gb" {
  description = "Redis memory size (GB)"
  type        = number
  default     = 5
}

variable "redis_tier" {
  description = "Redis tier (BASIC or STANDARD_HA)"
  type        = string
  default     = "STANDARD_HA"  # High availability
}
```

#### File: `infrastructure/terraform/gcp/vpc.tf`

```hcl
# VPC Network
resource "google_compute_network" "vpc" {
  name                    = "${var.gke_cluster_name}-vpc"
  auto_create_subnetworks = false
  description             = "VPC for AI System"
}

# Subnet for GKE
resource "google_compute_subnetwork" "gke_subnet" {
  name          = "${var.gke_cluster_name}-subnet"
  ip_cidr_range = "10.0.0.0/20"
  region        = var.region
  network       = google_compute_network.vpc.id

  secondary_ip_range {
    range_name    = "gke-pods"
    ip_cidr_range = "10.4.0.0/14"
  }

  secondary_ip_range {
    range_name    = "gke-services"
    ip_cidr_range = "10.8.0.0/20"
  }

  private_ip_google_access = true
}

# Cloud NAT for outbound internet access
resource "google_compute_router" "router" {
  name    = "${var.gke_cluster_name}-router"
  region  = var.region
  network = google_compute_network.vpc.id
}

resource "google_compute_router_nat" "nat" {
  name                               = "${var.gke_cluster_name}-nat"
  router                             = google_compute_router.router.name
  region                             = var.region
  nat_ip_allocate_option             = "AUTO_ONLY"
  source_subnetwork_ip_ranges_to_nat = "ALL_SUBNETWORKS_ALL_IP_RANGES"
}

# Firewall rules
resource "google_compute_firewall" "allow_internal" {
  name    = "${var.gke_cluster_name}-allow-internal"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }

  allow {
    protocol = "icmp"
  }

  source_ranges = ["10.0.0.0/8"]
}

# Private Service Connection for Cloud SQL
resource "google_compute_global_address" "private_ip_address" {
  name          = "${var.gke_cluster_name}-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_address.name]
}
```

#### File: `infrastructure/terraform/gcp/gke.tf`

```hcl
# GKE Cluster
resource "google_container_cluster" "primary" {
  name     = var.gke_cluster_name
  location = var.region

  # We can't create a cluster with no node pool defined, but we want to only use
  # separately managed node pools. So we create the smallest possible default
  # node pool and immediately delete it.
  remove_default_node_pool = true
  initial_node_count       = 1

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.gke_subnet.name

  # IP allocation for pods and services
  ip_allocation_policy {
    cluster_secondary_range_name  = "gke-pods"
    services_secondary_range_name = "gke-services"
  }

  # Master authorized networks (restrict access to your IP)
  master_authorized_networks_config {
    cidr_blocks {
      cidr_block   = "0.0.0.0/0"  # Change to your IP for security
      display_name = "All"
    }
  }

  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }

  # Addons
  addons_config {
    http_load_balancing {
      disabled = false
    }
    horizontal_pod_autoscaling {
      disabled = false
    }
    network_policy_config {
      disabled = false
    }
  }

  network_policy {
    enabled = true
  }

  # Maintenance window
  maintenance_policy {
    daily_maintenance_window {
      start_time = "03:00"  # 3 AM UTC
    }
  }

  # Logging and monitoring
  logging_service    = "logging.googleapis.com/kubernetes"
  monitoring_service = "monitoring.googleapis.com/kubernetes"
}

# Managed Node Pool
resource "google_container_node_pool" "primary_nodes" {
  name       = "${var.gke_cluster_name}-node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = var.gke_node_count

  # Auto-scaling
  autoscaling {
    min_node_count = 2
    max_node_count = 10
  }

  # Management
  management {
    auto_repair  = true
    auto_upgrade = true
  }

  node_config {
    preemptible  = false
    machine_type = var.gke_machine_type
    disk_size_gb = var.gke_disk_size_gb
    disk_type    = "pd-standard"

    # Scopes
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]

    # Labels
    labels = {
      environment = var.environment
      cluster     = var.gke_cluster_name
    }

    # Metadata
    metadata = {
      disable-legacy-endpoints = "true"
    }

    # Workload Identity
    workload_metadata_config {
      mode = "GKE_METADATA"
    }

    # Shielded instance config
    shielded_instance_config {
      enable_secure_boot          = true
      enable_integrity_monitoring = true
    }
  }
}
```

#### File: `infrastructure/terraform/gcp/cloud_sql.tf`

```hcl
# Random password for database
resource "random_password" "db_password" {
  length  = 32
  special = true
}

# Cloud SQL PostgreSQL Instance
resource "google_sql_database_instance" "postgres" {
  name             = var.db_instance_name
  database_version = "POSTGRES_15"
  region           = var.region

  depends_on = [google_service_networking_connection.private_vpc_connection]

  settings {
    tier              = var.db_tier
    availability_type = "REGIONAL"  # High availability
    disk_size         = 100          # GB
    disk_type         = "PD_SSD"
    disk_autoresize   = true

    # Private IP
    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }

    # Backup configuration
    backup_configuration {
      enabled                        = true
      start_time                     = "02:00"
      point_in_time_recovery_enabled = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
      }
    }

    # Maintenance window
    maintenance_window {
      day          = 7  # Sunday
      hour         = 3
      update_track = "stable"
    }

    # Database flags
    database_flags {
      name  = "max_connections"
      value = "200"
    }

    database_flags {
      name  = "shared_buffers"
      value = "262144"  # 2GB (in 8KB pages)
    }
  }

  deletion_protection = true  # Prevent accidental deletion
}

# Database
resource "google_sql_database" "database" {
  name     = var.db_name
  instance = google_sql_database_instance.postgres.name
}

# Database user
resource "google_sql_user" "user" {
  name     = var.db_user
  instance = google_sql_database_instance.postgres.name
  password = random_password.db_password.result
}
```

#### File: `infrastructure/terraform/gcp/redis.tf`

```hcl
# Memorystore Redis Instance
resource "google_redis_instance" "redis" {
  name               = var.redis_instance_name
  tier               = var.redis_tier
  memory_size_gb     = var.redis_memory_size_gb
  region             = var.region
  authorized_network = google_compute_network.vpc.id

  redis_version = "REDIS_7_0"

  display_name = "AI System Redis Cache"

  # High availability (for STANDARD_HA tier)
  replica_count        = var.redis_tier == "STANDARD_HA" ? 1 : 0
  read_replicas_mode   = var.redis_tier == "STANDARD_HA" ? "READ_REPLICAS_ENABLED" : null

  # Persistence configuration
  persistence_config {
    persistence_mode    = "RDB"
    rdb_snapshot_period = "ONE_HOUR"
  }

  # Maintenance policy
  maintenance_policy {
    weekly_maintenance_window {
      day = "SUNDAY"
      start_time {
        hours   = 3
        minutes = 0
      }
    }
  }
}
```

#### File: `infrastructure/terraform/gcp/storage.tf`

```hcl
# Cloud Storage bucket for backups
resource "google_storage_bucket" "backups" {
  name          = "${var.project_id}-backups"
  location      = var.region
  force_destroy = false

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age = 90  # Delete after 90 days
    }
    action {
      type = "Delete"
    }
  }

  lifecycle_rule {
    condition {
      age = 30
    }
    action {
      type          = "SetStorageClass"
      storage_class = "NEARLINE"
    }
  }
}

# Cloud Storage bucket for Terraform state
resource "google_storage_bucket" "terraform_state" {
  name          = "${var.project_id}-terraform-state"
  location      = var.region
  force_destroy = false

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      num_newer_versions = 5
    }
    action {
      type = "Delete"
    }
  }
}
```

#### File: `infrastructure/terraform/gcp/outputs.tf`

```hcl
output "gke_cluster_name" {
  description = "GKE Cluster Name"
  value       = google_container_cluster.primary.name
}

output "gke_cluster_endpoint" {
  description = "GKE Cluster Endpoint"
  value       = google_container_cluster.primary.endpoint
  sensitive   = true
}

output "gke_cluster_ca_certificate" {
  description = "GKE Cluster CA Certificate"
  value       = google_container_cluster.primary.master_auth[0].cluster_ca_certificate
  sensitive   = true
}

output "database_connection_name" {
  description = "Cloud SQL Connection Name"
  value       = google_sql_database_instance.postgres.connection_name
}

output "database_private_ip" {
  description = "Cloud SQL Private IP"
  value       = google_sql_database_instance.postgres.private_ip_address
}

output "database_password" {
  description = "Database Password"
  value       = random_password.db_password.result
  sensitive   = true
}

output "redis_host" {
  description = "Redis Host"
  value       = google_redis_instance.redis.host
}

output "redis_port" {
  description = "Redis Port"
  value       = google_redis_instance.redis.port
}

output "backups_bucket" {
  description = "Backups Storage Bucket"
  value       = google_storage_bucket.backups.url
}

output "vpc_network" {
  description = "VPC Network Name"
  value       = google_compute_network.vpc.name
}
```

### 2.2 Deploy Infrastructure with Terraform

```bash
# Navigate to Terraform directory
cd infrastructure/terraform/gcp

# Initialize Terraform
terraform init

# Create execution plan
terraform plan -out=tfplan

# Review the plan carefully
# Apply the configuration
terraform apply tfplan

# This will take 10-15 minutes
# Note down the outputs (database password, IPs, etc.)

# Save outputs to file
terraform output -json > outputs.json
```

### 2.3 Configure kubectl to Access GKE

```bash
# Get credentials for GKE cluster
gcloud container clusters get-credentials ai-system-cluster \
  --region=us-central1 \
  --project=$PROJECT_ID

# Verify connection
kubectl cluster-info
kubectl get nodes

# You should see 3-6 nodes listed
```

---

## 📁 PHASE 3: BUILD AND PUSH DOCKER IMAGES

### 3.1 Set Environment Variables

```bash
# Set variables
export PROJECT_ID="ai-devops-system"
export REGION="us-central1"
export REGISTRY="${REGION}-docker.pkg.dev/${PROJECT_ID}/ai-system-images"
export IMAGE_TAG="v1.0.0"

# Or use git commit hash
# export IMAGE_TAG=$(git rev-parse --short HEAD)
```

### 3.2 Build and Push LLM Service

```bash
cd services/llm-service

# Build Docker image
docker build -t ${REGISTRY}/llm-service:${IMAGE_TAG} .
docker build -t ${REGISTRY}/llm-service:latest .

# Push to Artifact Registry
docker push ${REGISTRY}/llm-service:${IMAGE_TAG}
docker push ${REGISTRY}/llm-service:latest

cd ../..
```

### 3.3 Build and Push Worker Service

```bash
cd services/worker-service

docker build -t ${REGISTRY}/worker-service:${IMAGE_TAG} .
docker build -t ${REGISTRY}/worker-service:latest .

docker push ${REGISTRY}/worker-service:${IMAGE_TAG}
docker push ${REGISTRY}/worker-service:latest

cd ../..
```

### 3.4 Build and Push Approval Backend

```bash
cd services/approval-dashboard/backend

docker build -t ${REGISTRY}/approval-backend:${IMAGE_TAG} .
docker build -t ${REGISTRY}/approval-backend:latest .

docker push ${REGISTRY}/approval-backend:${IMAGE_TAG}
docker push ${REGISTRY}/approval-backend:latest

cd ../../..
```

### 3.5 Build and Push Approval Frontend

```bash
cd services/approval-dashboard/frontend

docker build -t ${REGISTRY}/approval-frontend:${IMAGE_TAG} .
docker build -t ${REGISTRY}/approval-frontend:latest .

docker push ${REGISTRY}/approval-frontend:${IMAGE_TAG}
docker push ${REGISTRY}/approval-frontend:latest

cd ../../..
```

### 3.6 Verify Images in Artifact Registry

```bash
# List all images
gcloud artifacts docker images list ${REGION}-docker.pkg.dev/${PROJECT_ID}/ai-system-images

# You should see:
# - llm-service:v1.0.0, llm-service:latest
# - worker-service:v1.0.0, worker-service:latest
# - approval-backend:v1.0.0, approval-backend:latest
# - approval-frontend:v1.0.0, approval-frontend:latest
```

---

## 📁 PHASE 4: CONFIGURE KUBERNETES RESOURCES

### 4.1 Create Kubernetes Namespace

```bash
# Create namespace
kubectl create namespace ai-system

# Set as default namespace for convenience
kubectl config set-context --current --namespace=ai-system
```

### 4.2 Create Kubernetes Secrets

**IMPORTANT: Replace placeholder values with your actual credentials!**

```bash
# Get database password from Terraform output
DB_PASSWORD=$(terraform output -raw database_password -state=infrastructure/terraform/gcp/terraform.tfstate)

# Generate secure secrets
JWT_SECRET=$(openssl rand -base64 32)
JWT_REFRESH_SECRET=$(openssl rand -base64 32)
SESSION_SECRET=$(openssl rand -base64 32)
ENCRYPTION_KEY=$(openssl rand -base64 32)

# Create secrets in Kubernetes
kubectl create secret generic ai-secrets \
  --from-literal=openai-api-key='YOUR_OPENAI_API_KEY_HERE' \
  --from-literal=anthropic-api-key='YOUR_ANTHROPIC_API_KEY_HERE' \
  --from-literal=postgres-password="${DB_PASSWORD}" \
  --from-literal=redis-password='' \
  --from-literal=rabbitmq-password='devops123' \
  --from-literal=jwt-secret="${JWT_SECRET}" \
  --from-literal=jwt-refresh-secret="${JWT_REFRESH_SECRET}" \
  --from-literal=session-secret="${SESSION_SECRET}" \
  --from-literal=encryption-key="${ENCRYPTION_KEY}" \
  --from-literal=smtp-user='YOUR_SMTP_USER' \
  --from-literal=smtp-password='YOUR_SMTP_PASSWORD' \
  -n ai-system

# Verify secret created
kubectl get secrets -n ai-system
```

### 4.3 Update Kubernetes Manifests with GCP-specific Configurations

#### Update: `k8s/base/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-system-config
  namespace: ai-system
data:
  # Environment
  ENVIRONMENT: "production"
  NODE_ENV: "production"

  # Database (Cloud SQL via Private IP)
  POSTGRES_HOST: "10.x.x.x"  # Replace with actual private IP from Terraform output
  POSTGRES_PORT: "5432"
  POSTGRES_DB: "devops_db"
  POSTGRES_USER: "devops"

  # Redis (Memorystore via Private IP)
  REDIS_HOST: "10.x.x.x"  # Replace with actual Redis host from Terraform output
  REDIS_PORT: "6379"

  # RabbitMQ (will be deployed in K8s)
  RABBITMQ_HOST: "rabbitmq.ai-system.svc.cluster.local"
  RABBITMQ_PORT: "5672"
  RABBITMQ_USER: "devops"
  RABBITMQ_VHOST: "devops"

  # Qdrant (will be deployed in K8s)
  QDRANT_HOST: "qdrant.ai-system.svc.cluster.local"
  QDRANT_HTTP_PORT: "6333"

  # LLM Service
  LLM_SERVICE_URL: "http://llm-service.ai-system.svc.cluster.local:8000"

  # Frontend URL (will be updated after ingress creation)
  CORS_ORIGIN: "https://your-domain.com"
  NEXT_PUBLIC_API_URL: "https://your-domain.com/api"
```

**To get the actual IPs:**

```bash
# Get database private IP
terraform output -raw database_private_ip -state=infrastructure/terraform/gcp/terraform.tfstate

# Get Redis host
terraform output -raw redis_host -state=infrastructure/terraform/gcp/terraform.tfstate

# Update the configmap with these values
```

### 4.4 Update Deployment Image References

Update all deployment YAML files to use your Artifact Registry images:

```bash
# File: k8s/base/llm-service-deployment.yaml
# Replace:
#   image: llm-service:latest
# With:
#   image: us-central1-docker.pkg.dev/ai-devops-system/ai-system-images/llm-service:v1.0.0

# Do this for all deployments:
# - llm-service-deployment.yaml
# - worker-service-deployment.yaml
# - approval-backend-deployment.yaml
# - approval-frontend-deployment.yaml
```

Or use this script:

```bash
export REGISTRY="us-central1-docker.pkg.dev/ai-devops-system/ai-system-images"
export TAG="v1.0.0"

# Update all deployment files
sed -i "s|image: llm-service:latest|image: ${REGISTRY}/llm-service:${TAG}|g" k8s/base/llm-service-deployment.yaml
sed -i "s|image: worker-service:latest|image: ${REGISTRY}/worker-service:${TAG}|g" k8s/base/worker-service-deployment.yaml
sed -i "s|image: approval-backend:latest|image: ${REGISTRY}/approval-backend:${TAG}|g" k8s/base/approval-backend-deployment.yaml
sed -i "s|image: approval-frontend:latest|image: ${REGISTRY}/approval-frontend:${TAG}|g" k8s/base/approval-frontend-deployment.yaml
```

**Continue to UPGRADE-PART3-DEPLOYMENT.md for deployment steps...**

---

## 📁 PHASE 5: DEPLOY TO GKE

### 5.1 Deploy Infrastructure Services First

```bash
# Deploy in order (services with dependencies first)

# 1. PostgreSQL (if not using Cloud SQL)
# Note: We're using Cloud SQL, so skip this

# 2. Redis (if not using Memorystore)
# Note: We're using Memorystore, so skip this

# 3. RabbitMQ
kubectl apply -f k8s/base/rabbitmq-statefulset.yaml
kubectl apply -f k8s/base/rabbitmq-service.yaml

# Wait for RabbitMQ to be ready
kubectl wait --for=condition=ready pod -l app=rabbitmq --timeout=300s -n ai-system

# 4. Qdrant Vector Database
kubectl apply -f k8s/base/qdrant-deployment.yaml
kubectl apply -f k8s/base/qdrant-service.yaml

# Wait for Qdrant
kubectl wait --for=condition=ready pod -l app=qdrant --timeout=300s -n ai-system
```

### 5.2 Deploy Application Services

```bash
# 1. ConfigMap and Secrets (already created)
kubectl apply -f k8s/base/configmap.yaml

# 2. LLM Service
kubectl apply -f k8s/base/llm-service-deployment.yaml
kubectl apply -f k8s/base/llm-service-service.yaml

# 3. Worker Service
kubectl apply -f k8s/base/worker-service-deployment.yaml
kubectl apply -f k8s/base/worker-service-service.yaml

# 4. Approval Backend
kubectl apply -f k8s/base/approval-backend-deployment.yaml
kubectl apply -f k8s/base/approval-backend-service.yaml

# 5. Approval Frontend
kubectl apply -f k8s/base/approval-frontend-deployment.yaml
kubectl apply -f k8s/base/approval-frontend-service.yaml

# Check all pods are running
kubectl get pods -n ai-system

# Check logs if any pod is not running
kubectl logs -f deployment/llm-service -n ai-system
```

### 5.3 Set Up Ingress with Google Cloud Load Balancer

#### Create Ingress Resource

File: `k8s/gcp/ingress-gcp.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ai-system-ingress
  namespace: ai-system
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "ai-system-ip"
    networking.gke.io/managed-certificates: "ai-system-cert"
    kubernetes.io/ingress.allow-http: "true"
spec:
  rules:
  - host: your-domain.com  # Replace with your domain
    http:
      paths:
      - path: /api/*
        pathType: Prefix
        backend:
          service:
            name: approval-backend
            port:
              number: 3000
      - path: /llm/*
        pathType: Prefix
        backend:
          service:
            name: llm-service
            port:
              number: 8000
      - path: /*
        pathType: Prefix
        backend:
          service:
            name: approval-frontend
            port:
              number: 80
```

#### Create Managed Certificate

File: `k8s/gcp/managed-cert.yaml`

```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: ai-system-cert
  namespace: ai-system
spec:
  domains:
    - your-domain.com          # Replace with your domain
    - www.your-domain.com      # Replace with your domain
```

#### Reserve Static IP and Deploy Ingress

```bash
# Reserve static external IP
gcloud compute addresses create ai-system-ip \
  --global \
  --ip-version IPV4

# Get the IP address
gcloud compute addresses describe ai-system-ip --global

# Note the IP address (e.g., 34.123.45.67)
# Point your domain DNS A record to this IP

# Deploy managed certificate
kubectl apply -f k8s/gcp/managed-cert.yaml

# Deploy ingress
kubectl apply -f k8s/gcp/ingress-gcp.yaml

# Wait for ingress to be ready (can take 10-15 minutes)
kubectl get ingress ai-system-ingress -n ai-system --watch

# Check certificate provisioning status
kubectl describe managedcertificate ai-system-cert -n ai-system
```

### 5.4 Set Up Monitoring and Logging

#### Deploy Prometheus

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false

# Wait for all pods to be ready
kubectl get pods -n monitoring --watch
```

#### Deploy Grafana Dashboards

```bash
# Grafana is already installed with Prometheus stack
# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

# Port-forward Grafana (for testing)
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Access at http://localhost:3000
# Username: admin
# Password: (from above command)

# Import dashboards from monitoring/grafana/dashboards/
```

#### Configure Cloud Logging

```bash
# GKE automatically sends logs to Cloud Logging
# View logs in GCP Console:
# https://console.cloud.google.com/logs

# Or use gcloud CLI
gcloud logging read "resource.type=k8s_container AND resource.labels.namespace_name=ai-system" \
  --limit 50 \
  --format json

# Set up log-based metrics (optional)
gcloud logging metrics create error_rate \
  --description="Rate of error logs" \
  --log-filter='resource.type="k8s_container"
               AND resource.labels.namespace_name="ai-system"
               AND severity="ERROR"'
```

---

## 📁 PHASE 6: POST-DEPLOYMENT CONFIGURATION

### 6.1 Initialize Database Schema

```bash
# Connect to Cloud SQL via Cloud SQL Proxy (for schema creation)

# Download Cloud SQL Proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.0.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Get connection name
INSTANCE_CONNECTION_NAME=$(terraform output -raw database_connection_name -state=infrastructure/terraform/gcp/terraform.tfstate)

# Start proxy
./cloud-sql-proxy $INSTANCE_CONNECTION_NAME

# In another terminal, connect with psql
DB_PASSWORD=$(terraform output -raw database_password -state=infrastructure/terraform/gcp/terraform.tfstate)

PGPASSWORD=$DB_PASSWORD psql -h 127.0.0.1 -U devops -d devops_db

# Run schema creation SQL
\i infrastructure/database/init/001_initial_schema.sql
\i infrastructure/database/init/002_seed_data.sql

# Exit psql
\q

# Stop proxy
# Ctrl+C in the first terminal
```

### 6.2 Test Application Endpoints

```bash
# Get external IP from ingress
EXTERNAL_IP=$(kubectl get ingress ai-system-ingress -n ai-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "External IP: $EXTERNAL_IP"

# Test health endpoints
curl -k http://$EXTERNAL_IP/api/health
curl -k http://$EXTERNAL_IP/llm/health

# Test LLM service (will fail without API keys)
curl -k -X POST http://$EXTERNAL_IP/llm/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello, how are you?"}
    ]
  }'

# If you get a valid response, your system is working!
```

### 6.3 Set Up Cloud Build for CI/CD

#### Create Cloud Build Trigger

File: `cloudbuild.yaml` (in project root)

```yaml
steps:
  # Build LLM Service
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/llm-service:$SHORT_SHA'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/llm-service:latest'
      - './services/llm-service'
    id: 'build-llm-service'

  # Build Worker Service
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/worker-service:$SHORT_SHA'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/worker-service:latest'
      - './services/worker-service'
    id: 'build-worker-service'

  # Build Approval Backend
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-backend:$SHORT_SHA'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-backend:latest'
      - './services/approval-dashboard/backend'
    id: 'build-approval-backend'

  # Build Approval Frontend
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-frontend:$SHORT_SHA'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-frontend:latest'
      - './services/approval-dashboard/frontend'
    id: 'build-approval-frontend'

  # Push images
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags', 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/llm-service']
    waitFor: ['build-llm-service']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags', 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/worker-service']
    waitFor: ['build-worker-service']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags', 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-backend']
    waitFor: ['build-approval-backend']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags', 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-frontend']
    waitFor: ['build-approval-frontend']

  # Deploy to GKE
  - name: 'gcr.io/cloud-builders/kubectl'
    args:
      - 'set'
      - 'image'
      - 'deployment/llm-service'
      - 'llm-service=us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/llm-service:$SHORT_SHA'
      - '-n'
      - 'ai-system'
    env:
      - 'CLOUDSDK_COMPUTE_REGION=us-central1'
      - 'CLOUDSDK_CONTAINER_CLUSTER=ai-system-cluster'

  - name: 'gcr.io/cloud-builders/kubectl'
    args:
      - 'set'
      - 'image'
      - 'deployment/worker-service'
      - 'worker-service=us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/worker-service:$SHORT_SHA'
      - '-n'
      - 'ai-system'
    env:
      - 'CLOUDSDK_COMPUTE_REGION=us-central1'
      - 'CLOUDSDK_CONTAINER_CLUSTER=ai-system-cluster'

  - name: 'gcr.io/cloud-builders/kubectl'
    args:
      - 'set'
      - 'image'
      - 'deployment/approval-backend'
      - 'approval-backend=us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-backend:$SHORT_SHA'
      - '-n'
      - 'ai-system'
    env:
      - 'CLOUDSDK_COMPUTE_REGION=us-central1'
      - 'CLOUDSDK_CONTAINER_CLUSTER=ai-system-cluster'

  - name: 'gcr.io/cloud-builders/kubectl'
    args:
      - 'set'
      - 'image'
      - 'deployment/approval-frontend'
      - 'approval-frontend=us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-frontend:$SHORT_SHA'
      - '-n'
      - 'ai-system'
    env:
      - 'CLOUDSDK_COMPUTE_REGION=us-central1'
      - 'CLOUDSDK_CONTAINER_CLUSTER=ai-system-cluster'

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/llm-service:$SHORT_SHA'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/worker-service:$SHORT_SHA'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-backend:$SHORT_SHA'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/ai-system-images/approval-frontend:$SHORT_SHA'

options:
  machineType: 'E2_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY

timeout: '1800s'  # 30 minutes
```

#### Create Build Trigger in GCP Console

```bash
# Or via CLI
gcloud builds triggers create github \
  --name="ai-system-deploy" \
  --repo-name="your-repo-name" \
  --repo-owner="your-github-username" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml"

# Connect your GitHub repository when prompted
```

---

## 📁 PHASE 7: MONITORING & MAINTENANCE

### 7.1 Set Up Alerts

```bash
# Create alert for pod crashes
gcloud alpha monitoring policies create \
  --notification-channels=YOUR_CHANNEL_ID \
  --display-name="Pod Crash Alert" \
  --condition-display-name="Pod crashed" \
  --condition-expression='
    resource.type = "k8s_pod"
    AND resource.labels.namespace_name = "ai-system"
    AND metric.type = "kubernetes.io/pod/restart_count"
    AND metric.value > 3'

# Create alert for high error rate
# Create alert for high latency
# etc.
```

### 7.2 Set Up Backup Jobs

```bash
# Create CronJob for database backups
kubectl apply -f k8s/gcp/backup-cronjob.yaml
```

File: `k8s/gcp/backup-cronjob.yaml`

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: ai-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: google/cloud-sdk:alpine
            command:
            - /bin/sh
            - -c
            - |
              gcloud sql export sql INSTANCE_NAME \
                gs://BUCKET_NAME/backups/$(date +%Y%m%d-%H%M%S).sql \
                --database=devops_db
          restartPolicy: OnFailure
```

### 7.3 Regular Maintenance Checklist

**Weekly:**
- [ ] Review monitoring dashboards
- [ ] Check for any pod restarts
- [ ] Review error logs
- [ ] Check resource utilization

**Monthly:**
- [ ] Update dependencies (npm audit, pip-audit)
- [ ] Review and optimize costs
- [ ] Test backup restore procedure
- [ ] Update documentation

**Quarterly:**
- [ ] Security audit
- [ ] Performance review
- [ ] Capacity planning
- [ ] DR drill

---

## 📁 TROUBLESHOOTING

### Common Issues and Solutions

#### 1. Pods Not Starting

```bash
# Check pod status
kubectl get pods -n ai-system

# Describe pod to see events
kubectl describe pod POD_NAME -n ai-system

# Check logs
kubectl logs POD_NAME -n ai-system

# Common causes:
# - Image pull errors: Check image name and Artifact Registry permissions
# - Config errors: Check ConfigMap and Secrets
# - Resource limits: Check if node has enough resources
```

#### 2. Database Connection Issues

```bash
# Verify database is accessible
kubectl run -it --rm debug --image=postgres:15 --restart=Never -n ai-system -- \
  psql -h POSTGRES_IP -U devops -d devops_db

# Common causes:
# - Wrong IP address in ConfigMap
# - VPC peering not set up correctly
# - Firewall rules blocking connection
```

#### 3. Ingress Not Working

```bash
# Check ingress status
kubectl describe ingress ai-system-ingress -n ai-system

# Check if IP is assigned
kubectl get ingress ai-system-ingress -n ai-system

# Check managed certificate status
kubectl describe managedcertificate ai-system-cert -n ai-system

# Common causes:
# - DNS not pointing to correct IP
# - Certificate provisioning takes time (10-15 min)
# - Backend services not healthy
```

#### 4. High Costs

```bash
# Check GKE cluster costs
gcloud billing accounts list
gcloud billing budgets list --billing-account=ACCOUNT_ID

# Optimize:
# - Use preemptible/spot nodes for non-critical workloads
# - Scale down node pool during off-hours
# - Use committed use discounts
# - Review and delete unused resources
```

---

## 📁 COST ESTIMATION

### Monthly Cost Breakdown (Approximate)

| Resource | Configuration | Monthly Cost (USD) |
|----------|--------------|-------------------|
| **GKE Cluster** | 3 x e2-standard-4 nodes | ~$240 |
| **Cloud SQL** | db-custom-2-7680 (HA) | ~$250 |
| **Memorystore Redis** | 5GB Standard HA | ~$160 |
| **Cloud Load Balancer** | With SSL | ~$20 |
| **Artifact Registry** | Storage + egress | ~$10 |
| **Cloud Storage** | Backups | ~$5 |
| **Cloud Logging** | Standard tier | ~$50 |
| **Network Egress** | Internet | ~$50 |
| **Total** | | **~$785/month** |

### Cost Optimization Tips

1. **Use Preemptible Nodes** for worker service (save 60-80%)
2. **Scale down during off-hours** using cluster autoscaler
3. **Use committed use discounts** (save 25-50%)
4. **Optimize logging** - reduce retention or sampling
5. **Use Cloud CDN** for frontend assets
6. **Monitor with billing alerts**

---

## 🎉 DEPLOYMENT COMPLETE!

Your AI-Driven Hybrid Kubernetes System is now live on Google Cloud!

**Access Your Application:**
- Frontend: https://your-domain.com
- API: https://your-domain.com/api
- LLM Service: https://your-domain.com/llm

**Monitor Your System:**
- GCP Console: https://console.cloud.google.com/
- Kubernetes Dashboard: `kubectl proxy` then access http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
- Grafana: Port-forward or expose via ingress
- Cloud Monitoring: https://console.cloud.google.com/monitoring

**Next Steps:**
1. Set up custom domain and SSL
2. Configure monitoring alerts
3. Implement automated backups
4. Set up CI/CD pipeline
5. Load test your application
6. Document runbooks for operations team

**Need Help?**
- GKE Documentation: https://cloud.google.com/kubernetes-engine/docs
- Kubernetes Documentation: https://kubernetes.io/docs/
- Project Issues: Create an issue in your repository

---

**Congratulations on deploying to Google Cloud! 🚀**
