# Full-Kubernetes-Concepts-Project


---
 
**📦 App**: Containerized Node.js app  
**🎯 Goal**: Deploy, scale, secure, and monitor this app using all major Kubernetes concepts  
**🔁 Flow**: Code → Docker → Kubernetes → CI/CD → Monitoring  

---

## 🧠 Overview of Concept Interactions

| Layer             | Concepts Involved                                                                 |
|------------------|------------------------------------------------------------------------------------|
| **Logical Grouping**   | `Namespaces`, `RBAC`, `NetworkPolicy`                                             |
| **App Definition**     | `Pod`, `Deployment`, `ReplicaSet`, `Resource Requests/Limits`, `Volumes`          |
| **Service Discovery**  | `Service (ClusterIP, NodePort, LoadBalancer)`, `Ingress`                          |
| **Configuration**      | `ConfigMap`, `Secrets`                                                           |
| **Availability**       | `HPA`, `LivenessProbe`, `ReadinessProbe`, `Rolling Updates`                      |
| **CI/CD Automation**   | `Jenkins + kubectl`                                                              |
| **Observability**      | `Prometheus`, `Grafana`, `Metrics API`, `HPA`                                    |

---

## 🧱 Step-by-Step with Interactions

---

### 🧩 **1. Namespace (Isolation)**  
Namespaces isolate environments (e.g., dev, staging, prod) to **avoid naming conflicts** and apply **scoped RBAC policies**.

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-ns
```

⏩ All other objects below will include:  
```yaml
metadata:
  namespace: demo-ns
```

---

### 🧩 **2. Configuration (ConfigMap & Secret)**  
Avoid hardcoding values. Pass runtime config to the app (like DB credentials or ENV settings).

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: node-config
data:
  APP_ENV: production
```

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: node-secret
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0  # "secret" in base64
```

🔁 **Interaction**: These are injected into Pods using `envFrom` in the Deployment spec.

---

### 🧩 **3. Persistent Storage (PVC)**  
Some apps need to store logs or files. PVCs are used to **request dynamic volumes**.

```yaml
# k8s/pvc.yaml
kind: PersistentVolumeClaim
metadata:
  name: node-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

🔁 **Interaction**: Used in the Pod by mounting it inside the container using `volumeMounts`.

---

### 🧩 **4. Deployment (Pods + ReplicaSet + Volumes + Probes)**  
This is where the app actually **runs** and is **replicated**.

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
        - name: node-container
          image: laly9999/node-app:1
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: node-config
            - secretRef:
                name: node-secret
          resources:
            limits:
              cpu: "500m"
              memory: "256Mi"
          volumeMounts:
            - name: app-storage
              mountPath: /usr/src/app/data
          livenessProbe:
            httpGet:
              path: /
              port: 3000
          readinessProbe:
            httpGet:
              path: /
              port: 3000
      volumes:
        - name: app-storage
          persistentVolumeClaim:
            claimName: node-pvc
```

🔁 **Interaction**:  
- `ConfigMap` and `Secret` configure the app  
- `PVC` stores app data  
- `Probes` ensure containers are healthy  
- `ReplicaSet` keeps 3 copies running  

---

### 🧩 **5. Services (Accessing the App)**  
#### Internal Access
```yaml
# ClusterIP
kind: Service
metadata:
  name: node-service
spec:
  selector:
    app: node-app
  ports:
    - port: 80
      targetPort: 3000
```

#### External Access
```yaml
# NodePort
spec:
  type: NodePort
  ports:
    - nodePort: 30080
```

#### Cloud Load Balancer
```yaml
# LoadBalancer
spec:
  type: LoadBalancer
```

🔁 **Interaction**:  
Services forward requests to Pods behind the scenes.

---

### 🧩 **6. Ingress (Domain Routing + TLS)**  
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-ingress
spec:
  rules:
    - host: node.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: node-service
                port:
                  number: 80
```

🔁 **Interaction**:  
Ingress routes external HTTP traffic to the `Service`, which routes to Pods.

---

### 🧩 **7. Horizontal Pod Autoscaler (HPA)**  
Dynamically scale pods based on CPU usage.

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: node-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: node-app
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

🔁 **Interaction**:  
Monitors Deployment → adjusts replicas → leverages `metrics-server`.

---

### 🧩 **8. RBAC (Role-Based Access Control)**  
Define who can do what. ServiceAccounts are used by apps or Jenkins to interact with the cluster.

```yaml
# serviceaccount.yaml
kind: ServiceAccount
metadata:
  name: node-sa

# rbac-role.yaml
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

# binding
kind: RoleBinding
subjects:
  - kind: ServiceAccount
    name: node-sa
roleRef:
  kind: Role
  name: node-role
```

🔁 **Interaction**:  
Jenkins can use `node-sa` to deploy or update apps securely.

---

### 🧩 **9. Network Policy (Restrict Traffic)**  
Allow only certain Pods or Namespaces to talk to others.

```yaml
# network-policy.yaml
kind: NetworkPolicy
metadata:
  name: allow-web
spec:
  podSelector:
    matchLabels:
      app: node-app
  ingress:
    - from:
        - podSelector: {}
```

🔁 **Interaction**:  
Secures `node-app` from other apps trying to talk to it unless allowed.

---

### 🧩 **10. CI/CD with Jenkins**  
**Jenkinsfile** automates the process:  
1. Clone repo  
2. Build image  
3. Push to DockerHub  
4. Update Deployment in Kubernetes  

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "laly9999/node-app:${BUILD_NUMBER}"
    NAMESPACE = "demo-ns"
  }
  stages {
    stage('Build & Push') {
      steps {
        sh 'docker build -t $IMAGE .'
        sh 'docker push $IMAGE'
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        sh 'kubectl set image deployment/node-app node-container=$IMAGE -n $NAMESPACE'
      }
    }
  }
}
```

🔁 **Interaction**:  
CI/CD interacts with the **Deployment**, **RBAC**, and **Image Registry**.

---

### 🧩 **11. Observability (Prometheus + Grafana)**  
Installed using Helm:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack
```

🔁 **Interaction**:  
HPA, CPU, Memory, and Pods metrics are pulled from the Prometheus metrics API.

---

## 🧪 Final Deployment Commands

```bash
# 1. Set up Namespace and configuration
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/pvc.yaml

# 2. Deploy application
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service-clusterip.yaml
kubectl apply -f k8s/ingress.yaml
kubectl apply -f k8s/hpa.yaml

# 3. Secure and monitor
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/rbac-role.yaml
kubectl apply -f k8s/network-policy.yaml
```

---

## 🏁 Summary of Interactions

| Concept            | Connected To                                                                 |
|--------------------|------------------------------------------------------------------------------|
| Deployment         | ConfigMap, Secret, PVC, HPA, RBAC, Probes, ReplicaSet                        |
| Service            | Ingress, Deployment                                                          |
| Ingress            | Service → Routing via Hostname                                               |
| Jenkins            | DockerHub, Deployment, RBAC, CI/CD                                           |
| HPA                | Metrics API, Deployment                                                      |
| Prometheus/Grafana | HPA, Cluster Metrics, Application Metrics                                    |
| NetworkPolicy      | Pod Security                                                                 |
| RBAC               | Jenkins or app ServiceAccount permissions                                    |

---

Would you like me to:
1. Generate a GitHub repo with all these files?
2. Package it as a Helm chart?
3. Convert this into a Medium blog with diagrams?

Let me know how you want to proceed!
