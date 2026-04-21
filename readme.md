
# Roboshop Microservices - Kubernetes Deployment Guide

This repository contains **Kubernetes yml manifests** for all core Roboshop microservices:


Each microservice includes:
✔ ConfigMap  
✔ Deployment / StatefulSet  
✔ Service  
✔ Horizontal Pod Autoscaler (HPA) (where applicable)  
✔ Persistent Volume Claims (for DB services)

---

# 📦 1. User Service
Handles user registrations, logins, and profiles.

### Key Environment Variables
```
MONGO=true
REDIS_URL=redis://redis:6379
MONGO_URL=mongodb://mongodb:27017/users
```

### Ports
- Application: **8080**

### Dependencies
- MongoDB  
- Redis  

---

# 🛍 2. Catalogue Service
Manages product catalogue for the web application.

### ConfigMap
```
MONGO=true
MONGO_URL=mongodb://mongodb:27017/catalogue
```

### Ports
- Application: **8080**

### Dependencies
- MongoDB  

---

# 🛒 3. Cart Service
Manages user carts, product additions/removals.

### ConfigMap
```
REDIS_HOST=redis
CATALOGUE_HOST=catalogue
CATALOGUE_PORT=8080
```

### Ports
- Application: **8080**

### Dependencies
- Redis  
- Catalogue  

---

# 🍃 4. MongoDB (StatefulSet)
Stores catalogue and user data.

### Components
✔ ClusterIP Service  
✔ Headless Service  
✔ StatefulSet with PVC  

### Ports
- DB Port: **27017**

---

# 🐬 5. MySQL (StatefulSet)
Stores shipping and transactional data.

### ConfigMap Injected File
`my.cnf` tuning MySQL performance.

### Ports
- DB Port: **3306**

---

# 🐇 6. RabbitMQ (StatefulSet)
Handles message queues for the Payment and Shipping services.

### Environment Variables
```
RABBITMQ_DEFAULT_USER=roboshop
RABBITMQ_DEFAULT_PASS=roboshop123
```

### Ports
- AMQP Port: **5672**

---

# 🔥 7. Redis (StatefulSet)
Used for caching sessions & cart data.

### Ports
- Redis Port: **6379**

---

## Install drivers for EBS CSI 

```yaml
#Enable OIDC provider (REQUIRED FIRST)
eksctl utils associate-iam-oidc-provider \
  --cluster <cluster-name> \
  --approve

#Create IAM role + attach it to EBS CSI driver (IRSA)
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster <cluster-name> \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts

#Install / update EBS CSI addon
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster <cluster-name> \
  --service-account-role-arn arn:aws:iam::<account-id>:role/<role-name> \
  --force
```

# 🚀 Deployment Commands

### Create Namespace
```
kubectl create namespace roboshop

### Apply All ymls
kubectl apply -f namespace.yml
kubectl apply -f ebs-sc.yml

# Databases first
kubectl apply -f mongodb/manifest.yml
kubectl apply -f mysql/manifest.yml
kubectl apply -f redis/manifest.yml
kubectl apply -f rabbitmq/manifest.yml

# Wait for DBs
kubectl get pods -n roboshop

# Then apps
kubectl apply -f catalogue/manifest.yml
kubectl apply -f cart/manifest.yml
kubectl apply -f user/manifest.yml
kubectl apply -f frontend/manifest.yml


# Delete Pod
kubectl delete pod -n roboshop --all
```

---

# 🛠 Troubleshooting

### Check Pod Logs
```
kubectl logs -f <pod-name> -n roboshop
```

### Describe Pod
```
kubectl describe pod <pod-name> -n roboshop
```

### Test Internal Service Connectivity
```
kubectl run test --rm -it --image=busybox -- wget -O- user:8080/health
```

---

# 🧠 Notes
- DB StatefulSets require storage class: **roboshop-ebs**
- All application microservices run on port **8080**
- HPA uses CPU utilization for autoscaling
- Headless services enable stable DNS resolution for StatefulSets

---

