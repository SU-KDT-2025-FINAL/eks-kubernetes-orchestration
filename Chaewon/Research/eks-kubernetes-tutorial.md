# EKS 및 쿠버네티스 오케스트레이션 가이드

이 가이드는 AWS EKS(Elastic Kubernetes Service) 클러스터를 설정하고 관리하는 방법을 단계별로 설명합니다. 맥북 환경에 맞춘 설치 지침, 오류 처리, 시각적 보조자료, 그리고 비용 절감을 위한 리소스 정리 방법을 포함합니다. 초보자도 따라 할 수 있도록 개념과 실습을 균형 있게 구성했습니다.

---

## 1단계: 준비 및 환경 구성 

### 1.1 개념 설명
- **IAM 사용자/역할**: AWS 리소스에 접근하는 사용자(Identity)와 EKS Control Plane이 사용하는 역할(Role)을 분리합니다. 최소 권한 원칙(Least Privilege)을 적용하여 필요한 권한만 부여합니다.
  - 예: `eksctl`로 클러스터를 생성하려면 `AmazonEKSClusterPolicy`, `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly` 정책이 필요.
- **도구 비교**:
  - `eksctl`: EKS 클러스터 생성 및 NodeGroup 관리를 자동화.
  - `AWS CLI`: IAM, VPC, ECR 등 세부 리소스 관리.
  - `kubectl`: 쿠버네티스 API 호출로 클러스터 내부 리소스 관리.
- **시각적 보조자료**:
  ```
  [EKS 아키텍처]
  Control Plane (AWS 관리) <--> Worker Nodes (EC2)
  |                       |
  IAM Role              Pods (애플리케이션 실행)
  ```

### 1.2 실습 준비 (맥북)
#### 1.2.1 AWS IAM 사용자 생성
1. AWS 콘솔에서 IAM 사용자 생성:
   - 이름: `eks-user` (예시)
   - 권한: 테스트용으로 아래 최소 정책 부여 (프로덕션에서는 `AdministratorAccess` 사용 금지):
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {"Effect": "Allow", "Action": "eks:*", "Resource": "*"},
         {"Effect": "Allow", "Action": "ec2:*", "Resource": "*"},
         {"Effect": "Allow", "Action": "iam:PassRole", "Resource": "*"},
         {"Effect": "Allow", "Action": "ecr:ReadOnlyAccess", "Resource": "*"}
       ]
     }
     ```
2. **Access Key ID**와 **Secret Access Key**를 저장하세요.

#### 1.2.2 로컬 도구 설치 (맥북)
```bash
# AWS CLI v2 설치 확인
aws --version
# 설치 (필요 시)
brew install awscli
# AWS CLI 설정
aws configure
# 입력: Access Key ID, Secret Access Key, region (ap-northeast-2), output (json)

# eksctl 설치
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version

# kubectl 설치
brew install kubectl
kubectl version --client

# Helm 설치
brew install helm
helm version
```

#### 1.2.3 기본 네트워크 확인
- AWS 콘솔 → VPC 대시보드 → 기본 VPC, 서브넷, 보안 그룹(SG) 확인.
- `eksctl`은 기본 VPC를 자동 사용. 별도 VPC 생성은 선택 사항.

#### 1.2.4 문제 해결
- **오류**: `aws configure`에서 "InvalidAccessKeyId"
  - **해결**: Access Key ID와 Secret Access Key를 다시 확인. 비활성화된 키인지 점검.
- **오류**: `brew` 명령어 실패
  - **해결**: `brew update` 실행 후 재시도.

---

## 2단계: EKS 클러스터 생성

### 2.1 개념 설명
- **Control Plane**: AWS가 관리하며, 고가용성을 위해 여러 AZ에 분산.
- **Worker Node**: EC2 인스턴스에서 `kubelet`이 실행되어 Pod를 구동.
- **NodeGroup**: EC2 그룹 단위로 관리. **Managed**는 자동 패치, **Self-managed**는 수동 관리.
- **시각적 보조자료**:
  ```
  [EKS 클러스터 구조]
  Control Plane (AWS)
    |-> API Server
    |-> Scheduler
  Worker Nodes (EC2)
    |-> Pod 1
    |-> Pod 2
  ```

### 2.2 실습: eksctl로 클러스터 생성
```bash
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-northeast-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```
- **소요 시간**: 10~15분.
- **결과**: 클러스터, VPC, 서브넷, 보안 그룹 생성. `~/.kube/config`에 클러스터 정보 추가.
- **확인**:
  ```bash
  kubectl get nodes
  eksctl get cluster --name my-eks-cluster
  ```

#### 2.2.1 문제 해결
- **오류**: "Failed to create cluster" (IAM 권한 부족)
  - **해결**: IAM 사용자에 `AmazonEKSClusterPolicy` 추가.
- **오류**: 클러스터 생성이 느림
  - **해결**: `eksctl get cluster`로 진행 상황 확인.

---

## 3단계: 네트워킹

### 3.1 개념 설명
- **AWS VPC CNI**: Pod마다 ENI(Elastic Network Interface)를 할당하여 VPC 네트워크 사용.
- **Service 종류**:
  - `ClusterIP`: 클러스터 내부 통신.
  - `NodePort`: 노드 포트를 열어 외부 노출.
  - `LoadBalancer`: AWS ELB 생성.
- **Ingress**: L7 라우팅, 호스트/경로 기반 라우팅 지원.
- **시각적 보조자료**:
  ```
  [네트워킹 흐름]
  Internet -> ALB (Ingress) -> Service (ClusterIP) -> Pod
  ```

### 3.2 실습: nginx 배포 및 Ingress 설정
#### 3.2.1 Deployment 및 ClusterIP 서비스
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 2
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
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF
kubectl get svc nginx-svc
```

