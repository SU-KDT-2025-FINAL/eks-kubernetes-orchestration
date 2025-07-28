# EKS 실습 환경 구성

## 🛠️ 4.1 필요한 도구 설치

### AWS CLI 설치 및 설정
```bash
# macOS에서 AWS CLI 설치
brew install awscli

# 설치 확인
aws --version
# aws-cli/2.x.x Python/3.x.x Darwin/xx.x.x source/x86_64

# AWS 계정 설정
aws configure
# AWS Access Key ID [None]: YOUR_ACCESS_KEY
# AWS Secret Access Key [None]: YOUR_SECRET_KEY
# Default region name [None]: ap-northeast-2
# Default output format [None]: json

# 설정 확인
aws sts get-caller-identity
```

### eksctl 설치 (EKS 클러스터 생성 도구)
```bash
# macOS에서 eksctl 설치
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# 설치 확인
eksctl version
# 0.x.x
```

### kubectl 설치
```bash
# macOS에서 kubectl 설치
brew install kubectl

# 설치 확인
kubectl version --client
# Client Version: v1.28.x
```

### 추가 유용한 도구들
```bash
# k9s - 쿠버네티스 TUI 도구
brew install k9s

# helm - 쿠버네티스 패키지 매니저
brew install helm

# aws-iam-authenticator (보통 자동 설치됨)
brew install aws-iam-authenticator
```

## 🚀 4.2 첫 번째 EKS 클러스터 생성

### 간단한 클러스터 생성
```bash
# 기본 설정으로 클러스터 생성 (약 15-20분 소요)
eksctl create cluster \
  --name my-first-cluster \
  --region ap-northeast-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# 생성 과정에서 출력되는 로그 예시:
# [ℹ]  eksctl version 0.x.x
# [ℹ]  using region ap-northeast-2
# [ℹ]  setting availability zones to [ap-northeast-2a ap-northeast-2c]
# [ℹ]  subnets for ap-northeast-2a - public:192.168.0.0/19 private:192.168.96.0/19
# [ℹ]  subnets for ap-northeast-2c - public:192.168.32.0/19 private:192.168.128.0/19
# [ℹ]  nodegroup "standard-workers" will use "ami-0c02fb55956c7d316" [AmazonLinux2/1.28]
# [ℹ]  using Kubernetes version 1.28
# [ℹ]  creating EKS cluster "my-first-cluster" in "ap-northeast-2" region with managed nodes
```

### 클러스터 생성 옵션 설명
```bash
--name my-first-cluster     # 클러스터 이름
--region ap-northeast-2     # AWS 리전 (서울)
--nodegroup-name standard-workers  # 워커 노드 그룹 이름
--node-type t3.medium      # EC2 인스턴스 타입
--nodes 2                  # 초기 노드 수
--nodes-min 1              # 최소 노드 수
--nodes-max 4              # 최대 노드 수
--managed                  # 관리형 노드 그룹 사용
```

### YAML 파일을 사용한 고급 설정
```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-advanced-cluster
  region: ap-northeast-2

nodeGroups:
  - name: standard-workers
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 4
    volumeSize: 20
    ssh:
      allow: true # SSH 접근 허용
    iam:
      withAddonPolicies:
        imageBuilder: true
        autoScaler: true
        ebs: true
        efs: true
        albIngress: true
        cloudWatch: true

# 클러스터 생성
eksctl create cluster -f cluster-config.yaml
```

## 🔍 4.3 클러스터 확인 및 기본 명령어

### 클러스터 정보 확인
```bash
# 클러스터 기본 정보
kubectl cluster-info
# Kubernetes control plane is running at https://xxx.gr7.ap-northeast-2.eks.amazonaws.com
# CoreDNS is running at https://xxx.gr7.ap-northeast-2.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# 클러스터 상세 정보
kubectl cluster-info dump

# 현재 컨텍스트 확인
kubectl config current-context
# arn:aws:eks:ap-northeast-2:123456789012:cluster/my-first-cluster
```

### 노드 확인
```bash
# 노드 목록 확인
kubectl get nodes
# NAME                                               STATUS   ROLES    AGE   VERSION
# ip-192-168-1-234.ap-northeast-2.compute.internal   Ready    <none>   5m    v1.28.x
# ip-192-168-2-123.ap-northeast-2.compute.internal   Ready    <none>   5m    v1.28.x

# 노드 상세 정보
kubectl describe nodes

# 노드 리소스 사용량 확인
kubectl top nodes
```

### 시스템 Pod 확인
```bash
# kube-system 네임스페이스의 Pod들 확인
kubectl get pods -n kube-system
# NAME                       READY   STATUS    RESTARTS   AGE
# aws-node-xxxxx             1/1     Running   0          10m
# aws-node-yyyyy             1/1     Running   0          10m
# coredns-xxxxx              1/1     Running   0          15m
# coredns-yyyyy              1/1     Running   0          15m
# kube-proxy-xxxxx           1/1     Running   0          10m
# kube-proxy-yyyyy           1/1     Running   0          10m

# 모든 네임스페이스의 리소스 확인
kubectl get all --all-namespaces
```

## 🧪 4.4 첫 번째 애플리케이션 배포 테스트

### 간단한 nginx 배포
```bash
# nginx 배포
kubectl create deployment nginx-test --image=nginx:1.20

# 배포 상태 확인
kubectl get deployments
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-test   1/1     1            1           30s

# Pod 확인
kubectl get pods
# NAME                          READY   STATUS    RESTARTS   AGE
# nginx-test-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
```

