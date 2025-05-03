# 🚀 Kubernetes Cluster Setup 

Instruksi ini akan membantumu bootstrap cluster Kubernetes dari **fresh Ubuntu 20.04/22.04**, menggunakan Ansible. Cluster ini mendukung **1 Master dan 1+ Node Worker**, dengan opsi Load Balancer.

## 📋 Prasyarat

- Semua node sudah menggunakan **Ubuntu Server**.
- User `ubuntu` ada di semua node dan memiliki akses sudo.
- SSH key-based login antar node sudah diatur (tanpa password).
- Ansible di-install di node executor (biasanya master node).
- IP private masing-masing node diketahui.

## 🛠️ Langkah Instalasi

### 1. Clone Repo dan Masuk ke Direktori
```bash
git clone https://your.repo.url/K8S-init.git
cd K8S-init
```

### 2. Edit Inventory dan Konfigurasi Global
Edit file `inventory/hosts.ini` untuk mendefinisikan IP node master, worker, dan load balancer (jika digunakan):
```ini
[masters]
k8s-master ansible_host=172.29.249.184 ansible_connection=local

[workers]
k8s-worker ansible_host=172.29.249.183

[loadbalancer]

[cluster:children]
masters
workers

[all:vars]
ansible_user=ubuntu
```

Edit file `group_vars/all.yml` untuk konfigurasi global:
```yaml
pod_cidr: "172.244.0.0/16"  # CIDR untuk jaringan pod
use_loadbalancer: false     # Set ke true jika menggunakan load balancer
cluster_endpoint: "172.29.249.184"  # IP atau domain endpoint cluster
kubernetes_version: "v1.29" # Set kubernetes version ke versi yang di inginkan dapat di check disini https://shorturl.at/TMD6K
```

### 3. Pastikan SSH Key Tersambung
Pastikan SSH key-based login sudah diatur untuk semua node atau copy manual ssh-key:
```bash
ssh-copy-id ubuntu@172.29.249.183
ssh-copy-id ubuntu@172.29.249.184
```

### 4. Install Ansible di Node Executor
Install Ansible di node executor (biasanya master node):
```bash
sudo apt update && sudo apt install -y ansible 
```

### 5. Jalankan Playbook Persiapan Node
Persiapkan semua node (master dan worker):
```bash
ansible-playbook playbooks/prepare_nodes.yml
```

### 6. Inisialisasi Kube Master
Inisialisasi master node:
```bash
ansible-playbook playbooks/init_master.yml
```

### 7. Join Worker ke Cluster
Tambahkan worker node ke cluster:
```bash
ansible-playbook playbooks/join_worker.yml
```

### 8. Verifikasi
Pastikan semua node sudah bergabung ke cluster:
```bash
kubectl get nodes
```

---

## 🔄 Upgrade Cluster Kubernetes

### 1. Tentukan Versi Upgrade
Edit file `group_vars/all.yml` untuk menentukan versi Kubernetes yang diinginkan:
```yaml
kubernetes_version: "1.29.0-00"
```

### 2. Upgrade Master Node
Jalankan playbook untuk upgrade master node:
```bash
ansible-playbook playbooks/upgrade_node.yml --limit masters
```

### 3. Upgrade Worker Node
Jalankan playbook untuk upgrade worker node:
```bash
ansible-playbook playbooks/upgrade_node.yml --limit workers
```

### 4. Verifikasi
Pastikan semua node sudah menggunakan versi terbaru:
```bash
kubectl get nodes
kubectl version
```

---

## 🧼 Teardown Cluster (Opsional)

Jika ingin menghapus cluster Kubernetes, jalankan playbook berikut:
```bash
ansible-playbook playbooks/teardown_cluster.yml
```

Playbook ini akan:
- Menghapus semua konfigurasi Kubernetes.
- Menghapus paket Kubernetes (`kubeadm`, `kubelet`, `kubectl`).
- Membersihkan direktori terkait Kubernetes seperti `/etc/kubernetes`, `/var/lib/kubelet`, dll.

---

## 📝 Catatan Tambahan

- **Containerd**:
  - Playbook secara otomatis akan menginstal dan mengonfigurasi `containerd`.
  - Direktori `/etc/containerd` akan dibuat jika belum ada.
- **Load Balancer**:
  - Jika `use_loadbalancer: true`, playbook akan mengatur `keepalived` dan `haproxy` untuk load balancer.
  - Konfigurasi load balancer dapat ditemukan di `templates/haproxy.cfg.j2` dan `templates/keepalived.conf.j2`.
- **Flannel**:
  - Jaringan pod menggunakan Flannel. File konfigurasi Flannel diterapkan secara otomatis selama inisialisasi master node.

---

## 📂 Struktur Direktori

```
K8S-init/
├── ansible.cfg
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── all.yml
│       └── loadbalancer.yml
├── playbooks/
│   ├── init_master.yml
│   ├── join_master.yml
│   ├── join_worker.yml
│   ├── prepare_nodes.yml
│   ├── setup_loadbalancer.yml
│   ├── teardown_cluster.yml
│   └── upgrade_node.yml
├── roles/
│   ├── common/
│   │   └── tasks/
│   │       └── main.yml
│   ├── containerd/
│   │   └── tasks/
│   │       └── main.yml
│   ├── kube/
│   │   └── tasks/
│   │       └── main.yml
│   └── loadbalancer/
│       └── tasks/
│           └── main.yml
├── templates/
│   ├── haproxy.cfg.j2
│   └── keepalived.conf.j2
└── README.md
```

---
````
