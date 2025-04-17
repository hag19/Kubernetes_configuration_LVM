Here's your Kubernetes setup guide formatted in Markdown, with added steps for proper user permissions:

```markdown
# Kubernetes Cluster Setup with LVM Storage

## 1. Prepare LVM Storage
```bash
sudo pvcreate /dev/sda
sudo vgcreate vg_kubs /dev/sda
sudo lvcreate -l 100%FREE -n lv_kubs vg_kubs
sudo mkfs.xfs /dev/vg_kubs/lv_kubs
sudo mkdir -p /mnt/kubs
sudo mount /dev/vg_kubs/lv_kubs /mnt/kubs
```

## 2. Configure fstab
```bash
sudo nano /etc/fstab
```
Add:
```
/dev/vg_kubs/lv_kubs /mnt/kubs xfs defaults 0 2
```
Then:
```bash
sudo mount -a
df -h /mnt/kubs
```

## 3. Set Proper Permissions for User Access
```bash
sudo chown -R $USER:$USER /mnt/kubs
sudo chmod -R 775 /mnt/kubs
sudo setfacl -Rdm u:$USER:rwx /mnt/kubs
```

## 4. System Preparation
```bash
sudo apt update && sudo apt install -y rsync
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 5. Kernel Modules
```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

## 6. Network Configuration
```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

## 7. Install Containerd
```bash
sudo apt update
sudo apt -y install containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## 8. Install Kubernetes Components
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt update
sudo apt install kubelet kubeadm kubectl -y
sudo apt-mark hold kubelet kubeadm kubectl
```

## 9. Configure Containerd Storage
```bash
sudo rsync -av /var/lib/containerd/ /mnt/kubs/containerd/
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's|root = "/var/lib/containerd"|root = "/mnt/kubs/containerd"|' /etc/containerd/config.toml
sudo chown -R $USER:$USER /mnt/kubs/containerd
sudo systemctl restart containerd
```

## 10. Firewall Configuration (Master Node)
```bash
sudo ufw allow 6443/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw reload
```

## 11. Initialize Cluster
Create `kubelet.yaml`:
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.32.0"
controlPlaneEndpoint: "k8s-master"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
```

Then:
```bash
sudo kubeadm init --config kubelet.yaml
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 12. Install CNI (Calico)
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 51820/udp
sudo ufw allow 51821/udp
sudo ufw reload
```

## Post-Installation Checks
```bash
kubectl get nodes
kubectl get pods -A
```
```

### Key Permission Notes:
1. The `/mnt/kubs` directory is owned by your user with 775 permissions
2. Containerd's storage directory has proper user ownership
3. Kubernetes config file is accessible to your user
4. Default ACLs are set for future files in `/mnt/kubs`

This setup ensures:
- All container data is stored on your LVM volume
- Your regular user can access all necessary directories
- The cluster has proper networking configuration
- Storage is properly managed and scalable
