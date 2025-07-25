# AWS EKS 기본 아키텍처 및 배포 실습

## 1. AWS EKS 기본 아키텍처 구성도

```
  AWS EKS의 기본 아키텍처 구성도와 주요 구성
  요소(Pod, Node, Control Plane 등)에 대해 설명해줘.
  각 요소가 어떤 역할을 하는지도 자세히 알려줘.

  로컬 환경에서 AWS EKS 클러스터를 생성하고, 간단한
  애플리케이션을 배포하는 실습 과정을 단계별로
  알려줘.
  필요한 사전 조건과 설치 도구 목록도 함께 정리해줘.
```

### EKS 클러스터 전체 아키텍처
```
┌─────────────────────────────────────────────────────────────┐
│                    AWS EKS Cluster                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌───────────────────────────────┐   │
│  │   Control Plane │    │         Data Plane            │   │
│  │   (AWS 관리)    │    │      (사용자 관리)           │   │
│  │                 │    │                               │   │
│  │ ┌─────────────┐ │    │ ┌─────────────┐ ┌───────────┐ │   │
│  │ │API Server   │ │◄──►│ │   Node 1    │ │  Node 2   │ │   │
│  │ │etcd         │ │    │ │             │ │           │ │   │
│  │ │Scheduler    │ │    │ │ ┌─────────┐ │ │ ┌───────┐ │ │   │
│  │ │Controller   │ │    │ │ │  Pod 1  │ │ │ │ Pod 3 │ │ │   │
│  │ │Manager      │ │    │ │ │         │ │ │ │       │ │ │   │
│  │ └─────────────┘ │    │ │ └─────────┘ │ │ └───────┘ │ │   │
│  └─────────────────┘    │ │ ┌─────────┐ │ │           │ │   │
│                         │ │ │  Pod 2  │ │ │           │ │   │
│                         │ │ │         │ │ │           │ │   │
│                         │ │ └─────────┘ │ │           │ │   │
│                         │ └─────────────┘ └───────────┘ │   │
│                         └───────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 2. 주요 구성 요소별 상세 설명

### 2.1 Control Plane (컨트롤 플레인)
**역할**: 클러스터의 두뇌 역할을 하는 관리 영역
- **AWS에서 완전 관리**: 사용자가 직접 관리할 필요 없음
- **고가용성**: 여러 가용 영역에 걸쳐 자동으로 분산 배치

#### 주요 구성 요소:
1. **API Server**
   - 모든 클러스터 통신의 중앙 허브
   - kubectl 명령어와 다른 도구들의 요청을 처리
   - 인증 및 권한 부여 담당

2. **etcd**
   - 클러스터의 모든 설정 데이터와 상태 정보를 저장하는 분산 키-값 저장소
   - 클러스터의 "진실의 단일 소스" 역할

3. **Scheduler**
   - Pod를 어떤 Node에 배치할지 결정
   - 리소스 요구사항, 제약 조건, 정책 등을 고려

4. **Controller Manager**
   - 다양한 컨트롤러들을 실행
   - 원하는 상태와 현재 상태를 지속적으로 비교하고 조정

### 2.2 Data Plane (데이터 플레인)
**역할**: 실제 애플리케이션이 실행되는 작업 영역
- **사용자 관리**: EC2 인스턴스로 구성되며 사용자가 관리

#### Node (노드)
**정의**: 컨테이너화된 애플리케이션을 실행하는 물리적 또는 가상 머신

**주요 구성 요소**:
1. **kubelet**
   - Node의 에이전트 역할
   - API Server와 통신하여 Pod 실행 상태 관리
   - 컨테이너 헬스 체크 수행

2. **kube-proxy**
   - 네트워크 프록시 및 로드 밸런서
   - Service 추상화를 통한 Pod 간 통신 관리

3. **Container Runtime**
   - 실제 컨테이너를 실행하는 소프트웨어 (Docker, containerd 등)

#### Pod (파드)
**정의**: Kubernetes에서 배포 가능한 가장 작은 단위

**특징**:
- 하나 이상의 컨테이너를 포함
- 같은 Pod 내 컨테이너들은 네트워크와 스토리지를 공유
- 함께 스케줄링되고 함께 종료됨
- 각 Pod는 고유한 IP 주소를 가짐

**Pod 내부 구조**:
```
┌─────────────────────────────────┐
│             Pod                 │
├─────────────────────────────────┤
│  ┌─────────────┐ ┌────────────┐ │
│  │ Container 1 │ │Container 2 │ │
│  │             │ │            │ │
│  └─────────────┘ └────────────┘ │
│                                 │
│  공유 네트워크 (IP: 10.0.1.5)  │
│  공유 스토리지 볼륨             │
└─────────────────────────────────┘
```

### 2.3 추가 중요 구성 요소

#### Service
- Pod들에 대한 안정적인 네트워크 엔드포인트 제공
- 로드 밸런싱 및 서비스 디스커버리 기능

#### Ingress
- 클러스터 외부에서 내부 서비스로의 HTTP/HTTPS 라우팅 관리

#### ConfigMap & Secret
- 애플리케이션 설정 정보와 민감한 데이터 관리

## 3. 로컬 환경에서 AWS EKS 클러스터 생성 및 애플리케이션 배포 실습

### 3.1 사전 조건 및 필요 도구

#### 필수 사전 조건
1. **AWS 계정**: 활성화된 AWS 계정
2. **IAM 권한**: EKS, EC2, VPC 관련 권한
3. **운영 체제**: Windows 10/11, macOS, 또는 Linux

#### 필수 설치 도구 목록

1. **AWS CLI v2**
   ```bash
   # Windows (PowerShell)
   msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
   
   # macOS
   curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
   sudo installer -pkg AWSCLIV2.pkg -target /
   
   # Linux
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```

2. **kubectl**
   ```bash
   # Windows (Chocolatey)
   choco install kubernetes-cli
   
   # macOS (Homebrew)
   brew install kubectl
   
   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```

3. **eksctl**
   ```bash
   # Windows (Chocolatey)
   choco install eksctl
   
   # macOS (Homebrew)
   brew tap weaveworks/tap
   brew install weaveworks/tap/eksctl
   
   # Linux
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   ```

4. **Docker Desktop** (선택사항, 로컬 개발용)
   - [Docker 공식 웹사이트](https://www.docker.com/products/docker-desktop/)에서 다운로드

### 3.2 단계별 실습 과정

#### 단계 1: AWS 자격 증명 설정
```bash
# AWS 자격 증명 구성
aws configure
# AWS Access Key ID: [여러분의 Access Key]
# AWS Secret Access Key: [여러분의 Secret Key]
# Default region name: ap-northeast-2
# Default output format: json

