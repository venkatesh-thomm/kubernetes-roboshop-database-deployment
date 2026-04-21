
## 🔴 1. Enable OIDC

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster roboshop \
  --region us-east-1 \
  --approve
```

---

## 🔴 2. Create IAM Service Account (IRSA)

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster roboshop \
  --region us-east-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

---

## 🔴 3. Get ROLE ARN

```bash
eksctl get iamserviceaccount \
  --cluster roboshop \
  --namespace kube-system
```

👉 Copy the ARN for:

```
ebs-csi-controller-sa
```

---

## 🔴 4. Install Add-on WITH ROLE (CRITICAL STEP)

```bash
aws eks create-addon \
  --cluster-name roboshop \
  --region us-east-1 \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn <PASTE_ROLE_ARN>
```

---

## 🔴 5. Verify (must be 6/6)

```bash
kubectl get pods -n kube-system | grep ebs
```

👉 Expected:

```
ebs-csi-controller   6/6 Running
ebs-csi-node         Running
```

---

## 🔴 6. Restart your workload

```bash
kubectl delete pod -n roboshop --all
```

---

## 🔴 7. Final check

```bash
kubectl get pvc -n roboshop
kubectl get pods -n roboshop
```

👉 Expected:

* PVC → **Bound**
* Pods → **Running**

---

## Here’s a **practical, real-world troubleshooting checklist** for **PV, PVC, and StorageClass issues** specifically impacting **MySQL & MongoDB workloads in Kubernetes**—focused on what actually breaks in production.

---

# 🔍 1. Start With the Basics (Always)

### ✅ Check PVC status

```bash
kubectl get pvc -n <namespace>
```

* **Pending** → StorageClass / provisioning issue
* **Bound** → Move to pod/storage debugging
* **Lost** → PV deleted or corrupted

---

### ✅ Describe PVC (MOST IMPORTANT)

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Look for:

* ❌ `no persistent volumes available`
* ❌ `failed to provision volume`
* ❌ `waiting for first consumer`

---

### ✅ Check PV

```bash
kubectl get pv
kubectl describe pv <pv-name>
```

Key things:

* Status: `Available / Bound / Released / Failed`
* Reclaim policy: `Delete / Retain`
* Correct storage class?

---

### ✅ Check StorageClass

```bash
kubectl get sc
kubectl describe sc <sc-name>
```

Watch for:

* ❌ Wrong provisioner (e.g., EBS CSI missing)
* ❌ Parameters mismatch (gp2/gp3, zones)

---

# ⚠️ 2. Common Issues & Fixes

---

## 🚫 PVC stuck in Pending

### Causes:

* No matching PV
* StorageClass not working
* CSI driver missing

### Fix:

```bash
kubectl get sc
kubectl get pods -n kube-system | grep csi
```

👉 If using AWS → ensure EBS CSI driver installed
👉 Check IAM permissions

---

## 🔁 MySQL / MongoDB stuck in CrashLoopBackOff

### Check logs:

```bash
kubectl logs <pod>
```

### Typical errors:

### 🛑 MySQL

* `InnoDB: corruption detected`
* `Can't open file ibdata1`

### 🛑 MongoDB

* `WiredTiger error`
* `Permission denied`
* `Data files corrupted`

---

## 💥 Corrupted PVC Data (VERY COMMON)

### Symptoms:

* Pod restarts continuously
* DB fails to initialize

### Fix options:

### 🔹 Option 1: Delete PVC (DATA LOSS ⚠️)

```bash
kubectl delete pvc <pvc-name>
```

👉 Only if reclaim policy = Delete OR test env

---

### 🔹 Option 2: Clean volume manually

```bash
kubectl run debug --rm -it --image=busybox -- /bin/sh
```

Mount same PVC and:

```sh
rm -rf /data/*
```

---

### 🔹 Option 3: Restore from backup

* MySQL → dump restore
* MongoDB → mongodump restore

---

## 🔐 Permission Issues (MongoDB common)

### Error:

```
Permission denied
```

### Fix:

```yaml
securityContext:
  fsGroup: 1000
```

Or init container:

```bash
chown -R 1000:1000 /data
```

---

## 📦 Volume Mount Issues

Check:

```bash
kubectl describe pod <pod-name>
```

Look for:

* ❌ `MountVolume.SetUp failed`
* ❌ `Volume not found`

---

## ⚡ Multi-AZ / Node Scheduling Issue

### Error:

```
volume node affinity conflict
```

### Fix:

* Ensure pod runs in same AZ as volume
* Use:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

---

## 🔄 PVC not updating / stale data

👉 PVC cannot be “restarted”

### Workarounds:

* Restart pod:

```bash
kubectl delete pod <pod>
```

* Recreate PVC (if needed)

---

# 🧠 3. MySQL-Specific Troubleshooting

---

### 🔍 Check data directory

```bash
ls -l /var/lib/mysql
```

### Common fixes:

* Remove `ib_logfile*`
* Fix ownership:

```bash
chown -R mysql:mysql /var/lib/mysql
```

---

### ⚠️ Crash recovery mode

```bash
--innodb-force-recovery=1
```

Use only for recovery!

---

# 🍃 4. MongoDB-Specific Troubleshooting

---

### 🔍 Check DB path

```bash
ls -l /data/db
```

### Fix corruption:

```bash
mongod --repair
```

---

### 🔐 Fix permissions:

```bash
chown -R mongodb:mongodb /data/db
```

---

# 🔎 5. CSI / Cloud Storage Debugging (AWS EBS Example)

---

### Check CSI pods:

```bash
kubectl get pods -n kube-system | grep ebs
```

### Logs:

```bash
kubectl logs -n kube-system <ebs-csi-pod>
```

---

### Check volume in AWS:

* Volume attached?
* Correct AZ?

---

# 📊 6. End-to-End Debug Flow (REAL INTERVIEW ANSWER)

1. Check PVC status
2. Describe PVC → identify provisioning issue
3. Verify StorageClass and CSI driver
4. Check PV binding
5. Inspect pod events
6. Check container logs (MySQL/MongoDB errors)
7. Validate volume mount & permissions
8. Check for data corruption
9. Restore or recreate PVC if required

---

# 💡 Pro Tips (What Interviewers Expect)

* PVC issues = mostly **StorageClass / CSI**
* DB failures = mostly **corruption or permissions**
* Stateful apps → always use:

```yaml
volumeClaimTemplates
```

* Never delete PVC in production without backup

---

