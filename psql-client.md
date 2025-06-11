# PostgreSQL Client Access in Kubernetes using `psql-client`

This guide demonstrates how to deploy a PostgreSQL client pod in Kubernetes, connect to both read-write and read-only endpoints of a CloudNativePG (CNPG) cluster, create a table, and test failover promotion.

---

## 1. Deploy the `psql-client` Pod

Apply the Kubernetes manifest to deploy the pod and secret:

```bash
kubectl apply -f psql-client.yaml
```

**Expected Output:**

```text
secret/cluster-example-db created
pod/psql-client created
```

---

## 2. Access the Client Pod and Check Environment Variables

Enter the pod:

```bash
kubectl exec -it psql-client -- /bin/sh
```

Inside the pod, print the environment variables:

```sh
echo $POSTGRES_USER
echo $POSTGRES_PASSWORD
echo $POSTGRES_DB
echo $SERVICE_NAME
```

**Expected Output:**

```text
<your-username>
<your-password>
app
cluster-example-rw.default.svc.cluster.local
```

> These values are loaded from Kubernetes secrets and used to connect to PostgreSQL.

---

## 3. Connect to the Primary (Read-Write) Endpoint and Create a Table

Connect to the database:

```sh
PGPASSWORD=$POSTGRES_PASSWORD psql -h "$SERVICE_NAME" -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

At the `psql` prompt:

```sql
\d
```

Now, create a table:

```sql
CREATE TABLE pre_upgrade_backup (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Then run:

```sql
\d
```

**Expected Output:**

```text
List of relations
 Schema |        Name         | Type  | Owner
--------+---------------------+-------+----------
 public | pre_upgrade_backup  | table | <your-user>
```

Exit the `psql` prompt and the pod:

```sql
\q
```

---

## 4. Connect to the Read-Only Endpoint

Update the service name to the read-only endpoint:

```bash
export SERVICE_NAME=cluster-example-ro.default.svc.cluster.local
```

Connect to the DB again:

```sh
PGPASSWORD=$POSTGRES_PASSWORD psql -h "$SERVICE_NAME" -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

Try to create the same table:

```sql
CREATE TABLE pre_upgrade_backup (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Expected Output:**

```text
ERROR:  cannot execute CREATE TABLE in a read-only transaction
```

This confirms the read-only limitation of replica endpoints.

Exit:

```sql
\q
exit
```

---

## 5. Promote a Replica to Primary

Run the promotion script:

```bash
./08_promote.sh
```

**Expected Output:**

Something similar to:

```text
Promotion request sent to the replica pod.
```

This will trigger a replica to become the new primary.

---

## 6. Validate the New Primary

Re-enter the pod:

```bash
kubectl exec -it psql-client -- /bin/sh
```

Connect again using the same service name (which now points to the newly promoted primary):

```sh
PGPASSWORD=$POSTGRES_PASSWORD psql -h "$SERVICE_NAME" -U "$POSTGRES_USER" -d "$POSTGRES_DB"
```

Try to create the table again:

```sql
CREATE TABLE pre_upgrade_backup (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Expected Output:**

```text
ERROR:  relation "pre_upgrade_backup" already exists
```

This confirms:

* The promotion succeeded
* Data is intact
* The table created earlier on the original primary is present

Exit:

```sql
\q
exit
```

---