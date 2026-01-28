# Infrastructure: Pulumi IaC & Kustomize Patterns

**When to use this file:** Reference this for infrastructure provisioning (Pulumi), Cloud Run configuration (Kustomize), and understanding the infrastructure/application separation pattern.

**Related documentation:**
- For Polylith architecture concepts, see [01-setup.md](01-setup.md)
- For deployment mechanics (Docker, Cloud Functions), see [05-deployment.md](05-deployment.md)
- For CI/CD workflows, see [06-automation.md](06-automation.md)

---

## Table of Contents

1. [Infrastructure and Application Separation](#1-infrastructure-and-application-separation)
2. [Two-Stack Pattern](#2-two-stack-pattern)
3. [Three-Stack Pattern (Advanced)](#3-three-stack-pattern-advanced)
4. [Infrastructure Dependencies](#4-infrastructure-dependencies)
5. [Pulumi IaC Patterns](#5-pulumi-iac-patterns)
6. [Kustomize for Cloud Run](#6-kustomize-for-cloud-run)

---

## 1. Infrastructure and Application Separation

A fundamental architectural principle in Polylith repositories is the **separation of infrastructure definitions from application code**. This separation is critical for managing different change frequencies and deployment lifecycles.

### Core Principle: Lifecycle-Based Separation

Infrastructure and application code have fundamentally different lifecycles:

| Aspect               | Infrastructure (Foundation)                                                | Application Code                                    |
| -------------------- | -------------------------------------------------------------------------- | --------------------------------------------------- |
| **Change Frequency** | Rare (weeks to months)                                                     | Frequent (multiple times per day)                   |
| **Examples**         | Service accounts, Pub/Sub topics, Cloud Run services, Kubernetes manifests | Business logic, data transformations, API endpoints |
| **Deployment**       | Manual, deliberate, requires approval                                      | Automated, continuous, part of CI/CD                |
| **Risk Profile**     | High impact, affects multiple applications                                 | Lower blast radius, isolated to single app          |
| **Ownership**        | Platform/Infrastructure team                                               | Application development team                        |

**Key Insight**: When infrastructure and application code are mixed together, every application change requires redeploying infrastructure, leading to:
- Slow deployments (15-20 minutes instead of 2-3 minutes)
- Unnecessary risk (infrastructure changes when only app changed)
- Deployment bottlenecks (can't deploy apps independently)
- Difficult rollbacks (infrastructure and app coupled)

### The `infrastructure/` Directory Pattern

Polylith repositories MUST maintain a dedicated `infrastructure/` directory at the workspace root, separate from Polylith bricks.

**Directory structure:**
```
workspace-root/
├── bases/              # Application logic (Polylith)
├── components/         # Application logic (Polylith)
├── projects/           # Application deployments
├── infrastructure/     # Infrastructure definitions (separate)
│   ├── pyproject.toml  # Infrastructure dependencies (Pulumi, providers)
│   ├── cloudrun/       # Cloud Run service manifests (Kustomize base + overlays)
│   ├── cloudrunjob/    # Cloud Run job manifests (Kustomize base + overlays)
│   ├── cloudfunction/  # Cloud Function deployment configs (gcloud or other tools)
│   └── pulumi/         # Pulumi IaC for service accounts, Cloud Storage, IAM, etc.
└── ...
```

**What belongs in `infrastructure/`:**
- Cloud Run service/job manifests (Kustomize base + overlays for resource limits, scaling, health checks)
- Cloud Function deployment configurations (entry point definitions, runtime settings)
- Infrastructure as Code (Pulumi stacks for service accounts, IAM policies, Cloud Storage buckets)
- Environment-specific configuration overlays
- Pub/Sub topics, BigQuery datasets, Artifact Registry repositories
- Networking, VPCs, firewall rules (if applicable)

**What belongs in Polylith workspace (`bases/`, `components/`, `projects/`):**
- Business logic and domain code
- Data processing and transformations
- API routes and CLI commands (bases)
- Reusable libraries and utilities (components)
- Application configuration (settings, environment variables)

### Benefits of Separation

1. **Independent Deployment Cadences**
   - Deploy infrastructure rarely and deliberately
   - Deploy applications frequently and automatically
   - No unnecessary infrastructure changes when only app code changed

2. **Reduced Blast Radius**
   - Infrastructure changes don't trigger application redeployment
   - Application changes don't risk infrastructure stability
   - Easier to isolate and debug issues

3. **Faster Application Deployments**
   - Application-only changes deploy in 2-3 minutes
   - No waiting for infrastructure validation
   - Parallel deployments of multiple applications

4. **Clear Ownership and Responsibilities**
   - Platform team manages `infrastructure/`
   - Application teams manage `projects/`, `bases/`, `components/`
   - Clear boundaries for code review and approvals

5. **Better CI/CD Workflows**
   - Separate workflows for infrastructure vs application
   - Different approval gates and testing strategies
   - Infrastructure changes require manual approval
   - Application changes auto-deploy on merge

6. **Improved Rollback Capability**
   - Roll back application without touching infrastructure
   - Roll back infrastructure without redeploying applications
   - Independent version control

### Contrast with Monolithic Approach

**Without separation (anti-pattern):**
```
projects/my_app/
├── infrastructure.yaml  # Coupled with application
├── app_code.py
└── deploy.sh            # Deploys both together
```

**Problems:**
- Every app change redeploys infrastructure (slow, risky)
- Infrastructure changes require full application rebuild
- Can't deploy multiple apps independently
- Difficult to manage environment-specific infrastructure

**With separation (recommended):**
```
infrastructure/
└── cloudrunjob/
    └── my-job/
        ├── base/job.yaml
        └── overlays/
            └── production/kustomization.yaml

projects/my_app/
├── Dockerfile
└── app_code.py
```

**Benefits:**
- Infrastructure deployed once, applications deploy frequently
- Clear boundaries and ownership
- Fast, independent application deployments
- Environment-specific infrastructure managed centrally

---

## 2. Two-Stack Pattern

Most Polylith repositories implement a **2-stack pattern**:

### Stack 1: Infrastructure (Foundation)

- **Location:** `infrastructure/pulumi/` directory
- **Content:** Foundational GCP resources (service accounts, Cloud Storage, IAM, Artifact Registry)
- **Change frequency:** Rare (weeks to months)
- **Deployment:** Manual, via separate workflow (`infrastructure-provision.yaml`)
- **Examples:**
  - **Cloud Storage buckets**: For application data storage
  - **Service Accounts with IAM roles**: For Cloud Run or Cloud Functions identity
  - **Artifact Registry repositories**: For Docker image or package storage
  - **Pub/Sub topics**: For event streaming infrastructure
  - **BigQuery datasets and tables**: For data warehouse
  - **Cloud Scheduler jobs**: For triggering Cloud Run Jobs
- **Note**: This stack provisions infrastructure resources but does NOT deploy applications

### Stack 2: Application

- **Location:** `projects/` directory + `infrastructure/cloudrun/` or `infrastructure/cloudrunjob/` directory
- **Content:** Application code and deployment configuration working together
- **Change frequency:** Frequent (multiple times per day)
- **Deployment:** Automated, triggered by code changes (`deploy.yaml`, `sandbox-deploy.yaml`)
- **Components:**
  - **`projects/{project_name}/`**: Dockerfile or deployment scripts assemble application from Polylith bricks
  - **`infrastructure/cloudrun/` or `infrastructure/cloudrunjob/`**: Service/job manifests or deployment configurations
  - These two work together: Build the application, configure it, deploy it

### Deployment Flow Examples

**Cloud Run Jobs pattern:**
```
Infrastructure Foundation changes (rare):
  infrastructure/pulumi/ → pulumi up →
    Provision service accounts, buckets, IAM, Artifact Registry, BigQuery, Cloud Scheduler

Application changes (frequent):
  projects/{project}/ → Build Docker image from Polylith bricks →
    Push to Artifact Registry →
  infrastructure/cloudrunjob/ → Kustomize build manifest →
    gcloud run jobs replace → Deploy new job revision
```

**Key insight**: Folders like `infrastructure/cloudrun/`, `infrastructure/cloudrunjob/`, or `infrastructure/cloudfunction/` are part of the **application stack**, not the foundation stack. They contain deployment configurations (resource limits, scaling, health checks, environment variables) that change when you deploy new application code, even though they live in the `infrastructure/` directory.

### Real Example: Cloud Run Jobs with Kustomize

**Stack 1 (Infrastructure Foundation): `infrastructure/pulumi/`**
```
infrastructure/
├── pyproject.toml        # Infrastructure dependencies (Pulumi, providers)
└── pulumi/
    ├── __main__.py       # Pulumi program for GCP resources
    ├── helpers/
    │   └── naming.py     # Resource naming conventions
    ├── bigquery/         # BigQuery table schemas
    ├── monitoring/       # Dashboard JSON files
    └── Pulumi.{stack}.yaml  # Stack configurations (sandbox, production)
```

**What's provisioned (rarely changes):**
- Cloud Storage buckets for application data
- Service Accounts with IAM roles for Cloud Run
- Artifact Registry repositories for Docker images
- BigQuery datasets and tables
- Cloud Scheduler jobs to trigger Cloud Run Jobs
- Log-based metrics and alert policies

**Stack 2 (Application): `infrastructure/cloudrunjob/` + `projects/{job}/`**

*Part A: `infrastructure/cloudrunjob/` - Job deployment configuration*
```
infrastructure/cloudrunjob/
└── {job-name}/
    ├── base/
    │   ├── job.yaml           # Base Cloud Run Job definition
    │   └── kustomization.yaml
    └── overlays/
        └── production/
            └── kustomization.yaml  # Production-specific patches
```

**What's configured:**
- Resource limits (CPU, memory)
- Task configuration (taskCount, maxRetries, timeoutSeconds)
- Service account assignments
- Secret references
- Environment-specific values (project IDs, region codes)

*Part B: `projects/{job}/` - Application code*
```
projects/{job}/
├── pyproject.toml    # Dependencies and Polylith brick references
├── Dockerfile        # Application container definition
└── main.py           # Entry point
```

**What's included:**
- Business logic built from Polylith bricks (bases/components)
- Application entry point
- Runtime dependencies

**Deployment flow:**

1. **Infrastructure Foundation changes** (rare - weeks/months):
   ```yaml
   # Trigger: Manual via workflow_dispatch (production) or PR label (sandbox)
   # Workflow: .github/workflows/infrastructure-provision.yaml

   steps:
     - Set up Python and uv
     - Install Pulumi dependencies
     - Authenticate to GCP
     - Run: pulumi up --stack {production|sandbox}
   ```

   **Developer action**: Modify `infrastructure/pulumi/__main__.py` or stack config, then trigger workflow

2. **Application changes** (frequent - multiple times per day):
   ```yaml
   # Trigger: Manual workflow_dispatch (production) or PR label (sandbox)

   steps:
     - Build and push Docker image
     - Update image reference in Kustomize
     - Build final manifest with kustomize
     - Deploy with: gcloud run jobs replace
   ```

**Key benefits achieved:**
- Infrastructure foundation (Pulumi) separate from application deployment (Kustomize + Docker)
- Application deploys in ~2-3 minutes (Docker build + Kustomize + gcloud)
- Infrastructure foundation changes don't require application redeployment
- Clear ownership: platform team owns `infrastructure/pulumi/`, app team owns `infrastructure/cloudrunjob/` + `projects/`

---

## 3. Three-Stack Pattern (Advanced)

{{ PLACEHOLDER }}

---

## 4. Infrastructure Dependencies

The `infrastructure/pyproject.toml` file is separate from both workspace and project configurations and follows different patterns.

### Location and Characteristics

**Location:** `infrastructure/pyproject.toml` (at infrastructure root, same level as `pulumi/`)

**Key characteristics:**
- **Not included in workspace members:** The `infrastructure/` directory is NOT listed in workspace root `[tool.uv.workspace] members`
- **Isolated dependency management:** Has its own isolated dependency management, separate from application stack
- **Infrastructure-specific dependencies:** Contains Pulumi and GCP provider packages not needed by application code
- **Independent lifecycle:** Updated separately via `infrastructure-provision.yaml` workflow

### Example Configuration

```toml
[project]
name = "data-integration-infrastructure"
version = "1.0.0"
description = "Infrastructure provisioning for data integration"
requires-python = "~=3.13.0"
# Note: No 'readme' field - infrastructure pyproject.toml is isolated
dependencies = [
    "pulumi>=3.213.0",
    "pulumi-gcp>=9.6.0",
]
```

### Important Notes

- **No `readme` field:** Infrastructure `pyproject.toml` does NOT include a `readme` field. The `infrastructure/` directory should not contain a `README.md` file.
- **Documentation location:** Infrastructure patterns are documented in Claude rules (this file), not in scattered README files within the infrastructure directory.

### Why Separate

- Infrastructure provisioning (Stack 1) has different lifecycle than application deployment (Stack 2)
- Pulumi dependencies are large and not needed for application runtime
- Infrastructure changes are rare; application changes are frequent
- Clear separation of concerns between platform and application teams

---

## 5. Pulumi IaC Patterns

This section documents patterns for writing Pulumi Infrastructure as Code, based on actual implementation files.

### Directory Structure

```
infrastructure/pulumi/
├── __main__.py           # Main Pulumi program
├── helpers/
│   └── naming.py         # Resource naming utilities
├── bigquery/             # BigQuery table schema JSON files
│   └── {dataset}/
│       └── {table}.json
├── monitoring/           # Cloud Monitoring dashboard JSON
│   └── dashboard.json
├── Pulumi.yaml           # Pulumi project definition
└── Pulumi.{stack}.yaml   # Stack-specific configurations
```

### Configuration File Structure

Stack configuration files (`Pulumi.{stack}.yaml`) define all infrastructure resources:

```yaml
config:
  gcp:project: my-gcp-project
  input:bucket: my-state-bucket
  input:environment: production
  label:component: data-pipeline

  input:projects:
    project-a:
      service_account: ...
      artifact_registry: ...
      bigquery_dataset: ...
    project-b:
      ...
```

### YAML Anchors for Reusability

Use YAML anchors (`&anchor_name`) and aliases (`*anchor_name`) to avoid duplication:

```yaml
config:
  # Define reusable configurations with anchors
  input:artifact_registry_cleanup_policies: &artifact_registry_cleanup_policies
    - action: DELETE
      id: delete-prerelease
      condition:
        older_than: 2592000s
    - action: KEEP
      id: keep-minimum-versions
      most_recent_versions:
        keep_count: 10

  input:log_metric_distribution_descriptor: &log_metric_distribution_descriptor
    metric_kind: DELTA
    value_type: DISTRIBUTION
    unit: "1"

  # Reference with aliases
  input:projects:
    my-project:
      artifact_registry:
        cleanup_policies: *artifact_registry_cleanup_policies
```

**Benefits:**
- Single source of truth for shared configurations
- Reduces copy-paste errors
- Easy to update all usages at once

### Project-Based Organization

Group resources by logical project under `input:projects`:

```yaml
input:projects:
  dmd:
    service_account:
      account_id: cloud-run-dmd
      description: Service account for DMD Cloud Run Jobs
      display_name: Cloud Run Job DMD
    service_account_iam:
      account_id: cloud-run-dmd
      roles:
        - roles/bigquery.admin
        - roles/run.invoker
        - roles/logging.logWriter
    artifact_registry:
      repository_id: data-integration-dmd
      format: DOCKER
      location: asia-southeast1
    bigquery_dataset:
      dataset_id: bronze_dmd
      location: asia-southeast1
    bigquery_tables:
      - table_id: inventory_movement
        dataset_id: bronze_dmd
        schema_file: bigquery/bronze_dmd/inventory_movement.json
    cloud_scheduler:
      - name: dmd-inv-mov
        target_job: dmd-inv-mov-api2bq
        schedule: "0 4 * * *"
```

### Config Namespacing

Use prefixes to organize configuration values:

| Prefix | Purpose | Examples |
|--------|---------|----------|
| `gcp:` | GCP provider settings | `gcp:project` |
| `input:` | Input values for resources | `input:projects`, `input:monitoring` |
| `label:` | Labels applied to resources | `label:component` |

### Resource Loop Pattern

The `__main__.py` iterates through projects and creates resources by type:

```python
def main() -> None:
    gcp_config = pulumi.Config("gcp")
    input_config = pulumi.Config("input")

    projects = input_config.require_object("projects")

    # Storage for created resources
    service_accounts: dict[str, gcp.serviceaccount.Account] = {}

    # Create resources by type (not by project)
    for project_name, project_config in projects.items():
        # Service Accounts
        service_account = gcp.serviceaccount.Account(
            resource_name=make_resource_name("serviceaccount", project_config["service_account"]["account_id"]),
            **project_config["service_account"],
        )
        service_accounts[project_name] = service_account

        # IAM bindings
        for role in project_config["service_account_iam"]["roles"]:
            gcp.projects.IAMMember(
                resource_name=make_resource_name("iammember", project_config["service_account"]["account_id"], role),
                member=service_account.member,
                project=gcp_project,
                role=role,
            )

    # Continue with other resource types...
```

**Why this pattern:**
- Resources of the same type are grouped together for readability
- Dependencies between resource types are clear
- Easy to add new projects without restructuring code

### Naming Convention Helper

Use `make_resource_name()` for consistent Pulumi resource names:

```python
# infrastructure/pulumi/helpers/naming.py

def make_resource_name(resource_type: str, *identifiers: str) -> str:
    """Generate standardized Pulumi resource name.

    Args:
        resource_type: Type prefix (e.g., 'bucket', 'serviceaccount', 'registry')
        *identifiers: One or more identifying strings

    Returns:
        str: Standardized resource name in format: {type}_{id1}_{id2}...

    Examples:
        >>> make_resource_name("bucket", "my-bucket")
        'bucket_my_bucket'
        >>> make_resource_name("registry", "data-integration", "asia-southeast1")
        'registry_data_integration_asia_southeast1'
        >>> make_resource_name("iam", "my-sa", "roles/run.invoker")
        'iam_my_sa_run_invoker'
    """
    def normalize(s: str) -> str:
        return s.replace("-", "_").replace(".", "_").replace("/", "_")

    parts = [resource_type] + [normalize(identifier) for identifier in identifiers]
    return "_".join(parts)
```

**Usage:**
```python
gcp.storage.Bucket(
    resource_name=make_resource_name("bucket", "my-data-bucket"),
    name="my-data-bucket",
    ...
)
```

### Supplementary File Loading Pattern

Load external files for complex resource configurations:

```python
from pathlib import Path
import json

# BigQuery table schema from JSON file
schema_file = table_config["schema_file"]  # e.g., "bigquery/bronze_dmd/inventory_movement.json"
schema_path = Path(__file__).resolve().parent / schema_file

if not schema_path.exists():
    raise FileNotFoundError(f"Schema file not found: {schema_file}")

schema_json = schema_path.read_text()

gcp.bigquery.Table(
    resource_name=make_resource_name("table", dataset_id, table_id),
    schema=schema_json,
    ...
)

# Cloud Monitoring dashboard from JSON file
dashboard_path = Path(__file__).resolve().parent / "monitoring/dashboard.json"
dashboard_json = dashboard_path.read_text()
dashboard_data = json.loads(dashboard_json)

gcp.monitoring.Dashboard(
    resource_name=make_resource_name("dashboard", "my-dashboard"),
    dashboard_json=json.dumps(dashboard_data),
)
```

**Directory structure for supplementary files:**
```
infrastructure/pulumi/
├── bigquery/
│   └── bronze_dmd/
│       ├── inventory_movement.json
│       └── sku_location_movement.json
└── monitoring/
    └── dashboard.json
```

### Cloud Scheduler to Cloud Run Jobs Integration

Configure Cloud Scheduler to trigger Cloud Run Jobs:

```yaml
# In Pulumi.{stack}.yaml
cloud_scheduler:
  - name: dmd-inv-mov
    target_job: dmd-inv-mov-api2bq  # Cloud Run Job name
    region: asia-southeast1
    description: Daily DMD inventory movement extraction
    schedule: "0 4 * * *"
    time_zone: Asia/Kuala_Lumpur
```

```python
# In __main__.py
for scheduler_config in scheduler_configs:
    scheduler = scheduler_config.copy()
    target_job = scheduler.pop("target_job")

    # Build Cloud Run Job URL
    job_url = f"https://{scheduler['region']}-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/{gcp_project}/jobs/{target_job}:run"

    gcp.cloudscheduler.Job(
        resource_name=make_resource_name("scheduler", scheduler["name"]),
        **scheduler,
        http_target=gcp.cloudscheduler.JobHttpTargetArgs(
            uri=job_url,
            http_method="POST",
            oauth_token=gcp.cloudscheduler.JobHttpTargetOauthTokenArgs(
                service_account_email=service_accounts[project_name].email,
            ),
        ),
    )
```

---

## 6. Kustomize for Cloud Run

This section covers Kustomize patterns for Cloud Run Services, Jobs, and Functions.

### 6.1 Cloud Run Services

Cloud Run Services are 24/7 running applications that handle HTTP requests (web servers, APIs).

#### Directory Structure

```
infrastructure/cloudrun/
├── base/
│   ├── service.yaml        # Base Cloud Run service definition
│   └── kustomization.yaml  # Base kustomization config
└── overlays/
    ├── sandbox/
    │   └── kustomization.yaml  # Sandbox-specific patches
    └── production/
        └── kustomization.yaml  # Production-specific patches
```

#### Base Service Configuration

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-service
  labels:
    component: my-component
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      labels:
        component: my-component
      annotations:
        run.googleapis.com/cpu-throttling: "false"
        run.googleapis.com/startup-cpu-boost: "true"
        autoscaling.knative.dev/minScale: "0"
        autoscaling.knative.dev/maxScale: "2"
    spec:
      serviceAccountName: my-service@PROJECT_ID.iam.gserviceaccount.com
      timeoutSeconds: 300
      containers:
      - name: my-service
        image: IMAGE_URL
        ports:
        - containerPort: 8080
          name: http1
        env:
        - name: ENVIRONMENT
          value: ENVIRONMENT_PLACEHOLDER
        - name: VERSION
          value: VERSION_PLACEHOLDER
        startupProbe:
          httpGet:
            path: /health_check
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health_check
            port: 8080
        resources:
          limits:
            cpu: "1"
            memory: 2Gi
  traffic:
  - latestRevision: true
    percent: 100
```

#### Key Service Elements

- **`apiVersion: serving.knative.dev/v1`**: Cloud Run Services use Knative Serving API
- **`kind: Service`**: Indicates a Cloud Run Service
- **`ports`**: HTTP port configuration (required for services)
- **`startupProbe` / `livenessProbe`**: Health check configuration
- **`autoscaling.knative.dev/*`**: Min/max instance scaling
- **`traffic`**: Traffic routing to revisions

#### Startup Probe Configuration (Important)

Cloud Run uses startup probes to determine when a container is ready. **Aggressive settings can cause deployment failures.**

**Recommended settings:**
```yaml
startupProbe:
  httpGet:
    path: /health_check
    port: 8080
  initialDelaySeconds: 5   # Wait before first probe
  timeoutSeconds: 5        # Allow 5 seconds for response
  periodSeconds: 10        # Probe every 10 seconds
  successThreshold: 1      # 1 success to pass
  failureThreshold: 3      # Allow 3 failures (~35s total startup window)
```

**Common mistakes to avoid:**
- `failureThreshold: 1` - Container fails immediately on first probe failure
- `initialDelaySeconds: 0` - Probes before application can start
- `timeoutSeconds: 1` - Too short for slow initialization

### 6.2 Cloud Run Jobs

Cloud Run Jobs are batch tasks that run to completion (data pipelines, scheduled tasks).

#### Directory Structure

```
infrastructure/cloudrunjob/
└── {job-name}/
    ├── base/
    │   ├── job.yaml           # Base Cloud Run Job definition
    │   └── kustomization.yaml
    └── overlays/
        └── production/
            └── kustomization.yaml  # Production-specific patches
```

#### Base Job Configuration

```yaml
apiVersion: run.googleapis.com/v1
kind: Job
metadata:
  name: my-job-api2bq
  labels:
    component: data-pipeline
    source: my-api
  annotations:
    run.googleapis.com/launch-stage: BETA
spec:
  template:
    metadata:
      labels:
        component: data-pipeline
        source: my-api
      annotations:
        run.googleapis.com/execution-environment: gen2
    spec:
      taskCount: 1
      template:
        spec:
          maxRetries: 3
          timeoutSeconds: 600
          containers:
          - name: worker
            image: IMAGE_URL
            env:
            - name: COMPONENT
              value: data-pipeline
            - name: LOG_EXECUTION_ID
              value: "true"
            resources:
              limits:
                cpu: "1"
                memory: 512Mi
```

#### Key Job Elements

- **`apiVersion: run.googleapis.com/v1`**: Cloud Run Jobs use GCP-specific API (not Knative)
- **`kind: Job`**: Indicates a Cloud Run Job
- **`taskCount`**: Number of parallel tasks (usually 1)
- **`maxRetries`**: Retry attempts on failure
- **`timeoutSeconds`**: Maximum execution time
- **No ports**: Jobs don't expose HTTP endpoints
- **No health probes**: Jobs run to completion, no liveness checks

#### Production Overlay Example

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: my-gcp-project

resources:
  - ../../base

images:
  - name: IMAGE_URL
    newName: IMAGE_URL_PLACEHOLDER

patches:
  # Secrets annotation
  - patch: |-
      - op: add
        path: /spec/template/metadata/annotations/run.googleapis.com~1secrets
        value: "API_TOKEN:projects/my-project/secrets/MY_API_TOKEN"
    target:
      kind: Job
      name: my-job-api2bq

  # Service account
  - patch: |-
      - op: add
        path: /spec/template/spec/template/spec/serviceAccountName
        value: my-service-account@my-project.iam.gserviceaccount.com
    target:
      kind: Job
      name: my-job-api2bq

  # Environment variables
  - patch: |-
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: ENVIRONMENT
          value: production
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: VERSION
          value: VERSION_PLACEHOLDER
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: GCP_PROJECT_ID
          value: my-project
      - op: add
        path: /spec/template/spec/template/spec/containers/0/env/-
        value:
          name: API_TOKEN
          valueFrom:
            secretKeyRef:
              name: API_TOKEN
              key: latest
    target:
      kind: Job
      name: my-job-api2bq
```

#### Deployment Command

```bash
# Build manifest and deploy
cd infrastructure/cloudrunjob/my-job/overlays/production
kustomize edit set image IMAGE_URL=asia-southeast1-docker.pkg.dev/.../my-job:${VERSION}
kustomize build . | sed "s|VERSION_PLACEHOLDER|${VERSION}|g" > /tmp/job.yaml
gcloud run jobs replace /tmp/job.yaml --region=asia-southeast1
```

### 6.3 Cloud Run Functions

{{ PLACEHOLDER }}

### 6.4 Common Patterns

#### JSON Patch Escaping

Annotation keys contain `/` which conflicts with JSON Pointer path separators. Use `~1` to escape:

```yaml
# Wrong - slash interpreted as path separator
- op: replace
  path: /spec/template/metadata/annotations/run.googleapis.com/secrets
  value: "..."

# Correct - slash escaped with ~1
- op: replace
  path: /spec/template/metadata/annotations/run.googleapis.com~1secrets
  value: "..."
```

**Escape sequences:**
- `~0` escapes `~` (tilde)
- `~1` escapes `/` (forward slash)

#### Placeholder Substitution

Use placeholders in manifests that are replaced during deployment:

| Placeholder | Replacement | How |
|-------------|-------------|-----|
| `IMAGE_URL` | Full image path | `kustomize edit set image` |
| `VERSION_PLACEHOLDER` | Version string | `sed` substitution |

```bash
# Set image
kustomize edit set image IMAGE_URL=asia-southeast1-docker.pkg.dev/project/repo/image:v1.2.3

# Build and substitute version
kustomize build . | sed "s|VERSION_PLACEHOLDER|v1.2.3|g" > /tmp/manifest.yaml
```

#### Base Must Define Paths for Overlay Patches

`op: replace` requires the target path to exist in the base. Define placeholder values:

```yaml
# Base - define annotations for overlays to patch
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/minScale: "0"   # Default, overridden by overlays
        autoscaling.knative.dev/maxScale: "2"   # Default
```

```yaml
# Overlay can now use op: replace
patches:
  - patch: |-
      - op: replace
        path: /spec/template/metadata/annotations/autoscaling.knative.dev~1minScale
        value: "1"
```

#### Debugging Kustomize

Preview the final manifest before deployment:

```bash
cd infrastructure/cloudrunjob/my-job/overlays/production
kustomize build .
```

#### Service vs Job Comparison

| Aspect | Cloud Run Service | Cloud Run Job |
|--------|-------------------|---------------|
| **API Version** | `serving.knative.dev/v1` | `run.googleapis.com/v1` |
| **Kind** | `Service` | `Job` |
| **Use Case** | 24/7 web servers, APIs | Batch tasks, scheduled jobs |
| **Trigger** | HTTP requests | Cloud Scheduler, manual |
| **Lifecycle** | Always running (autoscales to 0) | Runs to completion |
| **Health Checks** | Required (startupProbe, livenessProbe) | Not applicable |
| **Key Fields** | `ports`, `traffic`, autoscaling | `taskCount`, `maxRetries`, `timeoutSeconds` |
| **Deploy Command** | `gcloud run services replace` | `gcloud run jobs replace` |
