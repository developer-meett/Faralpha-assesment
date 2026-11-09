# Kubernetes Deployment Assignment Submission Report
**Project Title**: Scalable Flask API with Persistent MongoDB on Kubernetes  
**Candidate**: Meet Suhagiya  
**GitHub Repository**: [https://github.com/developer-meett/Faralpha-assesment](https://github.com/developer-meett/Faralpha-assesment)  
**Cluster Environment**: Minikube (v1.37.0) on macOS (Docker Driver)

---

## 1. Executive Summary & Architecture Overview

This project implements a production-grade, highly available Kubernetes deployment of a containerized Python Flask REST API connected to an authenticated, persistent MongoDB database.

### Architecture Diagram

```
                                    +-----------------------------------------------+
                                    |                Minikube Node                  |
                                    |                                               |
                                    |  Namespace: flask-app                         |
                                    |                                               |
[ User / Postman / curl ]           |  +-----------------------------------------+  |
       |                            |  |        Flask Deployment                 |  |
       | NodePort: 30007            |  |  +------------------+ +---------------+ |  |
       +------------------------------>|  |  Flask Pod 1     | | Flask Pod 2   | |  |
       | (minikube service tunnel)  |  |  |  Port: 5001      | | Port: 5001    | |  |
       v                            |  |  +--------+---------+ +-------+-------+ |  |
+------------------------------+    |  |           |                   |         |  |
| flask-service (NodePort)     |--->|  +-----------+-------------------+---------+  |
| NodePort: 30007 / Port: 5001 |    |              |                   |            |
+------------------------------+    |              +---------+---------+            |
                                    |                        |                      |
                                    |                        v                      |
                                    |              ClusterIP: mongodb-service:27017 |
                                    |                        |                      |
                                    |                        v                      |
                                    |  +-----------------------------------------+  |
                                    |  |       MongoDB StatefulSet               |  |
                                    |  |  +-----------------------------------+  |  |
                                    |  |  |  Pod: mongodb-0                   |  |  |
                                    |  |  |  Port: 27017 (Auth Enabled)       |  |  |
                                    |  |  |  Mount: /data/db                  |  |  |
                                    |  |  +-----------------+-----------------+  |  |
                                    |  +--------------------+--------------------+  |
                                    |                       |                       |
                                    |                       v                       |
                                    |  +-----------------------------------------+  |
                                    |  | PersistentVolumeClaim (mongodb-pvc)     |  |
                                    |  +--------------------+--------------------+  |
                                    |                       |                       |
                                    |                       v                       |
                                    |  +-----------------------------------------+  |
                                    |  | PersistentVolume (mongodb-pv)           |  |
                                    |  | hostPath: /data/mongodb                 |  |
                                    |  +-----------------------------------------+  |
                                    +-----------------------------------------------+
```

---

## 2. Technical Explanations

### 2.1 Inter-Pod Communication & Kubernetes DNS Resolution

#### Why Pods Cannot Use `localhost`
In Kubernetes, every Pod has an isolated network namespace and its own IP address. Inside a Flask container, `localhost` (127.0.0.1) refers only to that specific container's loopback interface. Because MongoDB runs in a separate Pod (`mongodb-0`), attempting to reach `localhost:27017` from the Flask application fails.

#### How Service Discovery and CoreDNS Work
1. When `mongodb-service` is deployed, Kubernetes assigns it a stable virtual ClusterIP (`10.108.175.147`) and **CoreDNS** registers an authoritative internal record:
   ```
   mongodb-service.flask-app.svc.cluster.local -> 10.108.175.147
   ```
2. When Flask queries `mongodb-service:27017`, the container's resolver routes the DNS lookup to CoreDNS (`kube-dns`).
3. CoreDNS returns the Service's ClusterIP.
4. The Linux kernel's `kube-proxy` (via iptables/IPVS) redirects that traffic to the active backing Pod endpoint (`mongodb-0`).

---

### 2.2 Resource Requests and Limits in Kubernetes

Resource requests and limits control how CPU and memory are scheduled and bounded:

| Parameter | Configured Value | Operational Purpose |
|---|---|---|
| **CPU Request** | `200m` (0.2 core) | Minimum guaranteed CPU reserved by the Kubernetes scheduler when placing Pods on nodes. |
| **CPU Limit** | `500m` (0.5 core) | Hard ceiling enforced by Linux CFS (Completely Fair Scheduler) quotas. Prevents CPU starvation. |
| **Memory Request** | `250Mi` | Guaranteed physical RAM required to schedule the Pod. |
| **Memory Limit** | `500Mi` | Hard boundary. If container memory exceeds this limit, the Linux kernel terminates it with `OOMKilled`. |

#### Autoscaling (HPA) Relationship
The Horizontal Pod Autoscaler calculates utilization using the formula:
$$\text{Current CPU \%} = \frac{\text{Actual CPU Usage}}{\text{CPU Request}} \times 100\%$$
- Without declaring `resources.requests.cpu`, HPA cannot calculate a percentage, causing it to fail with an `<unknown>` status.
- Setting `requests.cpu: 200m` provides a reliable baseline for HPA to scale Pods between 2 and 5 replicas when average utilization exceeds 70%.

---

## 3. Design Choices & Alternatives Considered

| Decision | Selected Design | Alternatives Considered | Why Alternative Was Not Chosen |
|---|---|---|---|
| **Workload Controller for MongoDB** | **StatefulSet** (`mongodb-0`) | Deployment | Deployments assign random hashes to Pod names and do not guarantee persistent storage attachment order. StatefulSets guarantee stable identity and deterministic volume binding. |
| **MongoDB Networking** | **ClusterIP** Service | NodePort / LoadBalancer | The database must not be exposed to the public internet or host network. ClusterIP restricts access strictly to inside the Kubernetes cluster. |
| **Flask External Access** | **NodePort** (Port 30007) | Ingress Controller | NodePort is native to Minikube, requires zero third-party ingress controllers, and directly maps port 30007 for local evaluation. |
| **Database Credentials** | **Kubernetes Secret** (`mongodb-secret`) | ConfigMap / Plaintext | Credentials must never be committed in application source code or ConfigMaps. Secrets decouple sensitive data from container images. |
| **Container Base Image** | `python:3.9-slim` | `python:3.9` (full) or `alpine` | Full Python image is >900MB with unnecessary build tools. Alpine frequently suffers from wheel compilation delays with C-extensions. `slim` offers a minimal ~219MB footprint. |

---

## 4. Application Source Code & Dockerfile

### 4.1 Application Code: `app/app.py`
```python
from flask import Flask, request, jsonify  # Import necessary modules from Flask
from pymongo import MongoClient  # Import MongoClient to interact with MongoDB
from datetime import datetime  # Import datetime to get the current time
import os  # Import os to access environment variables

# Initialize the Flask application
app = Flask(__name__)

# Set up the MongoDB client
# The MongoDB URI is fetched from the environment variable 'MONGODB_URI'
# If not found, it defaults to 'mongodb://localhost:27017/'
client = MongoClient(os.environ.get("MONGODB_URI", "mongodb://localhost:27017/"))

# Connect to the database named 'flask_db'
db = client.flask_db

# Connect to the collection named 'data' within the 'flask_db' database
collection = db.data


# Define the route for the root URL
@app.route('/')
def index():
    # Return a welcome message with the current time
    return f"Welcome to the Flask app! The current time is: {datetime.now()}"


# Define the route for the '/data' endpoint
# This endpoint supports both GET and POST methods
@app.route('/data', methods=['GET', 'POST'])
def data():
    if request.method == 'POST':
        # If the request method is POST, get the JSON data from the request
        data = request.get_json()
        # Insert the data into the 'data' collection in MongoDB
        collection.insert_one(data)
        # Return a success message with status code 201 (Created)
        return jsonify({"status": "Data inserted"}), 201

    elif request.method == 'GET':
        # If the request method is GET, retrieve all documents from the 'data' collection
        # Convert the documents to a list, excluding the '_id' field
        data = list(collection.find({}, {"_id": 0}))
        # Return the data as a JSON response with status code 200 (OK)
        return jsonify(data), 200


# Run the Flask application
# The application will listen on all available IP addresses (0.0.0.0) and port 5001
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
```

### 4.2 Python Dependencies: `app/requirements.txt`
```text
Flask==2.0.2
pymongo==3.12.0
Werkzeug==2.0.3
```

### 4.3 Containerfile: `app/Dockerfile`
```dockerfile
# Use an official lightweight Python image
FROM python:3.9-slim

# Set working directory inside container
WORKDIR /app

# Copy dependency definition first to leverage Docker layer caching
COPY requirements.txt .

# Install dependencies without cache to minimize image size
RUN pip install --no-cache-dir -r requirements.txt

# Copy application source code
COPY app.py .

# Expose the application listening port
EXPOSE 5001

# Run the Flask web application
CMD ["python", "app.py"]
```

---

### 4.4 Docker Image Build & Registry Instructions
To build and push the container image:
```bash
# 1. Build locally
docker build -t flask-mongodb-app:latest ./app

# 2. Option A: Load directly into Minikube cache (Recommended for local evaluation)
minikube image load flask-mongodb-app:latest

# 3. Option B: Push to Docker Hub container registry
docker tag flask-mongodb-app:latest <your-dockerhub-username>/flask-mongodb-app:latest
docker push <your-dockerhub-username>/flask-mongodb-app:latest
```

---

![Screenshot 2 - Minikube Image](screenshots/screenshot_2_minikube_image.png)

---

## 5. Kubernetes Declarative Manifests

### 5.1 Namespace: `kubernetes/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: flask-app
  labels:
    name: flask-app
```

### 5.2 Secrets: `kubernetes/secrets.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: flask-app
type: Opaque
stringData:
  MONGO_INITDB_ROOT_USERNAME: "admin"
  MONGO_INITDB_ROOT_PASSWORD: "password123"
  MONGODB_URI: "mongodb://admin:password123@mongodb-service:27017/flask_db?authSource=admin"
```

### 5.3 PersistentVolume: `kubernetes/mongodb-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/mongodb"
```

### 5.4 PersistentVolumeClaim: `kubernetes/mongodb-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
  namespace: flask-app
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### 5.5 MongoDB Service (ClusterIP): `kubernetes/mongodb-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: flask-app
spec:
  type: ClusterIP
  ports:
    - port: 27017
      targetPort: 27017
  selector:
    app: mongodb
```

### 5.6 MongoDB StatefulSet: `kubernetes/mongodb-statefulset.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: flask-app
spec:
  serviceName: "mongodb-service"
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:latest
          ports:
            - containerPort: 27017
              name: mongodb
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
          volumeMounts:
            - name: mongodb-storage
              mountPath: /data/db
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            tcpSocket:
              port: 27017
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            tcpSocket:
              port: 27017
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: mongodb-storage
          persistentVolumeClaim:
            claimName: mongodb-pvc
```

### 5.7 Flask Deployment: `kubernetes/flask-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
  namespace: flask-app
  labels:
    app: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-app
          image: flask-mongodb-app:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5001
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: mongodb-secret
                  key: MONGODB_URI
          resources:
            requests:
              cpu: 200m
              memory: 250Mi
            limits:
              cpu: 500m
              memory: 500Mi
          livenessProbe:
            httpGet:
              path: /
              port: 5001
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 5001
            initialDelaySeconds: 5
            periodSeconds: 5
```

### 5.8 Flask Service (NodePort): `kubernetes/flask-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
  namespace: flask-app
spec:
  type: NodePort
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 5001
      targetPort: 5001
      nodePort: 30007
```

### 5.9 HorizontalPodAutoscaler: `kubernetes/flask-hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: flask-hpa
  namespace: flask-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-deployment
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

![Screenshot 3 - Manifests Listing](screenshots/screenshot_3_manifests_listing.png)

---

## 6. Verification Scenarios & Testing Evidence

### 6.1 Cluster Deployment Status Verification

![Screenshot 4 - Cluster Overview](screenshots/screenshot_4_cluster_overview.png)

---

### 6.2 DNS Resolution Verification

![Screenshot 5 - DNS Resolution](screenshots/screenshot_5_dns_resolution.png)

---

### 6.3 Resource Requests and Limits Verification

![Screenshot 6 - Resource Limits](screenshots/screenshot_6_resource_limits.png)

---

### 6.4 API Endpoints & MongoDB CRUD Testing

![Screenshot 7 - API CRUD Testing](screenshots/screenshot_7_api_testing.png)

---

### 6.5 Data Persistence & Self-Healing Testing

![Screenshot 8 - Data Persistence](screenshots/screenshot_8_data_persistence.png)

---

### 6.6 Cookie Point: High-Traffic Autoscaling (HPA) Simulation

#### Simulation Procedure:
1. In **Terminal 1**, monitor HPA and Pod replicas:
   ```bash
   kubectl get hpa,pods -n flask-app -w
   ```
2. In **Terminal 2**, launch high-traffic load generator:
   ```bash
   kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -n flask-app -- /bin/sh -c "while true; do wget -q -O- http://flask-service:5001/; done"
   ```
3. Watch **Terminal 1** as CPU utilization spikes past 70% and Kubernetes triggers scaling from 2 replicas to 3, 4, or 5 replicas.

![Screenshot 9 - HPA Autoscaling](screenshots/screenshot_9_hpa_autoscaling.png)

---

### 6.7 Project Documentation & Git Submission

![Screenshot 10 - Git Submission](screenshots/screenshot_10_git_submission.png)

---

## 7. Submission Checklist Verification

- [x] Application source code included in `app/app.py`
- [x] Dependencies pinned in `app/requirements.txt`
- [x] Optimized Dockerfile included in `app/Dockerfile`
- [x] All 9 Kubernetes YAML manifests included in `kubernetes/`
- [x] Inter-pod DNS resolution explained and validated via CoreDNS
- [x] Resource requests and limits documented and enforced
- [x] Design choices & architectural alternatives detailed
- [x] API CRUD endpoints (`GET /`, `POST /data`, `GET /data`) validated
- [x] StatefulSet & PV/PVC data persistence confirmed
- [x] High-traffic HPA autoscaling simulated and verified
- [x] Code pushed to remote repository: `https://github.com/developer-meett/Faralpha-assesment`
