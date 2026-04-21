

## 🔴 1. Enable OIDC

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster roboshop \
  --region us-east-1 \
  --approve
```

---

## 🔴 3. Create IAM Service Account (IRSA)

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

## 🔴 4. Get ROLE ARN

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

## 🔴 5. Install Add-on WITH ROLE (CRITICAL STEP)

```bash
aws eks create-addon \
  --cluster-name roboshop \
  --region us-east-1 \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn <PASTE_ROLE_ARN>
```

---

## 🔴 6. Verify (must be 6/6)

```bash
kubectl get pods -n kube-system | grep ebs
```

👉 Expected:

```
ebs-csi-controller   6/6 Running
ebs-csi-node         Running
```

---

## 🔴 7. Restart your workload

```bash
kubectl delete pod -n roboshop --all
```

---

## 🔴 8. Final check

```bash
kubectl get pvc -n roboshop
kubectl get pods -n roboshop
```

👉 Expected:

* PVC → **Bound**
* Pods → **Running**

---

