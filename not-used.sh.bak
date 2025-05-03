#!/bin/bash
set -euo pipefail

log() {
  local TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
  echo -e "\e[1;34m[$TIMESTAMP] [TASK]\e[0m $1"
}

fail() {
  local TIMESTAMP=$(date +"%Y-%m-%d %H:%M:%S")
  echo -e "\e[1;31m[$TIMESTAMP] [ERROR]\e[0m $1" >&2
}

cleanup() {
  log "[task] Membersihkan file sementara"
  rm -f /tmp/kube-flannel.yml
}

trap cleanup EXIT

retry_ssh() {
  local CMD=$1
  local RETRIES=3
  local DELAY=5

  for ((i=1; i<=RETRIES; i++)); do
    if $CMD; then
      return 0
    fi
    echo "[WARNING] Percobaan $i gagal. Mencoba lagi dalam $DELAY detik..."
    sleep $DELAY
  done

  fail "Perintah SSH gagal setelah $RETRIES percobaan."
  return 1
}

log "[task] Memulai proses setup Kubernetes Cluster"

read -p "Berapa jumlah total node (minimal 2)? " TOTAL_NODE
if [[ $TOTAL_NODE -lt 2 ]]; then
  fail "Minimal butuh 2 node (master dan worker)."
  exit 1
fi

read -p "Apakah cluster ini akan menggunakan load balancer? (y/n): " USE_LB

log "[task] Menyiapkan SSH key dan akses passwordless ke semua node"
if [[ ! -f ~/.ssh/id_rsa ]]; then
  log "[task] Membuat SSH key baru"
  ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
fi

PUB_KEY=$(cat ~/.ssh/id_rsa.pub)
declare -A NODE_ROLE
MASTER_IPS=()
SELF_IP=$(hostname -I | awk '{print $1}')
log "[info] IP node ini (self): $SELF_IP"