# 구성 확인
aws sts get-caller-identity
```

#### 단계 2: EKS 클러스터 생성
```bash
# 클러스터 생성 (약 15-20분 소요)
eksctl create cluster \
  --name my-eks-cluster \
  --version 1.24 \
  --region ap-northeast-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# 클러스터 생성 상태 확인
aws eks list-clusters --region ap-northeast-2
```

#### 단계 3: kubectl 설정
```bash
# kubeconfig 업데이트
aws eks update-kubeconfig --region ap-northeast-2 --name my-eks-cluster

# 클러스터 연결 확인
kubectl get nodes
kubectl get pods --all-namespaces
```

#### 단계 4: 간단한 애플리케이션 배포

1. **Nginx 애플리케이션 배포**
```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

2. **애플리케이션 배포 실행**
```bash
# 배포 파일 적용
kubectl apply -f nginx-deployment.yaml

# 배포 상태 확인
kubectl get deployments
kubectl get pods
kubectl get services

# LoadBalancer 외부 IP 확인 (몇 분 소요)
kubectl get service nginx-service
```

#### 단계 5: 애플리케이션 접속 확인
```bash
# 서비스 상세 정보 확인
kubectl describe service nginx-service

# 외부 IP로 브라우저에서 접속 확인
# http://[EXTERNAL-IP]
```

#### 단계 6: 모니터링 및 로그 확인
```bash
# Pod 로그 확인
kubectl logs -l app=nginx

# Pod 상세 정보 확인
kubectl describe pod [POD-NAME]

# 리소스 사용량 확인
kubectl top nodes
kubectl top pods
```

### 3.3 실습 후 정리 (비용 절약)
```bash
# 애플리케이션 삭제
kubectl delete -f nginx-deployment.yaml

# 클러스터 삭제 (모든 리소스 제거)
eksctl delete cluster --name my-eks-cluster --region ap-northeast-2
```

## 4. 주요 개념 정리

### 4.1 EKS의 장점
- **관리형 서비스**: Control Plane을 AWS가 관리
- **고가용성**: 여러 AZ에 걸친 자동 분산
- **보안**: AWS IAM과의 네이티브 통합
- **확장성**: Auto Scaling 지원

### 4.2 비용 최적화 팁
- **클러스터 사용 후 즉시 삭제**: 불필요한 비용 발생 방지
- **적절한 인스턴스 타입 선택**: t3.micro/small로 시작
- **Spot 인스턴스 활용**: 개발/테스트 환경에서 비용 절약

### 4.3 다음 단계 학습 방향
1. **Helm**: Kubernetes 패키지 매니저
2. **Ingress Controller**: 고급 라우팅
3. **Monitoring**: Prometheus, Grafana
4. **CI/CD**: GitHub Actions, AWS CodePipeline
5. **Service Mesh**: Istio, AWS App Mesh