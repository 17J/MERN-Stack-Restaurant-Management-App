# Full-Stack Restaurant Management Application

A production-ready **MERN Stack** restaurant management application with comprehensive features for menu management, order processing, and user authentication, deployed on **Kubernetes**.

## 🏗️ Architecture

```
┌─────────────────┐
│   Frontend      │  React + Redux (Port 80)
│ (Nginx/Vite)    │  User Interface
└────────┬────────┘
         │
┌────────▼────────┐
│    Backend      │  Node.js + Express (Port 5000)
│   (Express)     │  REST API + JWT Auth
└────────┬────────┘
         │
┌────────▼────────┐
│    Database     │  MongoDB StatefulSet (Port 27017)
│   (MongoDB)     │  Persistent Storage + Replica Set
└─────────────────┘
```

**Three-Tier MERN Architecture:**

1. **Presentation Tier:** React.js with Redux state management and responsive UI
2. **Application Tier:** Node.js/Express.js REST API with JWT authentication
3. **Data Tier:** MongoDB StatefulSet with persistent volumes and replica set configuration

---

## 🛠️ Tech Stack

### Frontend

- **React.js** - UI library
- **Redux** - State management
- **React Router** - Client-side routing
- **Axios** - HTTP client
- **Styled Components** - CSS-in-JS styling
- **Vite** - Build tool and dev server
- **Nginx** - Production web server

### Backend

- **Node.js** - JavaScript runtime
- **Express.js** - Web framework
- **JWT** - Authentication
- **Mongoose** - MongoDB ODM

### Database

- **MongoDB 4.4** - NoSQL database
- **StatefulSet** - Kubernetes stateful workload

### DevOps

- **Docker** - Containerization
- **Kubernetes** - Container orchestration
- **Nginx Ingress** - Traffic routing

---

## ✨ Features

- 🍽️ **Menu Management** - Browse and manage restaurant menu items
- 🛒 **Order Processing** - Complete order workflow from cart to checkout
- 👤 **User Authentication** - Secure login/registration with JWT tokens
- 🔐 **Role-Based Access** - Admin and customer role separation
- 📱 **Responsive Design** - Mobile-first, works on all devices
- 🔄 **State Management** - Redux for predictable state updates
- 💾 **Persistent Storage** - MongoDB with Kubernetes persistent volumes
- 📊 **Health Monitoring** - Built-in health checks and readiness probes
- ⚡ **High Availability** - Multiple replicas with auto-scaling support
- 🔒 **Production Security** - Environment-based secrets and ConfigMaps

---

## 📋 Prerequisites

Before deploying, ensure you have:

### Required Tools

- **Kubernetes cluster** (v1.20+)
  - Local: Minikube, Kind, Docker Desktop
  - Cloud: GKE, EKS, AKS
- **kubectl** installed and configured (v1.20+)
- **Docker** installed (for building custom images)
- **Nginx Ingress Controller** installed

### Cluster Resources

- Minimum **6GB RAM** available
- **3 CPU cores** minimum
- **60GB+ storage** for MongoDB StatefulSet (3 replicas × 20GB each)
- **StorageClass** configured for dynamic provisioning

### Verify Prerequisites

```bash
# Check Kubernetes version
kubectl version --short

# Check cluster nodes and resources
kubectl get nodes
kubectl top nodes

# Verify StorageClass exists
kubectl get storageclass

# Check Ingress Controller (nginx)
kubectl get pods -n ingress-nginx

# If Ingress Controller not installed:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

---

## 🚀 Quick Start Deployment

### Step 1: Clone the Repository

```bash
git clone https://github.com/17J/Full-Stack-Restaurant-Management-App.git
cd Full-Stack-Restaurant-Management-App
```

### Step 2: Create Namespace

```bash
kubectl create namespace restaurant-app

# Set as default namespace (optional)
kubectl config set-context --current --namespace=restaurant-app
```

### Step 3: Configure Secrets

**Important:** Replace the default secrets with your own secure values before deploying to production.

**: Use provided secrets file**

```bash
kubectl apply -f k8s/secrets-configmap.yml -n restaurant-app
```

### Step 4: Deploy ConfigMap

The ConfigMap contains non-sensitive application configuration:

```bash
# Apply the ConfigMap
kubectl apply -f k8s/secrets-configmap.yml -n restaurant-app

