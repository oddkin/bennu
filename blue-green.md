# Blue-Green Deployment Process

**Summary:** This document outlines the operational plan for migrating workloads between two EKS clusters (**Cluster A** and **Cluster B**) within the same VPC and subnets. It specifically addresses the "EBS volume hand-off" required for StatefulSets and the DNS-based traffic shifting using Route 53.

**1. Pre-Migration Prerequisites**

Before attempting the cutover, the following data and infrastructure must be ready.

### **A. Data Collection Worksheet**

Collect these values for both clusters. These will be used in your automation scripts or manual kubectl commands.

| Resource | Description | Example / Tool |
| :---- | :---- | :---- |
| **VPC & Subnet IDs** | Ensure both clusters share the same networking layer. | `aws eks describe-cluster --name <name>` |
| **IAM OIDC Provider** | Required for the AWS Load Balancer Controller. | `eksctl utils associate-iam-oidc-provider` |
| **EBS Volume IDs** | Map each StatefulSet Pod to its underlying AWS Volume ID. | `kubectl get pv -o custom-columns=NAME:.metadata.name,VOL_ID:.spec.awsElasticBlockStore.volumeID` |
| **DNS TTLs** | Lower TTLs on Route 53 records to **60 seconds** 48 hours before migration. | Route 53 Console |

### **B. Controller & Add-on Readiness**

* **AWS Load Balancer Controller:** Must be installed in **Cluster B**.  
  * *Tip:* Use the ip target type for better performance and direct pod routing.  
* **EBS CSI Driver:** Ensure the same version of the EBS CSI driver is running in both clusters to avoid filesystem mount incompatibilities.  
* **Security Groups:** Verify that the Security Group for Cluster B's nodes/pods allows inbound traffic from the Load Balancer subnets.

**2\. Phase 1: Stateless Service Preparation**

Stateless services are "Warm-Standby" in Cluster B.

1. **Deploy Application to Cluster B:** Apply your manifests (Deployments, Services, Ingress).  
2. **Validate Health Checks:** Ensure the AWS Load Balancer Controller has created the Target Groups and that targets are Healthy.  
3. **Setup Route 53 Weighted Records:**  
   * **Record 1:** api.example.com -> Cluster A LB (Weight: 100)  
   * **Record 2:** api.example.com -> Cluster B LB (Weight: 0)

**3\. Phase 2: The "Stateful Hand-off" Runbook**

This process "rolls" a StatefulSet (STS) by detaching the EBS volume from Cluster A and attaching it to Cluster B.

### **Step 1: Secure the Volume in Cluster A**

Ensure that deleting the PVC in K8s does **not** delete the actual volume in AWS.

```bash
# Set the reclaim policy to Retain
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### **Step 2: Stop the Service**

Scale the StatefulSet to zero to flush data and release the AWS volume lock.

```bash
kubectl scale sts <sts-name> --replicas=0
```

### **Step 3: Clear the Claim**

Delete the PVC in Cluster A. Because of the "Retain" policy, the EBS volume remains in AWS, but the PV in Cluster A will now show a status of Released.

```bash
kubectl delete pvc <pvc-name>
```

### **Step 4: Re-register Volume in Cluster B**

You must create a "Static" PV in Cluster B that points to the specific AWS volumeID.

**PV Manifest for Cluster B (pv-migration.yaml):**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-ebs-volume-01
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Gi # Must match original size
  awsElasticBlockStore:
    volumeID: <aws-volume-id-collected-earlier>
    fsType: ext4
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ebs-sc # Ensure this exists in Cluster B
```

```bash
kubectl apply -f pv-migration.yaml
```

### **Step 5: Start the Service in Cluster B**

Deploy the StatefulSet and PVC in Cluster B. Ensure the PVC is configured to bind to the PV you just created.

```bash
kubectl apply -f statefulset-b.yaml
```

**4\. Phase 3: Traffic Cutover & Validation**

Once the StatefulSet is up and verified in Cluster B, shift the traffic.

### **1. Shift Headless/Internal DNS**

If your STS uses a dedicated internal name (e.g., db.internal), update that CNAME record immediately.

```bash
# Example AWS CLI update for Route 53
aws route53 change-resource-record-sets --hosted-zone-id <ID> --change-batch file://dns-update.json
```

### **2. Shift Global Traffic (Stateless)**

Gradually adjust the weights on your main service record.

* **Step A:** Weight 50/50 (Monitor for errors).  
* **Step B:** Weight 0/100 (Full cutover).

### **3. Verification Checklist**

* [ ] Confirm kubectl get pods shows STS as Running.  
* [ ] Check application logs in Cluster B for database connection success.  
* [ ] Verify the AWS Console shows the EBS volume is "In-Use" by a Cluster B node.  
* [ ] Perform a "Read/Write" smoke test on the service.

---

**Warning:** If a pod in Cluster B fails to start with a Multi-Attach error, it means the AWS volume lock hasn't timed out or Cluster A's node hasn't fully detached. You may need to manually detach the volume via the AWS Console or CLI.
