# Part 092: Kubernetes เบื้องต้นสำหรับ Django

> **ขั้นตอนที่ 911-920** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 911: Kubernetes คืออะไร

### 911.1 แนวคิดพื้นฐาน

**Kubernetes (K8s)** คือ container orchestration platform ที่จัดการ containers จำนวนมากอัตโนมัติ

| Object | หน้าที่ |
|--------|----------|
| **Pod** | unit เล็กที่สุด — หนึ่งหรือหลาย containers |
| **Deployment** | จัดการ Pods + rolling updates + rollback |
| **Service** | Expose Pods ด้วย stable IP/DNS |
| **Ingress** | HTTP routing + TLS termination |
| **ConfigMap** | configuration data (non-secret) |
| **Secret** | sensitive data (encrypted) |
| **PersistentVolume** | storage ที่ outlive Pods |
| **HPA** | Auto-scale Pods ตาม CPU/memory |

### 911.2 ติดตั้งสำหรับ Development

```bash
# Minikube (local K8s)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

kubectl cluster-info
kubectl get nodes
```

---

## ขั้นตอนที่ 912: Namespace และ Labels

### 912.1 จัดกลุ่ม Resources

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    environment: production
```

```bash
kubectl apply -f k8s/namespace.yaml
kubectl config set-context --current --namespace=myapp
```

---

## ขั้นตอนที่ 913: ConfigMap และ Secret

### 913.1 ConfigMap สำหรับ Non-secret Config

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  DJANGO_SETTINGS_MODULE: "config.settings.production"
  DEBUG: "False"
  ALLOWED_HOSTS: "example.com,www.example.com"
  STATIC_ROOT: "/app/staticfiles"
  REDIS_URL: "redis://redis-service:6379/0"
```

### 913.2 Secret สำหรับ Sensitive Data

```bash
# สร้าง Secret จาก literal values
kubectl create secret generic myapp-secrets \
  --from-literal=SECRET_KEY='your-very-long-secret-key' \
  --from-literal=DATABASE_URL='postgres://user:pass@postgres-service:5432/myapp' \
  --namespace=myapp
```

---

## ขั้นตอนที่ 914: Deployment สำหรับ Django

### 914.1 k8s/deployment-web.yaml

```yaml
# k8s/deployment-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-web
  namespace: myapp
  labels:
    app: myapp
    component: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      component: web
  template:
    metadata:
      labels:
        app: myapp
        component: web
    spec:
      containers:
        - name: web
          image: ghcr.io/username/myapp:latest
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
      initContainers:
        - name: migrate
          image: ghcr.io/username/myapp:latest
          command: ["python", "manage.py", "migrate", "--noinput"]
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
```

---

## ขั้นตอนที่ 915: Service

### 915.1 k8s/service.yaml

```yaml
# k8s/service-web.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-web-service
  namespace: myapp
spec:
  selector:
    app: myapp
    component: web
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP  # ใช้ Ingress แทนการ expose โดยตรง
```

---

## ขั้นตอนที่ 916: Ingress

### 916.1 k8s/ingress.yaml

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  tls:
    - hosts:
        - example.com
        - www.example.com
      secretName: myapp-tls
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-web-service
                port:
                  number: 80
```

---

## ขั้นตอนที่ 917: Celery Deployment

### 917.1 k8s/deployment-celery.yaml

```yaml
# k8s/deployment-celery.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-celery-worker
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      component: celery-worker
  template:
    metadata:
      labels:
        app: myapp
        component: celery-worker
    spec:
      containers:
        - name: celery-worker
          image: ghcr.io/username/myapp:latest
          command: ["celery", "-A", "config", "worker", "-l", "warning", "-c", "4"]
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

---

## ขั้นตอนที่ 918: Horizontal Pod Autoscaler (HPA)

### 918.1 Auto-scale ตาม CPU

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-web-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
kubectl get hpa -n myapp
kubectl describe hpa myapp-web-hpa -n myapp
```

---

## ขั้นตอนที่ 919: คำสั่ง kubectl ที่ใช้บ่อย

### 919.1 Deploy และ Manage

```bash
# Apply manifests
kubectl apply -f k8s/

# ดู resources ทั้งหมด
kubectl get all -n myapp

# ดู pods
kubectl get pods -n myapp
kubectl describe pod <pod-name> -n myapp

# ดู logs
kubectl logs -f deployment/myapp-web -n myapp
kubectl logs <pod-name> -n myapp --previous  # logs จาก pod ที่ crash

# Execute command ใน pod
kubectl exec -it deployment/myapp-web -n myapp -- python manage.py shell

# Rolling update
kubectl set image deployment/myapp-web web=ghcr.io/username/myapp:v2.0.0 -n myapp
kubectl rollout status deployment/myapp-web -n myapp

# Rollback
kubectl rollout undo deployment/myapp-web -n myapp

# Scale manually
kubectl scale deployment myapp-web --replicas=5 -n myapp

# Port forwarding (debug)
kubectl port-forward service/myapp-web-service 8080:80 -n myapp
```

---

## ขั้นตอนที่ 920: สรุปและแบบฝึกหัด

### 920.1 K8s Architecture สำหรับ Django

```
[Ingress (Nginx)] → [Service: web]
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
      [Pod: web 1]         [Pod: web 2]
              │
     ┌────────┴────────┐
     ▼                 ▼
[Service: postgres] [Service: redis]
     │
[PVC: postgres-data]

[Deployment: celery-worker] → [Service: redis]
```

### 920.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: Deploy Django + PostgreSQL + Redis บน Minikube โดยใช้ Deployment + Service + ConfigMap + Secret ให้ทำงานได้จริง

**แบบฝึกหัดที่ 2**: สร้าง HPA ที่ auto-scale Django web pods เมื่อ CPU > 70% ทดสอบด้วย load testing

**แบบฝึกหัดที่ 3**: สร้าง `k8s/` directory ที่มี manifests ครบสำหรับ production stack พร้อม Ingress + TLS
