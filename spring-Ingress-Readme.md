# 🟢 Spring Boot와 Docker, Kubernetes Ingress를 활용한 웹 애플리케이션 배포 및 외부 통신 테스트

---

## 📌 프로젝트 목표

1. **Spring Boot 앱 Docker Hub 업로드**
   - DB 연동 없는 단순 GET/POST 앱
   - index.html 제공
2. **Kubernetes 리소스 생성**
   - 선언형 YAML 기반
   - Pod 3개 생성 및 관리 (생성/삭제/실행 확인 가능)
   - 외부 통신 확인 (브라우저 / Postman)
3. **서버 환경 구성**
   - myserver02 복제 → myserver03 생성
   - SSH 연결 설정
4. **Ingress 설치**
   - 기존 rule에 앱 추가
   - curl로 외부 접속 확인

---

## 🏗 프로젝트 구조

```
mini-project/
├── Dockerfile
├── spring-cluster.yaml
├── spring-ingress.yaml
├── src/
│   └── main/java/edu/ce/fisa/controller/Controller.java
└── README.md
```

---

## 📊 프로젝트 구조도

> (구조도 이미지 삽입 예정)

---

## ⚙ 프로젝트 단계

### 1️⃣ Spring Boot 앱 생성

Spring Boot 앱은 `/app2/get`, `/app2/post` 두 개의 엔드포인트를 제공합니다.

```java
package edu.ce.fisa.controller;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("app2")
public class Controller {

    @GetMapping("/get")
    public String getReqRes() {
        return "SpringApp - get 방식";
    }

    @PostMapping("/post")
    public String postReqRes() {
        return "SpringApp - post 방식";
    }
}
```

---

### 2️⃣ Docker Hub 업로드 및 컨테이너 실행

```bash
# Dockerfile로 이미지 생성
docker build -t bootapp:1.0 .

# 이미지 확인
docker images

# 컨테이너 실행
docker run -p 8081:8081 --name bootapp -d bootapp:1.0

# 실행 중 컨테이너 확인
docker ps

# curl 테스트
curl http://127.0.0.1/index.html
```

---

### 3️⃣ Kubernetes YAML 배포

#### `spring-cluster.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: springappsample2
        image: dbin887742/springappsample2:v1
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8080
      nodePort: 30080
  type: NodePort
```

#### 리소스 생성 및 확인

```bash
kubectl apply -f spring-cluster.yaml
kubectl get pod
kubectl get svc
```

---

###  주요 포트 개념

| 용어          | 설명 |
|---------------|------------------------------------------------|
| containerPort | 컨테이너 내부에서 앱이 실제로 사용하는 포트 (기본 8080) |
| targetPort    | Service → Pod 간 트래픽 전달 포트 |
| port          | Cluster 내부에서 Service가 노출하는 포트 |
| nodePort      | 외부 접근 가능 포트 (30000~32767 범위) |

---

### 4️⃣ 포트 포워딩 & NodePort 테스트

```bash
# 포트 포워딩
kubectl port-forward service/myapp-service 8082:8081

# 로컬 브라우저 테스트
http://localhost:8082

# NodePort 테스트
curl http://<NodeIP>:30080/app2/get
```

---

### 5️⃣ myserver02 복제 및 myserver03 생성

```bash
# 호스트 네트워크 설정
sudo vim /etc/netplan/01-netcfg.yaml
sudo netplan apply

# /etc/hosts 수정
10.0.2.25 myserver03
10.0.2.20 myserver02
10.0.2.15 myserver01

# SSH 키 등록
ssh-keygen -t rsa -b 4096
ssh-copy-id ubuntu@myserver03
ssh ubuntu@myserver03
```

---

### 6️⃣ Spring Boot 앱 Ingress 설정

#### `spring-ingress.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: springappsample2
        image: dbin887742/springappsample2:v1
        ports:
        - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service2
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 8082
      targetPort: 8080
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: spring.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service2
            port:
              number: 8082
```

#### 적용 및 확인

```bash
kubectl apply -f spring-ingress.yaml
kubectl get all
curl http://spring.local/app2/get
```

---

## ✅ 결과

- Docker Hub에 Spring Boot 앱 업로드 완료  
- Minikube에서 Pod 3개, Service, Ingress 설정 완료  
- myserver02 → myserver03 SSH 연결 완료  
- Ingress를 통한 외부 접속 및 curl 테스트 성공  
