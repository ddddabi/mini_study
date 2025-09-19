# 🚀 YAML 파일로 Spring App 배포하기

## 📂 1. 작업 디렉토리 이동

``` bash
$ pwd
/home/ubuntu/05.k8s
```

------------------------------------------------------------------------

## 📝 2. `spring-cluster.yaml` 파일 생성

``` bash
$ vi spring-cluster.yaml    # yaml 파일 작성 후 저장
```

### ✨ spring-cluster.yaml

``` yaml
# ---------------------- Deployment ----------------------
apiVersion: apps/v1                               # API 버전 (Deployment 리소스는 apps/v1 사용)
kind: Deployment                                  # 리소스 종류: Deployment
metadata:
  name: myapp-deployment                          # Deployment 이름
spec:
  replicas: 3                                     # 생성할 Pod 개수 (복제본 수)
  selector:                                       # 어떤 Pod들을 관리할지 선택 기준
    matchLabels:
      app: myapp                                  # "app=myapp" 라벨을 가진 Pod 선택
  template:                                       # 위 matchLabels 조건에 맞는 Pod 템플릿 정의
    metadata:
      labels:
        app: myapp                                # Pod에 라벨 부여 (Service와 연결될 기준)
    spec:
      containers:
      - name: springappsample2                    # 컨테이너 이름
        image: dbin887742/springappsample2:v1     # 실행할 Docker Hub 이미지
        ports:
        - containerPort: 8080                     # 컨테이너 내부에서 사용하는 포트

---
# ---------------------- Service ----------------------
apiVersion: v1                                    # API 버전 (Service 리소스는 v1 사용)
kind: Service                                     # 리소스 종류: Service
metadata:
  name: myapp-service                             # Service 이름
spec:
  selector:
    app: myapp                                    # "app=myapp" 라벨을 가진 Pod 대상으로 트래픽 전달
  ports:
    - protocol: TCP                               # 프로토콜
      port: 8081                                  # 클러스터 내부에서 Service가 노출하는 포트
      targetPort: 8080                            # Pod(컨테이너)가 실제로 사용하는 포트
      nodePort: 30080                             # 클러스터 외부(Node IP)에서 접근할 수 있는 포트 (30000~32767 범위)
  type: NodePort                                  # NodePort 타입: NodeIP:nodePort로 외부 접근 가능
```

------------------------------------------------------------------------

## 🚀 3. 리소스 생성

``` bash
$ kubectl apply -f spring-cluster.yaml

deployment.apps/myapp-deployment created
service/myapp-service created
```

------------------------------------------------------------------------

## 📋 4. Pod 확인

``` bash
$ kubectl get pod

NAME                                READY   STATUS    RESTARTS   AGE
myapp-deployment-6d5496c75c-78blm   1/1     Running   0          42s
myapp-deployment-6d5496c75c-c655d   1/1     Running   0          42s
myapp-deployment-6d5496c75c-rhxb4   1/1     Running   0          42s
```

------------------------------------------------------------------------

## 📖 주요 포트 개념

  ------------------------------------------------------------------------
  용어            설명
  --------------- --------------------------------------------------------
  containerPort   컨테이너 내부에서 앱이 실제로 듣는 포트 (Spring Boot
                  기본 8080)

  targetPort      Service가 컨테이너로 트래픽을 전달할 때 연결하는 포트
                  (보통 containerPort와 동일)

  port            ClusterIP/Service 내부에서 사용하는 포트

  nodePort        외부에서 Node IP + 이 포트로 접속할 수 있는 포트
                  (30000\~32767)
  ------------------------------------------------------------------------

⚠️ **중요:** `targetPort`와 `containerPort`는 같아야 정상적으로 연결됨.

------------------------------------------------------------------------

## 🌐 5. 포트 포워딩

``` bash
$ kubectl port-forward service/myapp-service 8082:8081

Forwarding from 127.0.0.1:8082 -> 8080
Forwarding from [::1]:8082 -> 8080
```

👉 로컬 브라우저에서 접속:

    http://localhost:8082

------------------------------------------------------------------------

## 🐳 6. Docker Pull

``` bash
$ docker pull dbin887742/springappsample2:v1
```

------------------------------------------------------------------------

## 🌍 7. NodePort 외부 접속

NodePort 방식을 사용하면 외부에서도 접근 가능합니다.

``` bash
curl http://192.168.49.2:30080/app2/get
```

> port-forward는 로컬에서만 접속 가능하지만, NodePort는 외부 접속도
> 허용합니다.

------------------------------------------------------------------------

## ✅ 최종 테스트

브라우저에서 아래 주소로 접속:

    http://localhost:8080/app2/get

![결과
이미지](https://github.com/user-attachments/assets/62b1aa52-31b1-4ab9-b314-194acbd64744)




# spring app에 ingress 적용