# Verify ConfigMap
kubectl get configmap mern-app-config -n restaurant-app -o yaml
```

**ConfigMap contains:**

- `db-name`: foodmore
- `backend-port`: 5000
- `node-env`: development
- `jwt-expires-in`: 7d
- `mongo-uri`: mongodb://mongo-svc:27017/
- `frontend-api-url`: http://backend-service:5000/api

### Step 5: Deploy MongoDB StatefulSet

MongoDB is deployed as a StatefulSet with 3 replicas for high availability.

```bash
# Deploy MongoDB StatefulSet and Services
kubectl apply -f k8s/db-ds.yml -n restaurant-app

# Wait for MongoDB pods to be ready (may take 2-3 minutes)
kubectl wait --for=condition=ready pod -l app=mongo \
  -n restaurant-app --timeout=300s

# Verify MongoDB deployment
kubectl get statefulset mongo-db -n restaurant-app
kubectl get pods -l app=mongo -n restaurant-app
kubectl get pvc -n restaurant-app

# Check MongoDB logs
kubectl logs mongo-db-0 -n restaurant-app --tail=50
```

### Step 6: Deploy Backend API

```bash
# Deploy Node.js/Express backend
kubectl apply -f k8s/backend-deployment-service.yml -n restaurant-app

# Wait for backend pods to be ready
kubectl wait --for=condition=ready pod -l app=backend \
  -n restaurant-app --timeout=300s

# Verify backend deployment
kubectl get deployment backend-deployment -n restaurant-app
kubectl get pods -l app=backend -n restaurant-app

# Check backend logs
kubectl logs -l app=backend -n restaurant-app --tail=100

# Test backend health
kubectl exec -it $(kubectl get pod -l app=backend -n restaurant-app -o jsonpath='{.items[0].metadata.name}') \
  -n restaurant-app -- curl -s http://localhost:5000/
```

### Step 8: Deploy Frontend Application

```bash
# Deploy React frontend
kubectl apply -f k8s/frontend-deployment-service.yml -n restaurant-app

# Wait for frontend pods to be ready
kubectl wait --for=condition=ready pod -l app=frontend \
  -n restaurant-app --timeout=300s

# Verify frontend deployment
kubectl get deployment frontend-deployment -n restaurant-app
kubectl get pods -l app=frontend -n restaurant-app

# Check frontend logs
kubectl logs -l app=frontend -n restaurant-app --tail=50
```

### Step 9: Deploy Ingress

```bash
# Deploy Ingress for routing
kubectl apply -f k8s/ingress.yml -n restaurant-app

# Get Ingress details
kubectl get ingress restaurant-mern-ingress -n restaurant-app

# Describe Ingress for troubleshooting
kubectl describe ingress restaurant-mern-ingress -n restaurant-app
```

### Step 10: Verify Complete Deployment

```bash
# Check all resources
kubectl get all -n restaurant-app

# Check pod status with detailed info
kubectl get pods -n restaurant-app -o wide

# View resource usage
kubectl top pods -n restaurant-app

# Check persistent volumes
kubectl get pvc -n restaurant-app

# Check services and endpoints
kubectl get svc,endpoints -n restaurant-app

# View events for troubleshooting
kubectl get events -n restaurant-app --sort-by='.lastTimestamp'
```

---

## 🌐 Accessing the Application

### Option 1: Using LoadBalancer (Cloud Providers)

```bash
# Get external IP
kubectl get svc frontend-service -n restaurant-app

# Access application at:
# http://<EXTERNAL-IP>
```

### Option 2: Using Ingress (Recommended for Production)

```bash
# Get Ingress address
kubectl get ingress restaurant-mern-ingress -n restaurant-app

# If using Minikube:
minikube ip

# Add to /etc/hosts (Linux/Mac) or C:\Windows\System32\drivers\etc\hosts (Windows):
<INGRESS-IP> restaurant.local

# Access at: http://restaurant.local
```

### Option 3: Using NodePort (Development)

```bash
# Get NodePort
kubectl get svc frontend-service -n restaurant-app

# For Minikube:
minikube service frontend-service -n restaurant-app --url

