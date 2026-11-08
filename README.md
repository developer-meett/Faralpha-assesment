# Flask + MongoDB Kubernetes Deployment (Part 2)

A production-grade, highly available Kubernetes deployment of a Python Flask REST API connected to a persistent, authenticated MongoDB database running on a local Minikube cluster.

---

## 1. Architecture Overview

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
       | (minikube service / tunnel)|  |  |  Port: 5001      | | Port: 5001    | |  |
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

### Components
1. **Flask Deployment**: Manages stateless web replicas (default 2, scaled 2-5 via HPA).
2. **Flask NodePort Service**: Exposes the Flask application on port `30007` for local machine access.
3. **MongoDB StatefulSet**: Manages the stateful database instance with stable identity (`mongodb-0`).
4. **MongoDB ClusterIP Service**: Restricts database access strictly to internal cluster components via DNS.
5. **Persistent Storage (PV & PVC)**: Maps `/data/db` to local host storage, ensuring data survives Pod recreation.
6. **Kubernetes Secret (`mongodb-secret`)**: Stores root credentials and the connection URI safely.
7. **Horizontal Pod Autoscaler (HPA)**: Dynamically scales Flask Pods when average CPU exceeds 70%.

---

## 2. Prerequisites

- **macOS** (or Linux/Windows)
- **Docker Desktop** (running)
- **Minikube** (v1.30+)
- **kubectl** (v1.26+)

Verify installations:
```bash
minikube version
kubectl version --client
docker --version
```

---

## 3. Project Structure

```
flask-mongodb-k8s/
├── app/
│   ├── app.py                # Flask application code (reads MONGODB_URI from env)
│   ├── requirements.txt      # Python dependencies (Flask, pymongo, Werkzeug)
│   └── Dockerfile            # Multi-layer Dockerfile (python:3.9-slim)
├── kubernetes/
│   ├── namespace.yaml        # Dedicated 'flask-app' namespace
│   ├── secrets.yaml          # Sensitive database credentials & connection string
│   ├── mongodb-pv.yaml       # PersistentVolume with 1Gi capacity
│   ├── mongodb-pvc.yaml      # PersistentVolumeClaim requesting 1Gi
│   ├── mongodb-service.yaml  # ClusterIP service for internal MongoDB networking
│   ├── mongodb-statefulset.yaml # StatefulSet controller for MongoDB
│   ├── flask-deployment.yaml # Deployment controller for Flask with 2 replicas
│   ├── flask-service.yaml    # NodePort service exposing Flask on port 30007
│   └── flask-hpa.yaml        # Autoscaler (min: 2, max: 5, target CPU: 70%)
├── .gitignore
└── README.md
```

---

## 4. Step-by-Step Setup & Deployment

### Step 1: Start Minikube & Enable Metrics Server
```bash
minikube start --driver=docker
minikube addons enable metrics-server
```

### Step 2: Build the Container Image
Build the container image using the Docker daemon:
```bash
cd app
docker build -t flask-mongodb-app:latest .
cd ..
```

Load the image directly into Minikube's internal container registry:
```bash
minikube image load flask-mongodb-app:latest
```

*Alternative (Optional - Docker Hub):*
```bash
docker tag flask-mongodb-app:latest <your-dockerhub-username>/flask-mongodb-app:latest
docker push <your-dockerhub-username>/flask-mongodb-app:latest
```

### Step 3: Deploy All Kubernetes Manifests
Apply the manifests in logical dependency order:

```bash
# 1. Create isolated namespace
kubectl apply -f kubernetes/namespace.yaml

# 2. Deploy database secrets
kubectl apply -f kubernetes/secrets.yaml

# 3. Provision storage
kubectl apply -f kubernetes/mongodb-pv.yaml
kubectl apply -f kubernetes/mongodb-pvc.yaml

# 4. Deploy internal database service and StatefulSet
kubectl apply -f kubernetes/mongodb-service.yaml
kubectl apply -f kubernetes/mongodb-statefulset.yaml

# 5. Wait for MongoDB Pod to be ready
kubectl rollout status statefulset/mongodb -n flask-app --timeout=120s

# 6. Deploy Flask application & expose service
kubectl apply -f kubernetes/flask-deployment.yaml
kubectl apply -f kubernetes/flask-service.yaml

# 7. Apply Horizontal Pod Autoscaler
kubectl apply -f kubernetes/flask-hpa.yaml
```

