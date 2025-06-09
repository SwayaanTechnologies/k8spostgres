# **PostgreSQL on Kubernetes**

## **Table of Content**

* [**Manual PostgreSQL Deployment with Persistent Storage**](#manual-postgresql-deployment-with-persistent-storage)
* [**PostgreSQL Initialization and Manual Replication**](#postgresql-initialization-and-manual-replication)
* [**Backup, PITR and Restore**](#backup-pitr-and-restore)
* [**High Availability and Manual Failover**](#high-availability-and-manual-failover)

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

### **Folder Structure for Day 1**

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

### **Folder Structure for Day 2**

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

### **Hands-On Guide for Day 2**

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

## **Backup, PITR and Restore**

* [**Backup Strategies in PostgreSQL**](#backup-strategies-in-postgresql)
* [**Write-Ahead Logging**](#write-ahead-logging)
* [**Point-In-Time Recovery**](#point-in-time-recovery)
* [**Folder Structure for Day 3**](#folder-structure-for-day-3)
* [**Hands-On Guide for Day 3**](#hands-on-guide-for-day-3)

### **Backup Strategies in PostgreSQL**

| Type     | Tool Used       | Description                        |
| -------- | --------------- | ---------------------------------- |
| Physical | `pg_basebackup` | Exact file-level copy of data dir  |
| Logical  | `pg_dump`       | SQL dump of database schema + data |

> We focus on **physical backup** today, required for WAL-based PITR.

---

### **Write-Ahead Logging**

* WAL logs every change before applying it to disk.
* Enables:

  * Crash recovery
  * Streaming replication
  * PITR

* Logs are archived externally for PITR.

---

### **Point-In-Time Recovery**

* Restore from a physical backup
* Reapply WAL files *up to a specific timestamp*
* Useful for recovering from accidental data loss

---

### **Folder Structure for Day 3**

```
day3-backup-pitr/
├── cronjob-backup.yaml
├── backup-script.sh
├── wal-archive-pvc.yaml
├── recovery-statefulset.yaml
├── recovery.conf (PostgreSQL 12+ = standby.signal)
```

---

### **Hands-On Guide for Day 3**

* [**Enable WAL Archiving in Primary**](#enable-wal-archiving-in-primary)
* [**Create PVC for WAL Archive**](#create-pvc-for-wal-archive)
* [**Take Physical Backup with `pg_basebackup`**](#take-physical-backup-with-pg_basebackup)
* [**Automate with Kubernetes CronJob**](#automate-with-kubernetes-cronjob)
* [**Perform PITR**](#perform-pitr)
* [**Validate PITR**](#validate-pitr)

#### **Enable WAL Archiving in Primary**

Update `postgresql.conf` via ConfigMap:

```ini
archive_mode = on
archive_command = 'cp %p /wal-archive/%f'
wal_level = replica
archive_timeout = 60
```

Mount `/wal-archive` to a PVC or NFS volume:

```yaml
volumeMounts:
  - name: wal-archive
    mountPath: /wal-archive
```

---

#### **Create PVC for WAL Archive**

```yaml
# wal-archive-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wal-archive-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f wal-archive-pvc.yaml
```

---

#### **Take Physical Backup with `pg_basebackup`**

Create a `Job` or use a sidecar container:

**backup-script.sh**

```bash
#!/bin/bash
pg_basebackup -h postgres-0.postgres.default.svc.cluster.local \
  -U replicator \
  -D /backup/pgdata \
  -Fp -Xs -P -R

touch /backup/pgdata/backup_label
```

Mount `/backup` to PVC or NFS volume.

---

#### **Automate with Kubernetes CronJob**

```yaml
# cronjob-backup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pg-backup-job
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:15
              command: ["/scripts/backup-script.sh"]
              volumeMounts:
                - name: backup
                  mountPath: /backup
                - name: scripts
                  mountPath: /scripts
          restartPolicy: OnFailure
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: pg-backup-pvc
            - name: scripts
              configMap:
                name: backup-script
```

---

#### **Perform PITR**

**Prepare Recovery Config**

Create a new StatefulSet using the backup and WAL archive:

```ini
# recovery.conf (for versions < 12) — For 12+, use recovery.signal file
restore_command = 'cp /wal-archive/%f %p'
recovery_target_time = '2025-06-04 10:00:00'
```

Create an empty file `recovery.signal` in `/var/lib/postgresql/data/`

**Modify StatefulSet**

```yaml
# recovery-statefulset.yaml
volumeMounts:
  - name: backup
    mountPath: /var/lib/postgresql/data
  - name: wal-archive
    mountPath: /wal-archive
initContainers:
  - name: copy-backup
    image: busybox
    command: ["sh", "-c", "cp -r /backup/pgdata/* /var/lib/postgresql/data/"]
    volumeMounts:
      - name: backup
        mountPath: /backup
      - name: data
        mountPath: /var/lib/postgresql/data
```

Deploy:

```bash
kubectl apply -f recovery-statefulset.yaml
```

---

#### **Validate PITR**

**1.** Connect to the recovered pod:

```bash
kubectl exec -it recovery-0 -- psql -U postgres -d appdb
```

**2.** Run a query:

```sql
SELECT * FROM test;
```

**3.** Confirm that data has been recovered **up to the specified timestamp**.

---

## **High Availability and Manual Failover**

* [**What is High Availability**](#what-is-high-availability)
* [**Limitations of Manual HA Without an Operator**](#limitations-of-manual-ha-without-an-operator)
* [**Leader Election and Traffic Redirection**](#leader-election-and-traffic-redirection)
* [**Folder Structure for Day 4**](#folder-structure-for-day-4)
* [**Hands-On Guide for Day 4**](#hands-on-guide-for-day-4)

### **What is High Availability**

High Availability ensures PostgreSQL remains accessible even in the event of:

* Pod failures
* Node crashes
* Scheduled maintenance

In our setup:

* **Primary** handles writes
* **Replicas** serve read-only queries and can be promoted manually

---

### **Limitations of Manual HA Without an Operator**

| Challenge               | Description                                                |
| ----------------------- | ---------------------------------------------------------- |
| Manual Promotion        | No automated detection or switchover                       |
| Split Brain             | Risk if both old primary and promoted replica are writable |
| DNS Propagation         | Re-routing clients can be error-prone and slow             |
| StatefulSet Limitations | Identity management is static; requires careful handling   |

---

### **Leader Election and Traffic Redirection**

Without an operator, **leadership logic** and **client traffic** redirection must be handled using:

* Kubernetes Services (e.g., `ClusterIP`/`Headless`)
* Manual `pg_ctl promote` commands
* Optional CronJobs or sidecars for health/failover checks

---

### **Folder Structure for Day 4**

```
day4-ha-failover/
├── primary-service.yaml
├── readiness-liveness.yaml
├── promote-replica.sh
├── failover-checker.yaml
├── pod-anti-affinity.yaml
```

---

### **Hands-On Guide for Day 4**

* [**Simulate Primary Failure**](#simulate-primary-failure)
* [**Promote a Replica**](#promote-a-replica)
* [**Redirect Client Traffic**](#redirect-client-traffic)
* [**Add Readiness & Liveness Probes**](#add-readiness--liveness-probes)
* [**Use Pod Anti-Affinity for HA**](#use-pod-anti-affinity-for-ha)
* [**Implement Failover Detection**](#implement-failover-detection)

#### **Simulate Primary Failure**

You can simulate a primary pod failure by deleting it:

```bash
kubectl delete pod postgres-0
```

OR simulate a node crash (if multi-node setup):

```bash
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets
```

---

#### **Promote a Replica**

Manually connect to a replica pod and promote:

```bash
kubectl exec -it postgres-1 -- bash
pg_ctl promote -D /var/lib/postgresql/data
```

> This converts the replica to a standalone writable primary.

---

#### **Redirect Client Traffic**

Update `postgres-primary-svc` to point to the new leader pod:

```yaml
# primary-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    statefulset.kubernetes.io/pod-name: postgres-1
  ports:
    - port: 5432
      targetPort: 5432
```

Apply:

```bash
kubectl apply -f primary-service.yaml
```

Clients using `postgres-primary` will now connect to the new promoted pod.

---

#### **Add Readiness & Liveness Probes**

```yaml
# readiness-liveness.yaml (to be added in StatefulSet spec)
readinessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

livenessProbe:
  exec:
    command: ["pg_isready", "-U", "postgres"]
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 5
```

These ensure Kubernetes recognizes pod health and availability before routing traffic.

---

#### **Use Pod Anti-Affinity for HA**

Distribute pods across nodes to prevent single-point failure:

```yaml
# pod-anti-affinity.yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: postgres
        topologyKey: "kubernetes.io/hostname"
```

Add this to the StatefulSet to schedule pods on different nodes.

---

#### **Implement Failover Detection**

Create a simple CronJob to detect primary failure and promote a replica:

```yaml
# failover-checker.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: failover-checker
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: failover-checker
              image: postgres:15
              command: ["/bin/bash", "-c"]
              args:
                - |
                  if ! pg_isready -h postgres-primary.default.svc.cluster.local; then
                    kubectl exec postgres-1 -- pg_ctl promote -D /var/lib/postgresql/data
                    # Update Service selector via script or webhook if needed
                  fi
          restartPolicy: OnFailure
```

---