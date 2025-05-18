# OpenTelemetry Collector Setup for GCP

This repository contains the configuration for deploying an OpenTelemetry Collector to Kubernetes with Google Cloud integration.

## Prerequisites

- A GKE cluster with Workload Identity enabled
- `gcloud` CLI tool installed
- `kubectl` configured to access your GKE cluster
- Terraform (if using infrastructure as code)

## Setup Instructions

### 1. Create Google Cloud Service Account

#### Using gcloud CLI

```bash
# Create the service account
gcloud iam service-accounts create opentelemetry-collector \
    --display-name="OpenTelemetry Collector" \
    --project=yan-lab

# Grant necessary roles
gcloud projects add-iam-policy-binding yan-lab \
    --member="serviceAccount:opentelemetry-collector@yan-lab.iam.gserviceaccount.com" \
    --role="roles/cloudtrace.agent"

gcloud projects add-iam-policy-binding yan-lab \
    --member="serviceAccount:opentelemetry-collector@yan-lab.iam.gserviceaccount.com" \
    --role="roles/monitoring.metricWriter"

gcloud projects add-iam-policy-binding yan-lab \
    --member="serviceAccount:opentelemetry-collector@yan-lab.iam.gserviceaccount.com" \
    --role="roles/logging.logWriter"

# Allow the Kubernetes service account to impersonate the GCP service account
gcloud iam service-accounts add-iam-policy-binding opentelemetry-collector@yan-lab.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:yan-lab.svc.id.goog[opentelemetry-system/opentelemetry-collector]"
```

#### Using Terraform

```hcl
# Create the service account
resource "google_service_account" "opentelemetry_collector" {
  account_id   = "opentelemetry-collector"
  display_name = "OpenTelemetry Collector"
  project      = "yan-lab"
}

# Grant necessary roles
resource "google_project_iam_member" "cloudtrace_agent" {
  project = "yan-lab"
  role    = "roles/cloudtrace.agent"
  member  = "serviceAccount:${google_service_account.opentelemetry_collector.email}"
}

resource "google_project_iam_member" "monitoring_metric_writer" {
  project = "yan-lab"
  role    = "roles/monitoring.metricWriter"
  member  = "serviceAccount:${google_service_account.opentelemetry_collector.email}"
}

resource "google_project_iam_member" "logging_log_writer" {
  project = "yan-lab"
  role    = "roles/logging.logWriter"
  member  = "serviceAccount:${google_service_account.opentelemetry_collector.email}"
}

# Allow the Kubernetes service account to impersonate the GCP service account
resource "google_service_account_iam_member" "workload_identity_user" {
  service_account_id = google_service_account.opentelemetry_collector.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:yan-lab.svc.id.goog[opentelemetry-system/opentelemetry-collector]"
}
```

### 2. Environment-Specific Configuration

#### Development Environment

1. Update the service account name in `k8s/overlays/dev/kustomization.yaml`:
```yaml
patches:
  - target:
      kind: ServiceAccount
      name: opentelemetry-collector
    patch: |-
      - op: replace
        path: /metadata/annotations/iam.gke.io~1gcp-service-account
        value: opentelemetry-collector@yan-lab.iam.gserviceaccount.com
```

2. Apply the configuration:
```bash
kubectl apply -k k8s/overlays/dev
```

#### Production Environment

1. Create a new overlay for production:
```bash
mkdir -p k8s/overlays/production
```

2. Create a production-specific kustomization file with appropriate resource limits and replicas.

3. Update the service account name in the production overlay.

4. Apply the configuration:
```bash
kubectl apply -k k8s/overlays/production
```

## Verification

To verify the setup is working:

1. Check the collector logs:
```bash
kubectl logs -n opentelemetry-system -l app=opentelemetry-collector
```

2. Verify data is being exported to:
   - Google Cloud Trace
   - Google Cloud Monitoring
   - Google Cloud Logging

## Troubleshooting

If you encounter permission issues:

1. Verify the service account has the correct roles:
```bash
gcloud projects get-iam-policy yan-lab
```

2. Check the Workload Identity binding:
```bash
gcloud iam service-accounts get-iam-policy opentelemetry-collector@yan-lab.iam.gserviceaccount.com
```

3. Ensure the Kubernetes service account exists:
```bash
kubectl get serviceaccount opentelemetry-collector -n opentelemetry-system
```
