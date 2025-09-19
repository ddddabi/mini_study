# 🟢 Spring Boot와 Docker, Kubernetes Ingress를 활용한 웹 애플리케이션 배포 및 외부 통신 테스트

<br>


## 📌 프로젝트 목표  

> Spring Boot 애플리케이션을 Docker와 Kubernetes를 이용해 컨테이너화한 후, 클러스터 내에서 다중 Pod로 배포 및 관리한다.  
> Ingress를 통해 클러스터 외부 트래픽을 내부 서비스로 라우팅하고 로드밸런싱한다.  

<br>

## 🛠 프로젝트 진행 흐름 (Flow)  

1. [**Spring Boot 앱 생성**](#1️⃣-Spring-Boot-앱-생성)
   - 간단한 GET/POST API 및 index.html 페이지 제공 

2. [**Docker 빌드 & Hub 업로드**](#2️⃣-Docker-Hub-업로드-및-컨테이너-실행)
   - Dockerfile 기반으로 이미지 생성 후 Docker Hub에 업로드  

3. [**Kubernetes 리소스 배포**](#3️⃣-Kubernetes-YAML-배포)  
   - Deployment(3 replicas) 및 Service(NodePort) 생성  
   - Pod/Service 상태 확인 및 통신 테스트  

4. [**외부 통신 확인**](#4️⃣-포트-포워딩--nodeport-테스트)
   - 포트 포워딩, NodePort 방식으로 curl 테스트
     
5. [**Ingress 설정 및 검증**](#5️⃣-Spring-Boot-앱-Ingress-설정)
   - Ingress rule에 애플리케이션 추가  
   - Host 기반 라우팅(`spring.local`)으로 외부 접근 및 로드밸런싱 확인  

<br>

## 📊 프로젝트 아키텍처

 <img width="2341" height="783" alt="Image" src="https://github.com/user-attachments/assets/3b01671c-7437-4940-9940-c5de0dde791f" />

<br>

---

<br>

## ⚙ 프로젝트 단계

### 1️⃣ Spring Boot 앱 생성

- Spring Boot 앱에서 `/app2/get`, `/app2/post` 두 개의 엔드포인트를 제공

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

<br>
<br>

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

<br>
<br>

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
        image: {dockerhub_id}/{docker_image_name}:v1
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

<br>

###  주요 포트 개념

| 용어          | 설명 |
|---------------|------------------------------------------------|
| containerPort | 컨테이너 내부에서 앱이 실제로 사용하는 포트 |
| targetPort    | Service → Pod 간 트래픽 전달 포트 |
| port          | Cluster 내부에서 Service가 노출하는 포트 |
| nodePort      | 외부 접근 가능 포트 (30000~32767 범위) |

<br>
<br>

### 4️⃣ 포트 포워딩 & NodePort 테스트

```bash
# 포트 포워딩
kubectl port-forward service/myapp-service 8082:8081

# 로컬 브라우저 테스트
http://localhost:8082

# NodePort 테스트
curl http://<NodeIP>:30080/app2/get
```

<br>
<br>

### 5️⃣ Spring Boot 앱 Ingress 설정

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
        image: {dockerhub_id}/{docker_image_name}:v1
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
> <img width="664" height="42" alt="Image" src="https://github.com/user-attachments/assets/128ffd8a-5828-42ec-860f-3f3dee7c4a8d" />

<br>
<br>

## ✅ 결과

- Docker Hub에 Spring Boot 앱 업로드 완료  
- Minikube에서 Pod 3개, Service, Ingress 설정 완료  
- myserver02 → myserver03 SSH 연결 완료  
- Ingress를 통한 외부 접속 및 curl 테스트 성공