# Access at: http://<NODE-IP>:<NODE-PORT>
```

### Option 4: Port Forwarding (Local Testing)

```bash
# Frontend
kubectl port-forward svc/frontend-service 8080:80 -n restaurant-app
# Access at: http://localhost:8080

# Backend API (for direct testing)
kubectl port-forward svc/backend-service 5000:5000 -n restaurant-app
# Access at: http://localhost:5000

# MongoDB (for database access)
kubectl port-forward svc/mongo-service 27017:27017 -n restaurant-app
# Connect with: mongodb://localhost:27017
```

---

## 🔧 Configuration

### Environment Variables

**Backend Environment Variables:**

```yaml
PORT: 5000 # Backend server port
NODE_ENV: production # Environment mode
MONGO_URI: mongodb://mongo-svc:27017/ # MongoDB connection string
DB_NAME: foodmore # Database name
JWT_SECRET: <secure-token> # JWT signing secret
JWT_EXPIRES_IN: 7d # Token expiration
```

**Frontend Environment Variables:**

```yaml
VITE_API_URL: http://backend-service:5000/api # Backend API endpoint
```

### Custom Configuration

To modify configuration values:

```bash
# Edit ConfigMap
kubectl edit configmap mern-app-config -n restaurant-app

# Or update the file and reapply
kubectl apply -f k8s/secrets-configmap.yml -n restaurant-app

# Restart pods to pick up new config
kubectl rollout restart deployment backend-deployment -n restaurant-app
kubectl rollout restart deployment frontend-deployment -n restaurant-app
```

## 🔍 Health Checks & Monitoring

### Health Check Endpoints

**Frontend Health:**

```bash
curl http://frontend-service/
```

**Backend Health:**

```bash
curl http://backend-service:5000/
```

**MongoDB Health:**

```bash
kubectl exec -it mongo-db-0 -n restaurant-app -- \
  mongo --eval "db.adminCommand('ping')"
```

### Monitoring Commands

```bash
# Watch pod status in real-time
kubectl get pods -n restaurant-app -w

# View resource usage
kubectl top pods -n restaurant-app
kubectl top nodes

# Check service endpoints
kubectl get endpoints -n restaurant-app

# View logs for all components
kubectl logs -l app=frontend -n restaurant-app --tail=50
kubectl logs -l app=backend -n restaurant-app --tail=50
kubectl logs -l app=mongo -n restaurant-app --tail=50

# Follow logs in real-time
kubectl logs -f deployment/backend-deployment -n restaurant-app
```

---

## 🧹 Cleanup

### Delete All Resources

```bash
# Delete entire namespace (recommended)
kubectl delete namespace restaurant-app

# This will delete:
# - All deployments and pods
# - All services
# - All persistent volume claims
# - All secrets and configmaps
# - Ingress rules
```

### Delete Individual Resources

```bash
# Delete deployments
kubectl delete deployment frontend-deployment -n restaurant-app
kubectl delete deployment backend-deployment -n restaurant-app
kubectl delete statefulset mongo-db -n restaurant-app

# Delete services
kubectl delete svc frontend-service backend-service mongo-service mongo-svc -n restaurant-app

# Delete ingress
kubectl delete ingress restaurant-mern-ingress -n restaurant-app

# Delete persistent volume claims (WARNING: This deletes data!)
kubectl delete pvc -n restaurant-app --all

# Delete secrets and configmaps
kubectl delete secret mern-app-secrets -n restaurant-app
kubectl delete configmap mern-app-config -n restaurant-app
```

---

## 📸 Application Screenshots

<p align="center">
    <img src="snapshot/restra.jpg" alt="Home Page" width="45%"/>
    <img src="snapshot/resta_menu.jpg" alt="Menu Page" width="45%"/>
</p>

<p align="center">
    <img src="snapshot/rest_login.jpg" alt="Login Page" width="45%"/>
    <img src="snapshot/resta_register.jpg" alt="Register Page" width="45%"/>
</p>

---

## 🤝 Contributing

Contributions are welcome! Please follow these guidelines:

### How to Contribute

1. Fork the repository
2. Create a feature branch
   ```bash
   git checkout -b feature/AmazingFeature
   ```
3. Commit your changes
   ```bash
   git commit -m 'Add some AmazingFeature'
   ```
4. Push to the branch
   ```bash
   git push origin feature/AmazingFeature
   ```
5. Open a Pull Request
