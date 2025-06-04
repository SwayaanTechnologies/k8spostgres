# **PostgreSQL on Kubernetes**

## **Table of Content**

* [**Manual PostgreSQL Deployment with Persistent Storage**](#manual-postgresql-deployment-with-persistent-storage)
* [**PostgreSQL Initialization and Manual Replication**](#postgresql-initialization-and-manual-replication)

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

## **PostgreSQL Initialization and Manual Replication**

**PostgreSQL Initialization**

* PostgreSQL uses the Docker image's `/docker-entrypoint.sh` script.
* Any `.sql` or `.sh` files placed in `/docker-entrypoint-initdb.d/` will be executed only **on first initialization** of the database directory.

**Streaming Replication Essentials**

| Setting               | Description                           |
| --------------------- | ------------------------------------- |
| `wal_level = replica` | Enables write-ahead logging           |
| `max_wal_senders`     | Number of replica connections allowed |
| `hot_standby = on`    | Enables read-only replicas            |
| `primary_conninfo`    | Connection string to primary          |
| `restore_command`     | Optional for PITR (used later)        |

**Authentication Setup**

* Password-less replication uses `.pgpass` file or environment variable injection via Kubernetes **Secrets**.

---

### **Folder Structure**

```
day2-postgres-replication/
├── init/
│   ├── init-db.sh
│   └── init-user.sql
├── primary-statefulset.yaml
├── replica-statefulset.yaml
├── replication-secret.yaml
├── replica-service.yaml
```

---

### **Hands-On Guide**

* [**Create Init Scripts for Primary Initialization**](#create-init-scripts-for-primary-initialization)
* [**Create Secret for Replication Credentials**](#create-secret-for-replication-credentials)
* [**Configure Primary StatefulSet**](#configure-primary-statefulset)
* [**Deploy Primary StatefulSet**](#deploy-primary-statefulset)
* [**Deploy Replica StatefulSet**](#deploy-replica-statefulset)
* [**Expose Replica Service**](#expose-replica-service)
* [**Verify Replication**](#verify-replication)

#### **Create Init Scripts for Primary Initialization**

Create a `ConfigMap` or mount local files using an `emptyDir` volume.

**init-db.sh**

```bash
#!/bin/bash
psql -U postgres <<EOF
CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'replica_pass';
CREATE DATABASE appdb;
EOF
```

Mount into an **init container** as follows:

```yaml
initContainers:
  - name: init-db
    image: postgres:15
    command: [ "bash", "-c", "/scripts/init-db.sh" ]
    volumeMounts:
      - name: init-scripts
        mountPath: /scripts
```

---

#### **Create Secret for Replication Credentials**

```yaml
# replication-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: replication-secret
type: Opaque
stringData:
  REPLICATION_USER: replicator
  REPLICATION_PASSWORD: replica_pass
```

Apply:

```bash
kubectl apply -f replication-secret.yaml
```

---

#### **Configure Primary StatefulSet**

Key environment variables and `postgresql.conf` parameters:

```yaml
env:
  - name: POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: POSTGRES_PASSWORD
  - name: POSTGRES_INITDB_ARGS
    value: "--auth-host=md5"

volumeMounts:
  - name: data
    mountPath: /var/lib/postgresql/data
  - name: config
    mountPath: /etc/postgresql
    readOnly: true

configMap includes:
```

**postgresql.conf**

```ini
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64
hot_standby = on
```

---

#### **Deploy Primary StatefulSet**

Use modified version from Day 1, include:

* init container
* volume for `init-db.sh`
* replication credentials
* ConfigMap with `postgresql.conf`

```bash
kubectl apply -f primary-statefulset.yaml
```

---

#### **Deploy Replica StatefulSet**

Key changes in replicas:

* Use same image and PVC setup
* Include `primary_conninfo` with credentials from Secret
* Add `standby.signal` file during startup to initiate replica mode

Example `replica-start.sh` (entrypoint override):

```bash
#!/bin/bash
echo "*:*:*:replicator:${REPLICATION_PASSWORD}" > ~/.pgpass
chmod 600 ~/.pgpass

echo "primary_conninfo = 'host=postgres-0.postgres.default.svc.cluster.local user=replicator password=${REPLICATION_PASSWORD}'" >> /var/lib/postgresql/data/postgresql.conf

touch /var/lib/postgresql/data/standby.signal

exec docker-entrypoint.sh postgres
```

Use this script in a ConfigMap and inject via init container or as a wrapper command.

---

#### **Expose Replica Service**

```yaml
# replica-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
spec:
  selector:
    app: postgres-replica
  ports:
    - port: 5432
      targetPort: 5432
```

---

#### **Verify Replication**

Run the client pod:

```bash
kubectl run -it postgres-client --rm --image=postgres \
  -- psql -h postgres-0.postgres.default.svc.cluster.local -U postgres
```

Inside `psql`, insert sample data:

```sql
CREATE TABLE test(id INT, name TEXT);
INSERT INTO test VALUES (1, 'replication test');
```

Now connect to replica:

```bash
psql -h postgres-replica.default.svc.cluster.local -U postgres -d appdb
SELECT * FROM test;
```

> You should see the same data — replication works!

---