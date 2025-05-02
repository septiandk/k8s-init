# 🚀 Kubernetes Cluster Setup 

Instruksi ini akan membantumu bootstrap cluster Kubernetes dari **fresh Ubuntu 20.04/22.04**, menggunakan Ansible. Cluster hanya berisi **1 Master dan 1+ Node Worker**, tanpa Load Balancer.

## 📋 Prasyarat

- Semua node sudah install **Ubuntu Server**
- User `ubuntu` ada dan punya akses sudo
- SSH key-based login antar node (tanpa password)
- Ansible di-install di master node
- IP private masing-masing node diketahui

## 🛠️ Langkah Instalasi

# 1. Clone Repo dan CD
```bash
git clone https://your.repo.url/K8S-init.git
cd K8S-init
```

# 2. Edit Inventory dan all.yml
```bash
nano inventory/hosts.ini
# Contoh isinya:
# [master]
# 172.29.249.183
#
# [workers]
# 172.29.249.184
#[loadbalancer]
# ip lb kalo use_loadbalancer: true
# [all:vars]
# ansible_user=ubuntu
```
```bash
nano group_vars/all.yml
#pod_cidr: "172.244.0.0/16"
# set ke true kalo mau pake lb
#use_loadbalancer: false
#cluster_endpoint: "172.29.249.184"
```
# 3. Pastikan SSH Key Tersambung atau copy ssh-key manual
```bash
ssh-copy-id ubuntu@172.29.249.183
ssh-copy-id ubuntu@172.29.249.184
```
# 4. Install Ansible (di Master/Executor)
```bash
sudo apt update && sudo apt install -y ansible 
```
# 5. Jalankan Playbook Persiapan Node
```bash
ansible-playbook playbooks/prepare_nodes.yml
```
# 6. Inisialisasi Kube Master
```bash
ansible-playbook playbooks/init_master.yml
```
# => Copy kubeadm join output
# 7. Join Worker ke Cluster
```bash
ansible-playbook playbooks/join_workers.yml
```
# 8. Verifikasi
```bash
kubectl get nodes
```

## 🧼 Uninstall (Opsional)
```bash
ansible all -a "sudo kubeadm reset -f && sudo systemctl stop kubelet && sudo systemctl stop containerd"
```

## 📝 Catatan Tambahan

- `containerd` role akan otomatis buat `/etc/containerd` kalau belum ada.
- Cocok untuk setup lab/dev tanpa load balancer.
