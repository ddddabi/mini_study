# 📘 Kubernetes 멀티 노드 클러스터 환경 구축
 
Ubuntu 24.04 환경에서 **멀티노드 Kubernetes 클러스터**를 구축  
**Nginx Ingress Controller**를 통한 서비스 노출 설정

---

## 📑 프로젝트 개요
- 3대의 Ubuntu 서버 (Master + Worker 2대)
- 컨테이너 실행 환경: `containerd`
- Kubernetes v1.30
- Nginx Ingress Controller

---

## ⚙️ 1. IP 및 Hostname 설정

### ✅ Netplan 설정
```bash
sudo nano /etc/netplan/01-netcfg.yaml
sudo netplan apply
ip a
```

- **myserver01**: `10.0.2.15`  
- **myserver02**: `10.0.2.20`  
- **myserver03**: `10.0.2.25`  

📷 네트워크 포트포워딩  
<img width="952" height="687" alt="image" src="https://github.com/user-attachments/assets/43a16301-f0d4-4192-8eb1-d30a6feb96de" />

---

### ✅ Hostname 변경
```bash
sudo hostnamectl set-hostname myserver01
cat /etc/hostname
sudo reboot
```

- `/etc/hosts` 파일에 각 서버 정보 등록  

hosts 파일 예시  
<img width="448" height="205" alt="Image" src="https://github.com/user-attachments/assets/e8a35273-2f92-443a-9793-026084992cfa" />

---

## 🔑 2. SSH 키 설정

```bash
# .ssh 초기화 (선택)
rm -rf /home/ubuntu/.ssh

# SSH 키(공개키, 개인키)를 생성하는 명령어
ssh-keygen -t rsa -b 4096

# SSH 키 복사 (myserver01 기준, 다른 서버들도 똑같이 해줘야 함)
ssh-copy-id ubuntu@myserver02
ssh-copy-id ubuntu@myserver03
```

비밀번호 없이 접속 확인:
```bash
ssh ubuntu@myserver02
```

---

## 🌐 3. 네트워크 설정

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

---

## 📦 4. Kubernetes 설치

```bash
sudo apt update && sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd

# GPG 키 및 저장소 등록
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key |   sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" |   sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update --allow-unauthenticated
sudo apt-get install -y --allow-unauthenticated kubelet kubeadm kubectl
```

---

## 🖥️ 5. 마스터 노드 초기화

```bash
sudo kubeadm config images pull --cri-socket unix:///run/containerd/containerd.sock

sudo kubeadm init   --apiserver-advertise-address=10.0.2.15   --pod-network-cidr=192.168.0.0/16   --cri-socket /run/containerd/containerd.sock
```

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes -o wide
```

---

## 🔗 6. 워커 노드 Join
- 마스터 노드에서 init 시에 생성되는 token 및 해시 값 워커 노드에 적용

```bash
scp -p ubuntu@myserver01:~/.kube/config ~/.kube/config
sudo kubeadm join 10.0.2.15:6443 --token <토큰값> --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 🌍 7. Nginx Ingress + Deployment

<details>
<summary>📂 nginx yaml 설정 파일 nginx3.yaml</summary>

```yaml
# nginx3-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx3
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx3
  template:
    metadata:
      labels:
        app: nginx3
    spec:
      containers:
      - name: nginx3
        image: nginx:latest
        ports:
        - containerPort: 80

---

# nginx3-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx3
  namespace: default
spec:
  type: NodePort
  selector:
    app: nginx3
  ports:
    - protocol: TCP
      port: 8083        # ClusterIP 내부 포트
      targetPort: 80    # 컨테이너 포트
      nodePort: 30083   # 외부 접속용 NodePort (30000~32767)

---

# nginx3-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx3-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx3
            port:
              number: 8083
```
</details>

- `nginx:latest` → `registry.k8s.io/nginx` 로 변경 권장  
- yaml 파일 수정 시 제거 후 적용
```bash
kubectl delete -f nginx3
kubectl apply -f nginx3
```
- nginx Pod가 어느 노드에 배포됐는지 확인
```bash
kubectl get pods -n ingress-nginx -o wide
```
- Ingress IP 및 포트 확인
```bash
kubectl describe ingress
```
- `/etc/hosts` 수정 후 접근 가능  

hosts 파일 설정  
<img width="478" height="215" alt="Image" src="https://github.com/user-attachments/assets/d08f2ea5-f758-4cef-abdf-1604e179d839" />

---

## 🧪 8. 테스트

```bash
curl myapp.local
```

테스트 결과  
<img width="1135" height="587" alt="image" src="https://github.com/user-attachments/assets/55e922c1-845b-480b-ae2b-c1b8d2be721a" />

---

## 📌 정리
- Ubuntu 24.04 환경에서 멀티노드 K8s 클러스터 구축  
- Ingress Controller를 통한 서비스 라우팅 확인  
