# Prerequisites

- K8s environment (K8s, k3d, kind)
- Docker
- Tested with K3d and kind. 
  - k3d is a lightweight wrapper to run k3s (Rancher Lab’s minimal Kubernetes distribution) in docker.
  - kind is a tool for running local Kubernetes clusters using Docker container “nodes”.
- jq (optional if you want to format JSON logs outputs)

# Demo

**Create a k8s cluster**

```bash
k3d cluster create mycluster --servers=1 --agents=2
```

**Example of a Kubernetes Custom Resource Definition (CRD) for a `Person` object**

**1. CRD Definition: `person-crd.yaml`**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: people.example.com
spec:
  group: example.com
  names:
    kind: Person
    plural: people
    singular: person
    shortNames:
      - psn
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - fullName
                - age
              properties:
                fullName:
                  type: string
                age:
                  type: integer
                email:
                  type: string
                address:
                  type: string
```

---

**2. Sample Custom Resource: `sample-person.yaml`**

```yaml
apiVersion: example.com/v1
kind: Person
metadata:
  name: john-doe
spec:
  fullName: John Doe
  age: 30
  email: john.doe@example.com
  address: 123 Main Street, Anytown, USA
```

---

**3. Commands to Apply and Use**

```bash
# Apply the CRD
kubectl apply -f person-crd.yaml

# Confirm CRD was created
kubectl get crds

# Create a sample Person object
kubectl apply -f sample-person.yaml

# View the custom resource
kubectl get people

# Get detailed info
kubectl get person john-doe -o yaml
```



**Execute commands in the correct order:**

```bash
./01_install_plugin.sh
```

**Check crds**

```bash
kubectl get crd
```

**Install Operator**

```bash
./02_install_operator.sh
```

**Check Operator Status**

```bash
./03_check_operator_installed.sh
kubectl get all -n cnpg-system
```

**Install PostgreSQL Cluster**

```bash
./04_get_cluster_config_file.sh
./05_install_cluster.sh
```

**Verify Cluster**

```bash
kubectl get pod
kubectl get pvc
kubectl get cluster
```

**Open a new session and execute:**

```bash
./06_show_status.sh
```

**Check postgresql version:**

```bash
kubectl exec -it cluster-example-1 -- psql
select version();
```

**Run minio and access web ui**

```bash
kubectl apply -f minio.yaml
kubectl port-forward pod/minio 9000:9000 9001:9001
```

Access Web UI: `http://localhost:9001`

login using username `admin` and password `password`

**Connect to Container and check for minio feature**

```bash
docker ps -a
docker exec -it <container-id> bash
mc alias list
```

**Insert Data**

```bash
./07_insert_data.sh
```

**Verify Data**

```bash
kubectl exec -it cluster-example-1 -- psql
```

```sql
-- check table
\d

-- check data is inserted
select * from customers;

exit
```

**Verify Pod ( Primary & Standby )**

```bash
kubectl get services
kubectl describe service cluster-example-rw
kubectl describe service cluster-example-r
```

**Promote a instance to primary**

```bash
./08_promote.sh
```

**Upgrade the PostgreSQL Version**

```bash
./09_upgrade.sh
```

Go to the web UI of Minio and check bucket is created with the name `cnp` and check the files inside it ( Check WAL(Write-Ahead Logging) files is created ).

**Backup Cluster Data to Minio's**

```bash
./10_backup_cluster.sh
./11_backup_describe.sh
```

Go to the web UI of Minio and check bucket is created with the name `cnp` and check the files inside it ( Check backup files is created ).

**Insert new table before restore**

```bash
kubectl exec -it cluster-example-1 -- psql
```

```sql
-- check table
\d  

-- Create a new table
CREATE TABLE pre_upgrade_backup (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
exit

-- Verify the new table is created
\d

exit
```

**Restore Postgres Older Version**

```bash
./12_restore_cluster.sh
```

After Restoration, verify the data and check if the new table is still present:

```bash
kubectl exec -it cluster-example-1 -- psql
```

```sql
-- check table
\d

exit
```