#### 3.2.2 LoadBalancer로 전환
```bash
kubectl patch svc nginx-svc -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc nginx-svc -w
```
- **확인**: `EXTERNAL-IP` (ELB DNS)로 브라우저 접속.

#### 3.2.3 ALB Ingress Controller 설치
```bash
# Helm 리포지터리 추가
helm repo add eks https://aws.github.io/eks-charts
helm repo update
# ALB Ingress Controller 설치
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=true \
  --set region=ap-northeast-2
```

#### 3.2.4 Ingress 리소스 생성
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
EOF
kubectl get ingress nginx-ingress
```
- **확인**: ALB DNS로 브라우저 접속 → nginx 환영 페이지 확인.

#### 3.2.5 문제 해결
- **오류**: ALB DNS가 생성되지 않음
  - **해결**: `kubectl describe ingress nginx-ingress`로 상태 확인. ALB 컨트롤러 Pod 실행 확인: `kubectl get pods -n kube-system`.

---

## 4단계: 스토리지

### 4.1 개념 설명
- **CSI 드라이버**: EBS, EFS 등 클라우드 스토리지를 쿠버네티스와 연동.
- **PV/PVC**:
  - `PersistentVolume (PV)`: 실제 스토리지.
  - `PersistentVolumeClaim (PVC)`: Pod가 요청하는 스토리지.
- **접근 모드**: `ReadWriteOnce` (단일 노드), `ReadWriteMany` (다중 노드).

### 4.2 실습: EBS CSI 및 PVC
#### 4.2.1 EBS CSI 드라이버 설치
```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  -n kube-system
```

#### 4.2.2 PVC 및 Pod 생성
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: "gp2"
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: ebs-pvc
EOF
kubectl exec -it busybox-pod -- sh -c "echo hello > /data/test.txt && cat /data/test.txt"
```
- **확인**: `/data/test.txt`에 데이터 유지 확인.

#### 4.2.3 문제 해결
- **오류**: PVC가 `Pending` 상태
  - **해결**: `kubectl describe pvc ebs-pvc`로 확인. EBS CSI 드라이버가 설치되었는지 점검.

---

## 5단계: CI/CD 파이프라인

### 5.1 개념 설명
- **GitOps**: Git 상태를 클러스터에 동기화 (예: Argo CD).
- **CI/CD**: 빌드→테스트→배포 자동화 (예: GitHub Actions).
- **ECR**: AWS 컨테이너 레지스트리.

### 5.2 실습: GitHub Actions 및 Argo CD
#### 5.2.1 ECR 리포지터리 생성
```bash
aws ecr create-repository --repository-name my-app
```

#### 5.2.2 GitHub Actions 워크플로우
1. GitHub 리포지터리에 `.github/workflows/build-push.yml` 생성:
```yaml
name: Build and Push to ECR
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v1
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        AWS_REGION: ap-northeast-2
    - name: Build and Push Docker image
      run: |
        IMAGE_URI=${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-northeast-2.amazonaws.com/my-app
        docker build -t $IMAGE_URI:latest .
        docker push $IMAGE_URI:latest
```
2. GitHub 리포지터리 설정 → Secrets → `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ACCOUNT_ID` 추가.