Or apply all at once:
```bash
kubectl apply -f kubernetes/
```

### Step 4: Verify Deployment Status
```bash
# Check all resources in the namespace
kubectl get all,pv,pvc,secret,hpa -n flask-app
```

Output should show:
- `mongodb-0` Pod `Running` (1/1)
- 2 `flask-deployment` Pods `Running` (1/1)
- `mongodb-service` (`ClusterIP`)
- `flask-service` (`NodePort`)
- `mongodb-pvc` status `Bound` to `mongodb-pv`
- `flask-hpa` targeting 70% CPU

---

## 5. Accessing and Testing the Application

### Get Service URL
On Minikube with the Docker driver on macOS, obtain the tunnel/URL:
```bash
minikube service flask-service -n flask-app --url
```
Example output: `http://127.0.0.1:58432` or `http://192.168.49.2:30007`

Alternatively, run a tunnel in a separate terminal:
```bash
minikube tunnel
```

### API Testing

#### 1. Health & Welcome Route (`GET /`)
```bash
curl -i http://<SERVICE_URL>/
```
**Response (200 OK):**
```
Welcome to the Flask app! The current time is: 2026-11-08 11:35:00.123456
```

#### 2. Insert Data (`POST /data`)
```bash
curl -i -X POST http://<SERVICE_URL>/data \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "role": "DevOps Engineer", "batch": "2026"}'
```
**Response (201 Created):**
```json
{
  "status": "Data inserted"
}
```

Insert a second document:
```bash
curl -i -X POST http://<SERVICE_URL>/data \
  -H "Content-Type: application/json" \
  -d '{"name": "Bob", "role": "Cloud Architect", "batch": "2026"}'
```

#### 3. Retrieve Data (`GET /data`)
```bash
curl -i http://<SERVICE_URL>/data
```
**Response (200 OK):**
```json
[
  {
    "name": "Alice",
    "role": "DevOps Engineer",
    "batch": "2026"
  },
  {
    "name": "Bob",
    "role": "Cloud Architect",
    "batch": "2026"
  }
]
```

---

## 6. Technical Explanations & Design Decisions

### 6.1 Kubernetes DNS & Internal Service Discovery

#### Why Flask Cannot Use `localhost`
When applications run in containers managed by Kubernetes:
- Each Pod is assigned its own isolated network namespace and unique IP address.
- Inside a Flask Pod, `localhost` (127.0.0.1) refers exclusively to the loopback interface of that specific Flask container.
- MongoDB runs in a distinct container inside a separate Pod (`mongodb-0`). Connecting to `localhost:27017` from the Flask Pod would attempt to connect to a nonexistent database running in the same container.

#### How Service Discovery Works
Kubernetes provisions an internal DNS cluster add-on (**CoreDNS**). When a Kubernetes Service is created, CoreDNS registers an A-record:
```
<service-name>.<namespace>.svc.cluster.local
```

For our setup:
- Full DNS name: `mongodb-service.flask-app.svc.cluster.local`
- Short DNS name (within the `flask-app` namespace): `mongodb-service`

When Flask initializes `MongoClient("mongodb://admin:password123@mongodb-service:27017/flask_db?authSource=admin")`:
1. The container queries CoreDNS for `mongodb-service`.
2. CoreDNS resolves it to the Service's stable virtual IP (`ClusterIP`).
3. Kubernetes `kube-proxy` routes the connection to the backend Pod (`mongodb-0`).

---

### 6.2 Resource Requests vs Limits & HPA Integration

Resource management is defined under `resources` for every container:

| Setting | Value | Operational Purpose |
|---|---|---|
| **CPU Request** | `200m` (0.2 CPU core) | The guaranteed minimum capacity reserved by Kubernetes when scheduling Pods on nodes. |
| **CPU Limit** | `500m` (0.5 CPU core) | The hard upper ceiling. If the container attempts to exceed this, the Linux CFS quota throttles CPU cycles. |
| **Memory Request** | `250Mi` | Guaranteed physical memory allocated during scheduling. |
| **Memory Limit** | `500Mi` | Hard memory boundary. Exceeding this triggers an Out-Of-Memory (OOM) kill (`OOMKilled`). |

