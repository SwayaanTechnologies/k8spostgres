# **PostgreSQL on Kubernetes**

## **Table of Content**

* [**Manual PostgreSQL Deployment with Persistent Storage**](#manual-postgresql-deployment-with-persistent-storage)

## **Manual PostgreSQL Deployment with Persistent Storage**

Deploy PostgreSQL manually on Kubernetes using StatefulSets, ConfigMaps, Secrets, and persistent storage (NFS or HostPath). No operator will be used—this setup is completely manual and educational.

---

### **Prerequisites**

Before beginning, ensure the following are installed and configured:

* A running Kubernetes cluster (Minikube/kind/k3s or multi-node)
* `kubectl` CLI configured
* Docker installed (for image work if required)
* NFS server running (if using NFS-based PVs)

---

### **Folder Structure**

```
day1-postgres-deployment/
├── postgresql.conf
├── nfs-pv.yaml
├── hostpath-pv.yaml
├── postgres-statefulset.yaml
├── postgres-service.yaml
```

---

### **Step-by-Step Guide**

* [**1. Create Persistent Volume & PVC**](#1-create-persistent-volume--pvc)
* [**2. Create PostgreSQL ConfigMap**](#2-create-postgresql-configmap)
* [**3. Create PostgreSQL Secret**](#3-create-postgresql-secret)
* [**4. Deploy PostgreSQL with StatefulSet**](#4-deploy-postgresql-with-statefulset)
* [**5. Expose via Headless Service**](#5-expose-via-headless-service)
* [**6. Verify Deployment**](#6-verify-deployment)
* [**7. Connect Using `psql`**](#7-connect-using-psql)

---

#### **1. Create Persistent Volume & PVC**

**Option A: NFS (multi-node compatible)**

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-nfs
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: <NFS_SERVER_IP>
    path: "/export/postgres"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc-nfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

**Option B: HostPath (use only for single-node clusters like Minikube)**

```yaml
# hostpath-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-hostpath
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/postgres"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc-hostpath
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply the file:

```bash
kubectl apply -f nfs-pv.yaml
# OR
kubectl apply -f hostpath-pv.yaml
```

---

#### **2. Create PostgreSQL ConfigMap**

```bash
kubectl create configmap postgres-config \
  --from-file=postgresql.conf=./postgresql.conf
```

Sample `postgresql.conf`:

```ini
listen_addresses = '*'
max_connections = 100
```

---

#### **3. Create PostgreSQL Secret**

```bash
kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD=mysecurepassword
```

---

#### **4. Deploy PostgreSQL with StatefulSet**

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: "postgres"
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /etc/postgresql
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: postgres-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```

Apply:

```bash
kubectl apply -f postgres-statefulset.yaml
```

---

#### **5. Expose via Headless Service**

```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Apply:

```bash
kubectl apply -f postgres-service.yaml
```

---

#### **6. Verify Deployment**

```bash
kubectl get pods -l app=postgres
kubectl get pvc
```

---

#### **7. Connect Using `psql`**

Run a temporary PostgreSQL client pod:

```bash
kubectl run -it postgres-client --rm --image=postgres \
  -- psql -h postgres-0.postgres.default.svc.cluster.local -U postgres
```

> When prompted for password, enter the one defined in the Secret.

---