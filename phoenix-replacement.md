# Phoenix Cluster Replacement Guide

**Summary:** This document provides a step-by-step operational plan for provisioning a new Dev EKS cluster using a streamlined GitOps architecture, setting up required infrastructure components, and gradually migrating workloads off of the existing Dev cluster, performing a blue-green retirement of the existing cluster.

---

## Phase 1 -- Preparation

### Step 1: Provision new EKS Cluster

Provision the new Firebird cluster using our existing Terraform processes in the `infrastructure` repo. The cluster will be deployed into our existing VPC and subnets.

### Step 2: Setup `provisioning` Repository

```plaintext
/environments
  /dev
    /firebird
    /phoenix
  /prod
    /hydra
    /kraken
  /sre
    /kenny
    /kenny-g
/resources
  /aws-load-balancer
  /cert-manager
  /elk
  /external-secrets
  /flux
  /grafana
  /strimzi
  /traffic-gateway
  /vault
  /weapm
  /wectl
```

### Step 3: Bootstrap Flux Operator

### Step 4: Verify Cluster Readiness

### Step 5: Collect Migration Data

| Resource | Description | Tool/Command |
| :--- | :--- | :--- |
| **VPC & Subnet IDs** | Verify both clusters share the same networking layer | `aws eks describe-cluster --name <name>` |
| **IAM OIDC Provider** | Confirm it's configured for both clusters | `eksctl utils associate-iam-oidc-provider` |
| **EBS Volume IDs** | Map each StatefulSet Pod to its AWS Volume ID | See command below |
| **DNS Records** | Document all Route 53 records that need updating | Route 53 Console |
| **Get EBS Volume mappings:** | | |

## Phase 2 -- Per-Service Migration to Firebird

This phase is performed iteratively for each service. Select one service at a time and follow the appropriate workflow based on whether it is stateless or stateful.

---

### Stateless Service Migration

Follow these steps for Deployments and other stateless workloads.

#### Step 6a: Deploy Service to Firebird

Deploy the service to Firebird via Flux. Validate health checks confirm the service is running and the AWS Load Balancer Controller has created healthy Target Groups.

#### Step 7a: Setup Route 53 Weighted Records

Configure weighted routing with all traffic initially going to Phoenix:

| Record | Target | Weight |
| :--- | :--- | :--- |
| `<service>.example.com` | Phoenix Cluster LB | 100 |
| `<service>.example.com` | Firebird Cluster LB | 0 |

#### Step 8a: Gradual Traffic Shift

Shift traffic incrementally while monitoring for errors:

| Stage | Phoenix Weight | Firebird Weight | Duration |
| :--- | :--- | :--- | :--- |
| **A** | 90 | 10 | Monitor 15 min |
| **B** | 50 | 50 | Monitor 15 min |
| **C** | 0 | 100 | Full cutover |

#### Step 9a: Decommission Service on Phoenix

Once the Phoenix weight has been **0 for a predefined bake time** (e.g., 24-48 hours):

1. Remove the Phoenix Route 53 weighted record
2. Scale down or delete the Deployment on Phoenix
3. Clean up associated resources (Services, ConfigMaps, etc.)

#### Step 10a: Verification (Stateless)

- [ ] Service responds correctly on Firebird
- [ ] No traffic hitting Phoenix (verify via logs/metrics)
- [ ] Alerts and monitoring are healthy

---

### Stateful Service Migration (Pod-by-Pod)

Follow these steps for StatefulSets. Pods are migrated **one at a time** to maintain availability and respect StatefulSet ordering constraints.

#### Step 6b: Prepare for StatefulSet Migration

1. **Document the StatefulSet topology:** Note the number of replicas and pod names (e.g., `redis-0`, `redis-1`, `redis-2`)
2. **Secure all volumes** by setting the reclaim policy to Retain:

```bash
# For each PV associated with the StatefulSet
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

#### Step 7b: Migrate Pod (Repeat for Each Pod)

Perform the following for each pod, starting with the highest ordinal (e.g., `redis-2` → `redis-1` → `redis-0`):

**On Phoenix — Scale down the target pod:**

```bash
# Reduce replica count by 1 to remove the highest-ordinal pod
kubectl scale sts <sts-name> --replicas=<current-replicas - 1>
```

**On Phoenix — Release the PVC:**

```bash
# Delete the PVC (volume is retained due to Retain policy)
kubectl delete pvc <pvc-name>  # e.g., data-redis-2
```

**On Firebird — Create a static PV pointing to the EBS volume:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>  # e.g., data-redis-2-pv
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi  # Must match original size
  awsElasticBlockStore:
    volumeID: <aws-volume-id>
    fsType: ext4
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ebs-sc
```

```bash
kubectl apply -f pv-<pod-name>.yaml
```

**On Firebird — Create a standalone Pod with matching PVC:**

Since StatefulSet ordering constraints prevent running pods across both clusters simultaneously, deploy the pod as a standalone Pod (not managed by a StatefulSet):

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-redis-2
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
  volumeName: data-redis-2-pv  # Bind to the static PV
---
apiVersion: v1
kind: Pod
metadata:
  name: redis-2
  labels:
    app: redis
    statefulset.kubernetes.io/pod-name: redis-2  # Preserve identity
spec:
  containers:
    - name: redis
      image: redis:7
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-redis-2
```

```bash
kubectl apply -f standalone-pod-<ordinal>.yaml
```

**Verify the pod is running and healthy before proceeding to the next pod.**

#### Step 8b: Finalize StatefulSet on Firebird

Once all pods have been migrated as standalone Pods:

1. **On Phoenix — Delete the StatefulSet** (pods are already scaled to 0):

```bash
kubectl delete sts <sts-name>
```

2. **On Firebird — Create the StatefulSet** with `replicas: 0` initially:

```bash
kubectl apply -f statefulset.yaml  # with replicas: 0
```

3. **On Firebird — Delete the standalone Pods** (PVCs remain):

```bash
kubectl delete pod redis-0 redis-1 redis-2
```

4. **On Firebird — Scale the StatefulSet** to adopt the existing PVCs:

```bash
kubectl scale sts <sts-name> --replicas=<original-count>
```

The StatefulSet will create new pods that bind to the existing PVCs, taking ownership of the migrated volumes.

#### Step 9b: Update Internal DNS

Update any internal CNAME or headless service DNS records to point to Firebird:

```bash
aws route53 change-resource-record-sets --hosted-zone-id <ID> --change-batch file://dns-update.json
```

#### Step 10b: Verification (Stateful)

- [ ] `kubectl get pods` shows all StatefulSet pods Running on Firebird
- [ ] Application logs confirm successful data access
- [ ] AWS Console shows EBS volumes "In-Use" by Firebird nodes
- [ ] Read/Write smoke tests pass
- [ ] Dependent services can connect to the new endpoints

---

### Troubleshooting: Multi-Attach Error

If a pod on Firebird fails to start with a **Multi-Attach error**:
- The EBS volume lock hasn't timed out
- Phoenix's node hasn't fully detached the volume

**Resolution:** Manually detach the volume via AWS CLI:

```bash
aws ec2 detach-volume --volume-id <volume-id> --force
```

## Phase 3 -- Decommission Phoenix

### Step 20: Phoenix Cluster Decommissioning

1. **Remove Phoenix from Route 53** (delete the weight-0 records)
2. **Scale down Phoenix workloads**
3. **Terminate Phoenix cluster infrastructure** via Terraform
4. **Clean up unused resources** (old IAM roles, security groups, etc.)