#### Relationship with HPA
The Horizontal Pod Autoscaler calculates utilization using the formula:
$$\text{Current Utilization \%} = \frac{\text{Current CPU Usage}}{\text{CPU Request}} \times 100\%$$

- **Crucial Rule**: Without a declared `resources.requests.cpu`, the HPA cannot compute a denominator. As a result, the HPA will remain in an `<unknown>` state and fail to trigger autoscaling events.
- **Metrics Server**: The Metrics Server scrapes resource metrics from each node's `kubelet` (cAdvisor) and exposes them through the `metrics.k8s.io` API. HPA queries this API every 15 seconds to determine if replicas should scale between `2` and `5`.

---

### 6.3 Persistent Storage & StatefulSet Rationale

#### Why StatefulSet instead of Deployment for MongoDB?
| Feature | Deployment | StatefulSet | Why StatefulSet is Required |
|---|---|---|---|
| **Pod Naming** | Random hash (e.g. `mongo-68f7d9b-x9z2q`) | Predictable index (`mongodb-0`) | Critical for DNS and stable service attachment. |
| **Startup / Shutdown Order** | Random, non-deterministic | Sequential (`0 → 1 → 2`) | Prevents split-brain and ensures clean DB shutdowns. |
| **Storage Binding** | Pods share/disconnect indiscriminately | PVC remains tied to the specific ordinal Pod | Guarantees that data volume `/data/db` is consistently remounted to the same identity. |

#### Storage Architecture (PV & PVC)
- **PersistentVolume (PV)**: Represents cluster-wide persistent storage mapped to `/data/mongodb` via `hostPath`.
- **PersistentVolumeClaim (PVC)**: The request made by the database specification claiming 1Gi with `ReadWriteOnce` access mode.
- **Persistence Verification**: Deleting the `mongodb-0` Pod will cause the StatefulSet controller to immediately recreate it and reattach `mongodb-pvc`. Data created via `/data` remains intact.

---

### 6.4 Security: Kubernetes Secrets vs ConfigMaps

- **ConfigMaps** are designed for public, non-sensitive configuration values (e.g., ports, logging levels).
- **Secrets** are designed for credentials, keys, and tokens.
- We stored root credentials (`MONGO_INITDB_ROOT_USERNAME`, `MONGO_INITDB_ROOT_PASSWORD`) and the constructed authenticated `MONGODB_URI` in `mongodb-secret`.
- The secret values are injected into container environments using `valueFrom.secretKeyRef`, preventing credential leakage in repository manifests or application images.

---

## 7. Verification & QA Scenarios

### Test Scenario A: Data Persistence Validation
Verify that database records survive complete Pod destruction:
1. Insert test record:
   ```bash
   curl -X POST http://<SERVICE_URL>/data -H "Content-Type: application/json" -d '{"test": "persistence"}'
   ```
2. Delete the MongoDB Pod:
   ```bash
   kubectl delete pod mongodb-0 -n flask-app
   ```
3. Wait for the StatefulSet to restore the Pod:
   ```bash
   kubectl get pods -n flask-app -w
   ```
4. Query data once `mongodb-0` is ready:
   ```bash
   curl http://<SERVICE_URL>/data
   ```
   *Expected Result: The test record `{"test": "persistence"}` is returned successfully.*

### Test Scenario B: Pod Self-Healing & Rolling Availability
1. Kill one of the Flask Pods:
   ```bash
   kubectl delete pod $(kubectl get pods -n flask-app -l app=flask-app -o jsonpath='{.items[0].metadata.name}') -n flask-app
   ```
2. Send immediate requests to `GET /`:
   *Expected Result: Zero downtime. The NodePort service routes traffic seamlessly to the second surviving replica while Kubernetes spawns a replacement Pod.*

### Test Scenario C: Autoscaling (HPA) Verification
Generate simulated CPU load using an ephemeral container:
```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service.flask-app.svc.cluster.local:5001/; done"
```
Monitor scaling in another terminal:
```bash
kubectl get hpa flask-hpa -n flask-app -w
```
*Expected Result: CPU usage climbs past 70%, triggering HPA to scale replicas from 2 up towards 5.*

---

## 8. Cleanup Instructions

To tear down all resources and clean up the cluster:
```bash
kubectl delete namespace flask-app
kubectl delete pv mongodb-pv
```
To stop or delete Minikube:
```bash
minikube stop
# Or completely delete the cluster:
# minikube delete
```
