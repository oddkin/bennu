# Flux provisioning

**Summary:** This strategy document outlines a high-compliance, scalable GitOps architecture tailored for the Flux and the Flux Operator. It decouples **Platform Logic** (Blueprints) from **Environment State** (Inputs), ensuring that your deployment pipeline is both flexible for developers and rigid for compliance.

### **Executive Summary**

* **Architecture:** Two-Repo Model (Catalog vs. Fleet).  
* **Templating Engine:** ControlPlane ResourceSet (Blueprints).  
* **Configuration Engine:** ResourceSetInputProvider (Environment Specifics).  
* **Secrets:** External Secrets Operator (ESO) bridging unique Vaults to Kubernetes Secrets.  
* **Promotion:** Directory-based promotion via Pull Requests on the Fleet Repo.

**1. Repository Architecture**

We will split concerns into two distinct repositories to enforce a clean separation between "How things are deployed" (Logic) and "Where things are deployed" (State).

#### **Repo A: The Platform Catalog (acme-platform-blueprints)**

* **Purpose:** Stores the "Logic" and "Templates."  
* **Content:** Generic ResourceSet definitions. These act as functions: they take inputs and generate Kubernetes resources.  
* **Access:** Owned by SRE/Platform Engineering. Developers can contribute via PRs to add new services or dependency logic.  
* **Versioning:** Tagged releases (e.g., v1.2.0).

#### **Repo B: The Fleet State (acme-fleet-config)**