#### 5.2.3 Argo CD 설치 및 애플리케이션 생성
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  project: default
  source:
    repoURL: https://github.com/<your-account>/my-app-manifests
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated: {}
EOF
```
- **사전 작업**: `my-app-manifests` 리포지터리에 `nginx` Deployment/Service YAML 업로드.

#### 5.2.4 문제 해결
- **오류**: GitHub Actions 빌드 실패
  - **해결**: Secrets 설정 확인. `docker build` 로그 점검.

---

## 6단계: 로깅 및 모니터링

### 6.1 개념 설명
- **Prometheus + Grafana**: 메트릭 수집 및 시각화.
- **EFK 스택**: Fluentd(로그 수집), Elasticsearch(저장), Kibana(분석).
- **CloudWatch Container Insights**: AWS 네이티브 모니터링.

### 6.2 실습: Prometheus 및 Grafana
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
helm install grafana grafana/grafana
# Grafana 접속
kubectl port-forward svc/grafana 3000:80
```
- **확인**: 브라우저에서 `http://localhost:3000` 접속 (초기 ID/PW: admin/admin).
- **대시보드 설정**: Grafana에서 "Kubernetes / Compute Resources / Cluster" 대시보드 추가.

#### 6.2.1 문제 해결
- **오류**: Grafana 접속 실패
  - **해결**: `kubectl get pods -n default`로 Grafana Pod 상태 확인.

---

## 7단계: 보안 강화

### 7.1 개념 설명
- **네임스페이스 격리**: 리소스 그룹별 분리.
- **RBAC**: Role과 RoleBinding으로 권한 관리.
- **이미지 스캔**: Trivy로 취약점 점검.

### 7.2 실습: RBAC 및 Trivy
#### 7.2.1 네임스페이스 및 RBAC
```bash
kubectl create namespace dev
cat <<EOF | kubectl apply -f -
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: dev
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "create", "delete"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: dev
  name: dev-binding
subjects:
- kind: User
  name: "developer@example.com"
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

#### 7.2.2 Trivy로 이미지 스캔
```bash
brew install aquasecurity/trivy/trivy
trivy image nginx:latest
```

#### 7.2.3 문제 해결
- **오류**: Trivy 스캔 실패
  - **해결**: `trivy` 버전 확인 (`trivy --version`). 최신 버전 설치.

---

## 8단계: 비용 최적화

### 8.1 개념 설명
- **Karpenter**: Pod 스펙 기반 인스턴스 프로비저닝.
- **HPA**: CPU 사용량 기반 Pod 자동 확장.
- **Spot 인스턴스**: 비용 절감.

### 8.2 실습: Karpenter 및 HPA
#### 8.2.1 Karpenter 설치
1. Karpenter IAM 역할 생성 (AWS 콘솔 또는 `awscli`):
   ```bash
   aws iam create-role --role-name KarpenterNodeInstanceProfile --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
   aws iam attach-role-policy --role-name KarpenterNodeInstanceProfile --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
   ```
2. Karpenter 설치:
   ```bash
   helm repo add karpenter https://charts.karpenter.sh
   helm repo update
   helm install karpenter karpenter/karpenter \
     --namespace karpenter --create-namespace \
     --set serviceAccount.create=true \
     --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile \
     --set clusterName=my-eks-cluster \
     --set region=ap-northeast-2
   ```

#### 8.2.2 HPA 설정
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-demo
  template:
    metadata:
      labels:
        app: cpu-demo
    spec:
      containers:
      - name: stress
        image: polinux/stress
        args: ["--cpu", "1", "--timeout", "600s"]
        resources:
          requests:
            cpu: "100m"
EOF
kubectl autoscale deployment cpu-demo --cpu-percent=50 --min=1 --max=5
kubectl get hpa -w
```

#### 8.2.3 문제 해결
- **오류**: Karpenter 프로비저닝 실패
  - **해결**: IAM 역할 권한 확인. `kubectl logs -n karpenter`로 로그 점검.

---

## 9단계: 리소스 정리
비용 절감을 위해 불필요한 리소스를 삭제하세요:
```bash
# 클러스터 삭제
eksctl delete cluster --name my-eks-cluster
# ECR 리포지터리 삭제
aws ecr delete-repository --repository-name my-app --force
# Helm 릴리스 삭제
helm uninstall prometheus
helm uninstall grafana
helm uninstall aws-ebs-csi-driver -n kube-system
helm uninstall karpenter -n karpenter
```

---

## 추가 팁
- **비용 추적**: AWS Cost Explorer로 EKS 관련 비용 모니터링.
- **보안 강화**: EKS 비밀 암호화 활성화 (KMS 키 사용).
- **리소스 확인 명령**:
  ```bash
  kubectl get all --all-namespaces
  aws eks list-clusters
  ```