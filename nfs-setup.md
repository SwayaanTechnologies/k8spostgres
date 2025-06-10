## Step-by-Step Guide: NFS Server + Kubernetes Client Pod Testing

### Prerequisites

* Ubuntu VM for NFS server (with static IP or reachable by pod)
* Kubernetes cluster running (e.g., `k3d`, `kind`, or real cluster)
* `kubectl` access configured

---

## Install Docker 

```
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
docker info
```

## Install kubectl

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```

Validate the kubectl binary against the checksum file:

```
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

If valid, the output is:

```
kubectl: OK
```

Install kubectl

```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

## Install k3d and create a cluster

```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
k3d --help
k3d cluster create mycluster --servers 1 --agents 2
```

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
mkdir /mnt/nfs
mount -t nfs -o nolock <NFS_SERVER_IP>:/srv/kubernetes /mnt/nfs
```