* **Purpose:** Stores the "State" and "Inputs."  
* **Content:** Flux Bootstrap configurations, ResourceSetInputProvider definitions, and ESO configuration.  
* **Structure:** Organized by Environment > Cluster > Tenant.  
* **Access:**  
  * dev/*: Writable by developers (auto-merge/weak protection).  
  * staging/*: Protected (SRE approval required).  
  * prod/*: Restricted (4-eyes principle + SRE approval).

**2\. The Component Model (Repo A)**

In acme-platform-blueprints, we define generic patterns rather than specific deployments.

**Artifact:** apps/standard-service/resourceset.yaml

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: standard-layered-app
spec:
  # The Logic: Define resources using Go Templates
  resources:
    - kind: HelmRelease
      name: {{ .inputs.appName }}-db
      chart: { ... }
        
    - kind: HelmRelease
      name: {{ .inputs.appName }}-api
      dependsOn:
        - name: {{ .inputs.appName }}-db # Enforce Layered Dependency
      values:
        image: {{ .inputs.image }}
        # Reference the Secret that ESO will create
        existingSecret: {{ .inputs.appName }}-secrets
``` 

**3\. The Configuration Model (Repo B)**

In acme-fleet-config, we instantiate the Blueprints. This is where the specific "Variables" for each cluster live.

**Directory Structure:**

```plaintext
/clusters
  /dev-use1-01
    /base          # Cluster-level infra (ESO, Ingress)
    /tenants
      /payment-app
        ├── input.yaml  # The "Configuration"
        └── sync.yaml   # The "Pointer"
  /prod-use1-01
    ...
```

**Artifact:** clusters/dev-use1-01/tenants/payment-app/input.yaml

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSetInputProvider
metadata:
  name: payment-app-inputs
  namespace: payment-dev
spec:
  # Non-sensitive configuration specific to this cluster
  values:
    appName: "payment-processor"
    image: "registry.acme.com/payment:v2.1.0-rc1"
    replicas: 2
    logLevel: "DEBUG"
    environment: "dev"
```

**Artifact:** clusters/dev-use1-01/tenants/payment-app/sync.yaml

```yaml
# This tells Flux to fetch the Blueprint from Repo A and apply it using Inputs from above
apiVersion: fluxcd.controlplane.io/v1
kind: ResourceSet
metadata:
  name: payment-app-sync
  namespace: payment-dev
spec:
  sourceRef:
    kind: GitRepository
    name: platform-blueprints # Points to Repo A
    namespace: flux-system
  path: "./apps/standard-service" # Path inside Repo A
    
  # Binds the logic to the inputs defined in this cluster
  inputsFrom:
    - kind: ResourceSetInputProvider
      name: payment-app-inputs
```

**4\. Secrets Management Strategy**

We assume a "shared-nothing" architecture for security. Each Kubernetes cluster pairs with a dedicated Vault (or isolated path).

#### **The Workflow**

1. **Terraform (Infrastructure Layer):**  
   * Provisions the **Vault Instance** (or path) for the cluster.  
   * Provisions the **IAM Role/Service Account** allowing the cluster to auth with Vault.  
   * Deploys the SecretStore CRD (part of ESO) into the cluster, pointing to that specific Vault URL.  
2. **Vault (Data Layer):**  
   * Stores secrets at path: secret/data/payment-app/prod  
   * Key: db\_password \-\> Value: sUp3rS3cr3t  
3. **External Secrets Operator (Cluster Layer):**  
   * In Repo B (acme-fleet-config), we define an ExternalSecret:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-app-secrets # This matches the reference in the ResourceSet
spec:
  secretStoreRef:
    name: cluster-vault # Pointing to this cluster's unique Vault
    kind: ClusterSecretStore
  target:
    name: payment-app-secrets # The K8s Secret to create
  data:
    - secretKey: db_password
      remoteRef:
        key: secret/data/payment-app/prod
        property: db_password
```

4. **ResourceSet Integration:**  
   * The ResourceSet in Repo A simply expects a K8s Secret named payment-app-secrets to exist. It does not know or care about Vault. This keeps your Blueprint repo "secret-agnostic."

**5\. The CI/CD Lifecycle**

#### **Step 1: Code & Build (CI)**

* Developer commits code to Application Repo.  
* CI Pipeline builds container image v1.1.0.  
* CI Pipeline pushes image to Registry.

#### **Step 2: Deploy to Dev (CD \- Automated)**

* CI Pipeline runs a script to clone acme-fleet-config.  
* Script updates clusters/dev-use1-01/tenants/app/input.yaml changing image: v1.1.0.  
* Script commits and pushes to main.  
* **Flux in Dev Cluster:** Detects change, renders ResourceSet with new input, and upgrades the app.

#### **Step 3: Promote to Staging (CD \- Manual Gate)**

* Developer (or Pipeline) opens a Pull Request in acme-fleet-config.  
* Change: Copy input.yaml values from Dev to Staging folder.  
* **Code Owners:** SRE Team is tagged for review.  
* **Merge:** Upon merge, Flux in Staging Cluster reconciles.

#### **Step 4: Promote to Prod (CD \- High Compliance)**

* Developer opens PR to update clusters/prod-use1-01/tenants/app/input.yaml.  
* **Requirements:**  
  * Image tag must match the signed/tested version from Staging.  
  * Requires 2 SRE approvals + 1 Tech Lead approval (enforced by Branch Protection).  
* **Audit:** The Git commit log for acme-fleet-config becomes your compliance audit trail.

**6\. Summary of Responsibilities**

| Component | Managed By | Tooling | Content |
| :---- | :---- | :---- | :---- |
| **Infrastructure** | Terraform | Terraform Cloud/Atlantis | VPCs, K8s Clusters, Vault Instances, IAM Roles. |
| **Cluster Bootstrapping** | Terraform | Flux Provider | Installs Flux, OPA, ESO, and the initial GitRepository source pointing to Repo B. |
| **Platform Logic** | SRE Team | **Repo A** (Flux ResourceSet) | Helm Release templates, dependency graphs, sidecar injections. |
| **Env Configuration** | App Devs / SRE | **Repo B** (Flux InputProvider) | Image tags, replicas, feature flags. |
| **Secrets** | Security Team | Vault + ESO | Database creds, API keys. |

