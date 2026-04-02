# Kubernetes N-Tier 실습

## 1. 프로젝트 목적
본 프로젝트의 목적은 실제 N-Tier 환경(마스터 노드 1대, 워커 노드 2대)을 리눅스 VM으로 직접 구성하여 Kubernetes 클러스터 구축 및 노드 간 통신을 실습하는 데 있다.

이를 통해 minikube 기반의 단일/로컬 테스트 환경을 넘어, 실무에 가까운 Kubernetes 운영 방식과 배포 구조를 이해하는 것이 목표이다.

## 2. 프로젝트 구성
### 노드 및 네트워크 구성

노드는 Ubuntu 24.04 기반 리눅스 VM 3대로 구성하였다.

- **마스터 노드:** 10.0.2.15 /24
- **워커 노드 1:** 10.0.2.20 /24
- **워커 노드 2:** 10.0.2.25 /24
- **네트워크 인터페이스(CNI):** Calico CNI

> pod간 통신 타입은 Node Port를 사용했다.

### 애플리케이션 구성

- **mysql** Pod 1개
- **springboot** Pod 2개 (Deployment replicas=2)
- **springboot 앱 이미지**: MySQL과의 연동을 확인하기 위한 간단한 스프링 부트 앱을 개발
    - **이미지 주소:** https://hub.docker.com/repository/docker/zaixian5/springboot-app/general

### 아키텍처

```text
                                +---------------------------+
                                |        Master Node        |
                                |         10.0.2.15         |
                                |   (Control Plane / API)   |
                                +-------------+-------------+
                                              |
                    +------+-----------------------------------+------+
                    |          Calico CNI Pod Network                 |
                    +------+-----------------------------------+------+
                        |                                       |
        +-----------+------------+                  +-----------+------------+
        |        Worker Node 1   |                  |        Worker Node 2   |
        |         10.0.2.20      |                  |         10.0.2.25      |
        |  springboot Pod (1/2)  |                  |  springboot Pod (2/2)  |
        +------------------------+                  |      mysql Pod (1)     |
                                                    +------------------------+
```

## 3. 주요 개념 정리

### 1) k8s 클러스터와 노드란?
- **k8s 클러스터:** 컨테이너 애플리케이션을 관리하는 전체 묶음이다.
- **노드:** 클러스터를 구성하는 개별 서버(VM 또는 물리 서버)이다.
    - **마스터 노드(컨트롤 플레인)**는 제어를 담당하고, **워커 노드**는 실제 Pod를 실행하는 역할이다.
    - 관리자는 마스터 노드에 접속하여 관리, 사용자는 워커 노트에 접속하여 서비스를 이용한다.

### 2) Pod란?
- Pod는 Kubernetes에서 컨테이너를 실행하는 가장 작은 배포 단위이다.
- 일반적으로 하나의 애플리케이션 컨테이너를 Pod 단위로 실행한다.

### 3) Deployment란?
- Deployment는 Pod를 선언적으로 생성하고, 개수와 업데이트를 관리하는 리소스이다.
- replicas 값을 통해 동일한 Pod를 원하는 개수로 유지할 수 있다.
    - replicas가 설정되어 있다면 pod를 삭제해도 자동 재생성되어 개수를 유지한다.

### 4) Service란?
- Service는 Pod 집합에 고정된 접근 경로(IP/DNS)를 제공하는 네트워크 리소스이다.
    - pod의ip와 port만으로는 네트워크 접근을 할 수 없으며, 반드시 service가 있어야 한다.
- Pod가 재생성되어도 동일한 이름으로 안정적으로 접근할 수 있게 해준다.

### 5) NodePort와 ClusterIP의 차이
- **ClusterIP:** 클러스터 내부에서만 접근 가능한 기본 Service 타입이다.
- **NodePort:** 각 노드의 특정 포트로 외부 접근이 가능하도록 하는 Service 타입이다.

### 6) Ingress란?
- Ingress는 HTTP/HTTPS 요청을 도메인/경로 기준으로 내부 Service에 라우팅하는 리소스이다.
    - 즉 ingress를 사용하려면 service를 ClusterIP로 만들어야 함. ingress가 외부 네트워크를 내부 ClusterIP로 라우팅 해주기 때문
- Ingress를 사용하면 여러 서비스를 하나의 진입점으로 관리할 수 있다.

### 7) CNI란?
- CNI는 Kubernetes에서 Pod 네트워크를 연결하기 위한 표준 인터페이스이다.
- CNI 플러그인(예: Calico)은 Pod 간 통신과 네트워크 정책을 처리한다.

## 4. 실습 스크립트
### 1) mysql-deploysvc.yaml
> mysql pod와 service를 만드는 스크립트

```yaml
# 1. MySQL 배포 설정 (Deployment)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  labels:
    app: mysql
spec:
  replicas: 1 # DB는 데이터 동기화 문제로 보통 1개로 시작
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0 # 사용할 MySQL 버전 이미지
        ports:
        - containerPort: 3306 # MySQL 기본 포트
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "root" # 실제 운영 시에는 'Secret'을 사용해야 하지만, 여기선 직접 입력
        - name: MYSQL_DATABASE
          value: "fisa" # 시작 시 자동으로 생성할 DB 이름

---

# 2. MySQL 접속 설정 (Service)
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306      # 서비스가 노출할 포트
      targetPort: 3306 # 컨테이너로 전달할 포트
  type: ClusterIP # 클러스터 내부에서만 접근 가능하도록 설정
```

