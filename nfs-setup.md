## Step-by-Step Guide: NFS Server + Kubernetes Client Pod Testing

### Prerequisites

* Ubuntu VM for NFS server (with static IP or reachable by pod)
* Kubernetes cluster running (e.g., `k3d`, `kind`, or real cluster)
* `kubectl` access configured

---

## Step 1: Install & Configure NFS Server on Ubuntu

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

### Create a shared directory

```bash
sudo mkdir -p /srv/kubernetes
sudo mkdir -p /srv/kubernetes/volumes
sudo mkdir -p /srv/kubernetes/backup
sudo chown nobody:nogroup /srv/kubernetes
sudo chmod 777 /srv/kubernetes
```

### Configure exports

Edit `/etc/exports`:

```bash
sudo nano /etc/exports
```

Add:

```
/srv/kubernetes *(rw,sync,no_subtree_check,no_root_squash,insecure)
```

### Apply export configuration

```bash
sudo exportfs -rav
```

### Restart NFS services

```bash
sudo systemctl restart nfs-kernel-server
sudo systemctl status nfs-kernel-server
```

---

## Step 2: Enable Firewall

```bash
sudo ufw enable
sudo reboot
```

**Allow NFS Ports via UFW**(Optional)

```
sudo ufw allow 111/tcp
sudo ufw allow 111/udp
sudo ufw allow 2049/tcp
sudo ufw allow 2049/udp
```

> Use `sudo ufw status` to verify.

---

## Step 3: Create Client Pod in Kubernetes to Mount NFS

### Create `nfs-client.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-client
spec:
  hostNetwork: true
  containers:
  - name: shell
    image: docker.io/vijaynvb/nfs-common:latest
    command: ["/bin/bash", "-c", "sleep infinity"]
    securityContext:
      privileged: true
    stdin: true
    tty: true
  restartPolicy: Never
```

### Apply and wait

```bash
kubectl apply -f nfs-client.yaml
kubectl get pods
```

---

## Step 4: Manually Mount NFS from the Pod

```bash
kubectl exec -it nfs-client -- bash
```

Inside the container:

```bash
mount -t nfs -o nolock <NFS_SERVER_IP>:/srv/kubernetes /mnt/nfs
```