### 서비스 생성 및 외부 접근
```bash
# LoadBalancer 타입 서비스 생성
kubectl expose deployment nginx-test --port=80 --type=LoadBalancer

# 서비스 확인 (External IP가 생성될 때까지 대기)
kubectl get services
# NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                                                   PORT(S)        AGE
# kubernetes   ClusterIP      10.100.0.1      <none>                                                                        443/TCP        20m
# nginx-test   LoadBalancer   10.100.123.45   a1b2c3d4e5f6g7h8-123456789.ap-northeast-2.elb.amazonaws.com                80:31234/TCP   2m

# 외부 IP로 접근 테스트 (External IP가 준비되면)
curl http://a1b2c3d4e5f6g7h8-123456789.ap-northeast-2.elb.amazonaws.com
```

### 리소스 정리
```bash
# 테스트 리소스 삭제
kubectl delete service nginx-test
kubectl delete deployment nginx-test

# 삭제 확인
kubectl get all
```

## 📊 4.5 AWS 콘솔에서 확인하기

### EKS 콘솔 확인 사항
1. **AWS 콘솔 → EKS → 클러스터**
   - 클러스터 상태: Active
   - 쿠버네티스 버전 확인
   - 엔드포인트 URL 확인

2. **노드 그룹 탭**
   - 노드 그룹 상태: Active
   - 인스턴스 타입과 개수 확인
   - Auto Scaling 설정 확인

3. **네트워킹 탭**
   - VPC 및 서브넷 정보
   - 보안 그룹 설정
   - 클러스터 엔드포인트 접근 설정

### EC2 콘솔에서 워커 노드 확인
```
AWS 콘솔 → EC2 → 인스턴스
- EKS 워커 노드들이 실행 중인지 확인
- 인스턴스 이름에 클러스터 이름이 포함됨
- 보안 그룹과 키 페어 설정 확인
```

## 🔧 4.6 유용한 관리 명령어

### eksctl 클러스터 관리
```bash
# 클러스터 목록 확인
eksctl get cluster

# 클러스터 상세 정보
eksctl get cluster --name my-first-cluster

# 노드 그룹 목록 확인
eksctl get nodegroup --cluster my-first-cluster

# 노드 그룹 스케일링
eksctl scale nodegroup --cluster my-first-cluster --nodes 3 --name standard-workers
```

### kubectl 컨텍스트 관리
```bash
# 사용 가능한 컨텍스트 목록
kubectl config get-contexts

# 컨텍스트 전환 (여러 클러스터 사용 시)
kubectl config use-context arn:aws:eks:ap-northeast-2:123456789012:cluster/my-first-cluster

# kubeconfig 업데이트 (필요 시)
aws eks update-kubeconfig --region ap-northeast-2 --name my-first-cluster
```

## 🎯 4단계 완료 체크리스트

### 도구 설치 완료
- [ ] AWS CLI 설치 및 설정 완료
- [ ] eksctl 설치 완료
- [ ] kubectl 설치 완료
- [ ] 모든 도구의 버전 확인 완료

### 클러스터 생성 완료
- [ ] EKS 클러스터 생성 성공
- [ ] 워커 노드 2개 정상 실행
- [ ] kubectl로 클러스터 접근 가능
- [ ] AWS 콘솔에서 클러스터 상태 확인

### 기본 동작 확인
- [ ] 간단한 애플리케이션 배포 성공
- [ ] LoadBalancer 서비스 생성 및 외부 접근 확인
- [ ] 리소스 정리 완료

### 비용 관리
- [ ] 현재 실행 중인 리소스 파악
- [ ] 불필요한 리소스 정리 방법 숙지
- [ ] AWS 비용 알림 설정 (권장)

## ⚠️ 중요한 주의사항

### 비용 관리
```bash
# 실습 완료 후 클러스터 삭제 (비용 절약)
eksctl delete cluster --name my-first-cluster

# 삭제 확인
eksctl get cluster
```

### 보안 설정
```bash
# 클러스터 엔드포인트 접근 제한 (프로덕션 환경)
eksctl utils update-cluster-endpoints --cluster my-first-cluster --private-access=true --public-access=false
```

### 문제 해결
```bash
# 클러스터 생성 실패 시 로그 확인
eksctl utils describe-stacks --region ap-northeast-2 --cluster my-first-cluster

# kubeconfig 문제 시 재설정
aws eks update-kubeconfig --region ap-northeast-2 --name my-first-cluster --force
```

## 🤔 자주 발생하는 문제들

**Q: 클러스터 생성이 실패합니다.**
A: AWS 권한, 리전 설정, VPC 한도 등을 확인해보세요. CloudFormation 콘솔에서 상세 오류를 확인할 수 있습니다.

**Q: kubectl 명령어가 작동하지 않습니다.**
A: `aws eks update-kubeconfig` 명령어로 kubeconfig를 다시 설정해보세요.

**Q: LoadBalancer 서비스의 External IP가 생성되지 않습니다.**
A: AWS Load Balancer Controller가 설치되어 있는지 확인하고, 몇 분 정도 기다려보세요.

---

**다음 단계**: [06-기본-애플리케이션-배포.md](./06-기본-애플리케이션-배포.md)로 이동하여 실제 애플리케이션을 배포해보세요.