### 2) springboot-deploysvc.yaml
> 스프링 부트 앱 이미지를 가져와 pod와 service를 만드는 스크립트

```yaml
# 1. 자바 애플리케이션 배포 (Deployment)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-app-deployment
spec:
  replicas: 2 # 가용성을 위해 레플리카 2개 설정
  selector:
    matchLabels:
      app: springboot-app
  template:
    metadata:
      labels:
        app: springboot-app
    spec:
      containers:
      - name: springboot-app
        image: zaixian5/springboot-app:latest # 도커 허브 이미지 주소
        ports:
        - containerPort: 8080 # 스프링 부트 기본 포트

        # DB 연결 정보 (application.properties의 ${...} 자리표시자와 매칭)
        # env:
        #   - name: MYSQL_HOST
        #     value: "mysql-service"
        #   - name: MYSQL_PORT
        #     value: "3306"
        #   - name: MYSQL_DATABASE
        #     value: "fisa"
        #   - name: MYSQL_USER
        #     value: "root"
        #   - name: MYSQL_PASSWORD

        envFrom: # DB 연결 정보를 configmap으로 분리
          - configMapRef:
              name: app-config
        env:
          - name: MYSQL_PASSWORD
            value: "root" # 비밀번호는 Secret으로 분리 권장 (여기서는 그대로 둠)

---

# 2. 외부에서 웹 접속을 위한 설정 (Service)
apiVersion: v1
kind: Service
metadata:
  name: springboot-app-service
spec:
  selector:
    app: springboot-app
  ports:
    - protocol: TCP
      port: 80          # 서비스의 포트 (브라우저 접속용)
      targetPort: 8080  # 컨테이너 안의 스프링 포트
      nodePort: 30000   # ClusterIP룰 사용할 경우 삭제할 것
  type: NodePort # 외부 브라우저에서 접속하기 위해 NodePort로 설정(테스트 용)
  # type: ClusterIP # ingress로 접속하려면 이 서비스를 내부 클러스터에서만 접속 가능한 ClusterIP로 설정해야 함
```

### 3) configmap.yaml
> mysql 관련 환경변수 일부를 정의하는 스크립트(ConfigMap)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  MYSQL_HOST: mysql-service
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: "fisa"
  MYSQL_USER: "root"
```

### 4) ingress.yaml (실제로 사용하지 않음)
> ingress를 적용하기 위한 스크립트\
> 실제로는 ingress 대신 node port만 사용하여 실습하였다.(트러블 슈팅 참고)

```yaml
apiVersion: networking.k8s.io/v1 # Ingress 리소스 API 버전
kind: Ingress # Ingress: 외부 요청을 내부 Service로 라우팅하는 리소스
metadata:
  name: springboot-app-ingress # Ingress 리소스 이름
  annotations: # ingress controller가 동작을 제어하는 설정
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx # nginx ingress controller 사용
  rules:
    - host: springboot.local # 이 도메인으로 들어온 요청 처리
      http:
        paths:
          - path: / # 루트 경로 이하 전체 매칭
            pathType: Prefix # /emp, /emp/deptall 같은 하위 경로도 포함
            backend:
              service:
                name: springboot-app-service # 요청을 전달할 Service 이름
                port:
                  number: 80 # Service 포트
```

## 5. 실습 명령
### mysql deployment 생성
```bash
kubectl apply -f mysql-deploysvc.yaml
```

### spring boot deployment 생성
```bash
kubectl apply -f springboot-deploysvc.yaml
```

### ConfigMap 적용
```bash
kubectl apply -f configmap.yaml
```

### ingress 적용(실제 사용하지 않음)
```bash
# ingress controller(ingress-nginx 사용)이 먼저 설치되어야 함
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# 설치 확인
kubectl get all -n ingress-nginx
kubectl get pods -n ingress-nginx

# ingress.yaml 적용
kubectl apply -f ingress.yaml
```

## 6. 결과 화면

### `kubectl get all -o wide`로 생성된 객체 확인
<img width="1675" height="677" alt="getall" src="https://github.com/user-attachments/assets/cb96047b-ee24-496c-9de1-9a9d6aec0dd3" />

### spring boot와 mysql 통신 확인
<img width="955" height="80" alt="curl" src="https://github.com/user-attachments/assets/fa8f16f2-5a10-4c32-b5cc-ddf0938e0f94" />

## 7. 트러블 슈팅

### Ingress 사용의 어려움
> minikube와 실제 kubernetes와의 환경 차이로 발생한 문제

- minikube 환경에서는 ingress addon과 위의 ingress.yaml 스크립트 만으로 ingress 통신이 가능했다.
- 그러나 본 실습 환경(직접 구성한 VM 기반 클러스터)에서는 `ingress-nginx-controller` Service가 `LoadBalancer` 타입인데 실제 실습 환경(베어메탈)에 로드밸런싱 제공자가 없어 `EXTERNAL-IP`가 `pending` 상태로 남아 외부 진입 주소가 자동 생성되지 않았다.
<img width="1845" height="296" alt="trouble" src="https://github.com/user-attachments/assets/75fc267a-6322-4354-952d-aed878db634f" />


- AWS ALB/NLB와 같은 로드밸런서를 사용하거나 MetalB를 사용하여 이러한 문제를 해결할 수 있지만 본 실습에서는 생략하였다.