for (( i=1; i<=TOTAL_NODE; i++ )); do
  if [[ $i -eq 1 && "$SELF_IP" != "" ]]; then
    echo "[info] Node ini akan otomatis menjadi Master 1."
    NODE_IP=$SELF_IP
    ROLE="master"
  else
    read -p "Masukkan IP node ke-$i: " NODE_IP
    if [[ ! $NODE_IP =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
      fail "IP tidak valid: $NODE_IP"
      exit 1
    fi

    if [[ "$USE_LB" == "y" ]]; then
      read -p "Peran node ini (loadbalancer/master/worker): " ROLE
    else
      read -p "Peran node ini (master/worker): " ROLE
    fi

    if [[ "$ROLE" != "master" && "$ROLE" != "worker" ]]; then
      fail "Peran tidak valid: $ROLE. Harus 'master' atau 'worker'."
      exit 1
    fi
  fi

  NODE_ROLE[$NODE_IP]=$ROLE

  if [[ "$NODE_IP" != "$SELF_IP" ]]; then
    log "[task] Mencoba menyalin SSH key ke node $NODE_IP"
    if ! ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@$NODE_IP; then
      log "[ERROR] Gagal melakukan SSH ke node $NODE_IP. Silakan copy manual public key berikut ini ke node $NODE_IP:"
      echo "$PUB_KEY"
      read -p "Sudahkah public key disalin manual? (y/n): " copied
      if [[ "$copied" != "y" ]]; then
        fail "Proses setup dibatalkan karena SSH key belum disalin manual."
        exit 1
      fi
    fi
  fi

  [[ "$ROLE" == "master" ]] && MASTER_IPS+=("$NODE_IP")
done

MASTER1_IP="${MASTER_IPS[0]:-}"
if [[ -z "$MASTER1_IP" ]]; then
  fail "Tidak ada node master yang ditentukan."
  exit 1
fi

if [[ "$USE_LB" == "y" ]]; then
  read -p "Masukkan VIP (virtual IP) untuk load balancer: " CLUSTER_ENDPOINT
  read -p "Masukkan nama interface untuk keepalived (contoh: eth0): " INTERFACE
  read -p "Masukkan password keepalived: " PASSWORD
else
  CLUSTER_ENDPOINT=$MASTER1_IP
fi

log "[task] Menentukan master dan endpoint cluster"
log "Master pertama: $MASTER1_IP"
log "Cluster endpoint: $CLUSTER_ENDPOINT"

if [[ "$USE_LB" == "y" ]]; then
  log "[task] Memeriksa status load balancer"
  if ! curl -k https://$CLUSTER_ENDPOINT:6443/healthz; then
    fail "Load balancer tidak berfungsi. Periksa konfigurasi."
    exit 1
  fi
  log "Load balancer berfungsi dengan baik."
fi

SSH_CMD="ssh -o StrictHostKeyChecking=no ubuntu"
KUBECTL_CMD="kubectl --kubeconfig=$HOME/.kube/config"

for IP in "${!NODE_ROLE[@]}"; do
  ROLE=${NODE_ROLE[$IP]}
  log "[task] Menyiapkan node $IP sebagai $ROLE"

  if [[ "$ROLE" == "loadbalancer" ]]; then
    MASTER_LIST=""
    n=1
    for mIP in "${MASTER_IPS[@]}"; do
      MASTER_LIST+="    server k8s-master-$n $mIP:6443 check fall 3 rise 2\n"
      ((n++))
    done

    ssh ubuntu@$IP "mkdir -p /etc/keepalived /etc/haproxy"
    sed -e "s|{VIP}|$CLUSTER_ENDPOINT|g" \
        -e "s|{INTERFACE}|$INTERFACE|g" \
        -e "s|{PASSWORD}|$PASSWORD|g" keepalived.conf.template | \
        ssh ubuntu@$IP "cat > /etc/keepalived/keepalived.conf"

    sed "s|{MASTER_LIST}|$MASTER_LIST|g" haproxy.cfg.template | \
        ssh ubuntu@$IP "cat >> /etc/haproxy/haproxy.cfg"

    ssh ubuntu@$IP 'bash -euo pipefail -s' <<'EOF'
echo "[task] Menginstall dan mengkonfigurasi Keepalived dan HAProxy"
sudo apt update && sudo apt install -y keepalived haproxy
cat <<EOS | sudo tee /etc/keepalived/check_apiserver.sh > /dev/null
#!/bin/sh
errorExit() {
  echo "*** \$@" 1>&2
  exit 1
}
curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q {VIP}; then
  curl --silent --max-time 2 --insecure https://{VIP}:6443/ -o /dev/null || errorExit "Error GET https://{VIP}:6443/"
fi
EOS
sudo chmod +x /etc/keepalived/check_apiserver.sh
sudo systemctl enable --now keepalived
sudo systemctl enable haproxy && sudo systemctl restart haproxy
EOF

  elif [[ "$ROLE" == "master" || "$ROLE" == "worker" ]]; then
    log "[task] Menyiapkan konfigurasi dasar untuk node $IP"

    if [[ "$IP" == "$SELF_IP" ]]; then
      bash <<'EOF'
echo "[task] Menonaktifkan swap"
sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab

echo "[task] Menonaktifkan firewall (UFW)"
sudo systemctl disable --now ufw

echo "[task] Memuat modul kernel yang dibutuhkan"
cat <<MOD | sudo tee /etc/modules-load.d/containerd.conf > /dev/null
overlay
br_netfilter
MOD
sudo modprobe overlay && sudo modprobe br_netfilter

echo "[task] Mengkonfigurasi sysctl untuk Kubernetes"
cat <<SYSCTL | sudo tee /etc/sysctl.d/kubernetes.conf > /dev/null
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
SYSCTL
sudo sysctl --system

echo "[task] Menginstal containerd dan dependensinya"
sudo apt update && sudo apt install -y containerd apt-transport-https gpg ca-certificates

echo "[task] Menyiapkan containerd"
sudo mkdir -p /etc/containerd
if sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null; then
  echo "[info] Konfigurasi containerd berhasil dibuat"
else
  echo "[ERROR] Gagal membuat konfigurasi containerd. Memeriksa log dan mencoba lagi."
  exit 1
fi

if sudo systemctl restart containerd && sudo systemctl enable containerd; then
  echo "[info] containerd berhasil di-restart dan diaktifkan"
else
  echo "[ERROR] Gagal menyiapkan containerd. Memeriksa log dan mencoba lagi."
  exit 1
fi
echo "[task] containerd siap"

echo "[task] Menginstal Kubernetes (kubeadm, kubelet, kubectl)"
echo "[task] membuat folder keyrings"
sudo mkdir -p -m 755 /etc/apt/keyrings
echo "[task] mendownload kunci GPG Kubernetes"
if curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | gpg --dearmor --no-tty | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.gpg > /dev/null; then
  echo "[info] Kunci GPG Kubernetes berhasil diunduh dan disimpan"
else
  echo "[ERROR] Gagal mengunduh kunci GPG Kubernetes. Memeriksa log dan mencoba lagi."
  exit 1
fi
echo "[task] menambahkan repositori Kubernetes ke apt"
if echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list; then
  echo "[info] Repositori Kubernetes berhasil ditambahkan"
else
  echo "[ERROR] Gagal menambahkan repositori Kubernetes. Memeriksa log dan mencoba lagi."
  exit 1
fi
echo "[task] Mengupdate apt dan menginstal Kubernetes (kubeadm, kubelet, kubectl)"
if sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl; then
  echo "[info] Kubernetes (kubeadm, kubelet, kubectl) berhasil diinstal"
else
  echo "[ERROR] Gagal menginstal Kubernetes. Memeriksa log dan mencoba lagi."
  exit 1
fi

if sudo apt-mark hold kubelet kubeadm kubectl; then
  echo "[info] Paket Kubernetes berhasil ditandai untuk tidak di-upgrade"
else
  echo "[ERROR] Gagal menandai paket Kubernetes. Memeriksa log dan mencoba lagi."
  exit 1
fi

# Validate installation
if ! command -v kubeadm &> /dev/null || ! command -v kubelet &> /dev/null || ! command -v kubectl &> /dev/null; then
  echo "[ERROR] Kubernetes components not installed correctly."
  exit 1
fi
sudo systemctl enable --now kubelet
EOF
    else
      ssh ubuntu@$IP 'bash -euo pipefail -s' <<'EOF'
echo "[task] Menonaktifkan swap"
sudo swapoff -a; sudo sed -i '/swap/d' /etc/fstab

echo "[task] Menonaktifkan firewall (UFW)"
sudo systemctl disable --now ufw

echo "[task] Memuat modul kernel yang dibutuhkan"
cat <<MOD | sudo tee /etc/modules-load.d/containerd.conf > /dev/null
overlay
br_netfilter
MOD
sudo modprobe overlay && sudo modprobe br_netfilter

echo "[task] Mengkonfigurasi sysctl untuk Kubernetes"
cat <<SYSCTL | sudo tee /etc/sysctl.d/kubernetes.conf > /dev/null
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
SYSCTL
sudo sysctl --system

echo "[task] Menginstal containerd dan dependensinya"
sudo apt update && sudo apt install -y containerd apt-transport-https gpg ca-certificates

echo "[task] Menyiapkan containerd"
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo systemctl restart containerd && sudo systemctl enable containerd

echo "[task] Menginstal Kubernetes (kubeadm, kubelet, kubectl)"
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Validate installation
if ! command -v kubeadm &> /dev/null || ! command -v kubelet &> /dev/null || ! command -v kubectl &> /dev/null; then
  echo "[ERROR] Kubernetes components not installed correctly."
  exit 1
fi
sudo systemctl enable --now kubelet
EOF
    fi
  fi
done

log "[task] Inisialisasi cluster Kubernetes di $MASTER1_IP"
if [[ "$SELF_IP" == "$MASTER1_IP" ]]; then
  log "[task] Menjalankan kubeadm init di master pertama"
  read -p "Masukkan CIDR jaringan untuk pod (default: 172.244.0.0/16): " POD_CIDR
  POD_CIDR=${POD_CIDR:-172.244.0.0/16}

  log "[task] Menggunakan CIDR jaringan untuk pod: $POD_CIDR"
  sudo kubeadm init --control-plane-endpoint="$CLUSTER_ENDPOINT:6443" \
    --upload-certs --apiserver-advertise-address=$MASTER1_IP \
    --pod-network-cidr=$POD_CIDR
  log "[task] Menyiapkan file konfigurasi kubeconfig"
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
else
  log "[task] Menjalankan kubeadm init di master pertama melalui SSH"
  retry_ssh "$SSH_CMD@$MASTER1_IP 'sudo kubeadm init --control-plane-endpoint=$CLUSTER_ENDPOINT:6443 --upload-certs --apiserver-advertise-address=$MASTER1_IP --pod-network-cidr=172.244.0.0/16'"
  $SSH_CMD@$MASTER1_IP "mkdir -p $HOME/.kube"
  $SSH_CMD@$MASTER1_IP "sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config"
  $SSH_CMD@$MASTER1_IP "sudo chown $(id -u):$(id -g) $HOME/.kube/config"
  $SSH_CMD@$MASTER1_IP "$KUBECTL_CMD apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml"
fi

JOIN_CMD=$($SSH_CMD@$MASTER1_IP "kubeadm token create --print-join-command")
CERT_KEY=$($SSH_CMD@$MASTER1_IP "kubeadm init phase upload-certs --upload-certs 2>/dev/null | grep -oP '(?<=--certificate-key )\\S+'")

log "[task] Men-join node ke cluster"
for IP in "${!NODE_ROLE[@]}"; do
  if [[ "$IP" == "$SELF_IP" && "${NODE_ROLE[$IP]}" == "master" ]]; then
    continue
  fi

  if [[ "${NODE_ROLE[$IP]}" == "master" ]]; then
    $SSH_CMD@$MASTER1_IP $SSH_CMD@$IP "$JOIN_CMD --control-plane --certificate-key $CERT_KEY"
  elif [[ "${NODE_ROLE[$IP]}" == "worker" ]]; then
    $SSH_CMD@$MASTER1_IP $SSH_CMD@$IP "$JOIN_CMD"
  fi
done

log "[task] Validasi akhir: Menampilkan status node-node"
$KUBECTL_CMD get nodes -o wide

log "[task] Memastikan semua node dalam status Ready"
READY_NODES=$($KUBECTL_CMD get nodes --no-headers | grep -c " Ready ")
if [[ $READY_NODES -ne $TOTAL_NODE ]]; then
  fail "Tidak semua node dalam status Ready. Periksa konfigurasi cluster."
  exit 1
fi
log "Semua node dalam status Ready."

log "[task] Setup Kubernetes Cluster selesai!"