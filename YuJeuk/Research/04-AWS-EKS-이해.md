# AWS EKS 이해

## ☁️ 3.1 EKS란 무엇인가?

### Elastic Kubernetes Service (EKS)
- AWS에서 제공하는 **관리형 쿠버네티스 서비스**
- 쿠버네티스 Control Plane(Master Node)을 AWS가 완전히 관리
- 사용자는 Worker Node와 애플리케이션에만 집중

### 관리형 서비스의 의미
```
직접 구축한 쿠버네티스:
┌─────────────────────────────────────┐
│ 사용자가 직접 관리해야 하는 영역      │
├─────────────────────────────────────┤
│ Master Node (Control Plane)         │
│ - API Server 설치/설정              │
│ - etcd 백업/복구                    │
│ - 보안 패치                         │
│ - 고가용성 구성                     │
├─────────────────────────────────────┤
│ Worker Node                         │
│ - kubelet, kube-proxy 설치          │
│ - 노드 관리                         │
└─────────────────────────────────────┘

EKS:
┌─────────────────────────────────────┐
│ AWS가 관리하는 영역                  │
├─────────────────────────────────────┤
│ Master Node (Control Plane)         │
│ - 자동 설치/설정                    │
│ - 자동 백업                         │
│ - 자동 보안 패치                    │
│ - 자동 고가용성 구성                │
├─────────────────────────────────────┤
│ 사용자가 관리하는 영역               │
├─────────────────────────────────────┤
│ Worker Node                         │
│ - EC2 인스턴스 관리                 │
│ - 애플리케이션 배포                 │
└─────────────────────────────────────┘
```

## 🎯 3.2 EKS의 주요 장점

### 1. 관리 부담 대폭 감소
**직접 구축 시 해야 할 일들:**
```bash
# Master Node 설치 및 설정
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# etcd 백업 스크립트 작성
#!/bin/bash
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 정기적인 보안 패치
sudo apt update && sudo apt upgrade -y
```

**EKS 사용 시:**
```bash
# 클러스터 생성 (Control Plane은 AWS가 자동 관리)
eksctl create cluster --name my-cluster --region ap-northeast-2

# 끝! 나머지는 AWS가 알아서 처리
```

### 2. 고가용성 자동 보장
```
직접 구축:
┌─────────────────────────────────────┐
│ 고가용성을 위해 직접 구성해야 함      │
├─────────────────────────────────────┤
│ Master Node 1 (AZ-a)                │
│ Master Node 2 (AZ-b)                │
│ Master Node 3 (AZ-c)                │
│ Load Balancer 설정                  │
│ etcd 클러스터 구성                  │
└─────────────────────────────────────┘

EKS:
┌─────────────────────────────────────┐
│ AWS가 자동으로 고가용성 구성         │
├─────────────────────────────────────┤
│ 여러 AZ에 Control Plane 분산 배치   │
│ 자동 장애 복구                      │
│ 99.95% SLA 보장                     │
└─────────────────────────────────────┘
```

### 3. AWS 서비스와의 완벽한 통합
```yaml
# AWS Load Balancer Controller 자동 통합
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

**통합되는 AWS 서비스들:**
- **IAM**: 세밀한 권한 관리
- **VPC**: 네트워크 보안
- **EBS**: 영구 스토리지
- **EFS**: 공유 파일 시스템
- **ECR**: 컨테이너 이미지 저장소
- **CloudWatch**: 모니터링 및 로깅
- **ALB/NLB**: 로드 밸런서

### 4. 자동 업그레이드와 보안 패치
```bash
# EKS 클러스터 버전 업그레이드
aws eks update-cluster-version --region ap-northeast-2 --name my-cluster --kubernetes-version 1.28

# AWS가 자동으로 처리하는 것들:
# - Control Plane 컴포넌트 업그레이드
# - 보안 패치 적용
# - 백워드 호환성 검증
# - 단계적 롤아웃
```

## 📊 3.3 EKS vs 직접 구축 상세 비교

| 구분 | EKS | 직접 구축 |
|------|-----|----------|
| **초기 설정 시간** | 15-30분 | 1-2일 |
| **Control Plane 관리** | AWS 자동 관리 | 직접 관리 필요 |
| **고가용성 구성** | 자동 제공 | 직접 구성 필요 |
| **보안 패치** | 자동 적용 | 수동 적용 |
| **백업/복구** | 자동 백업 | 직접 구성 |
| **모니터링** | CloudWatch 통합 | 별도 구성 |
| **비용** | Control Plane: $0.10/시간<br>+ Worker Node 비용 | Worker Node 비용만<br>+ 운영 인력 비용 |
| **업그레이드** | 원클릭 업그레이드 | 복잡한 수동 과정 |
| **전문 지식 요구도** | 중간 | 높음 |
| **장애 대응** | AWS 지원 | 직접 해결 |

## 💰 3.4 EKS 비용 구조

### Control Plane 비용
```
EKS 클러스터당: $0.10/시간 = $72/월
- 24시간 365일 실행
- 클러스터 개수에 따라 과금
- Control Plane 리소스는 무제한 사용
```

### Worker Node 비용
```bash
# 예시: t3.medium 인스턴스 3개 사용 시
# 인스턴스 비용: $0.0416/시간 × 3개 × 24시간 × 30일 = $89.86/월
# EBS 볼륨: 20GB × 3개 × $0.10/GB/월 = $6/월
# 총 Worker Node 비용: 약 $96/월

# 전체 EKS 비용 = Control Plane($72) + Worker Node($96) = $168/월
```

### 비용 최적화 팁
```bash
# Spot 인스턴스 사용으로 최대 90% 절약
eksctl create nodegroup --cluster=my-cluster --spot

# 클러스터 오토스케일러로 필요한 만큼만 사용
kubectl apply -f cluster-autoscaler.yaml

# 개발/테스트 환경은 사용하지 않을 때 중지
eksctl delete cluster --name dev-cluster
```

## 🔒 3.5 EKS 보안 특징

### IAM 통합
```yaml
# ServiceAccount와 IAM Role 연결
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/MyPodRole
```

### 네트워크 보안
```yaml
# VPC CNI를 통한 Pod 수준 보안 그룹
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    vpc.amazonaws.com/pod-eni: "true"
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
```

### 암호화
- **전송 중 암호화**: TLS 1.2 이상
- **저장 시 암호화**: etcd 데이터 KMS 암호화
- **Secret 암호화**: AWS Secrets Manager 통합

## 🎯 3단계 완료 체크리스트

### 이론 이해
- [ ] EKS가 관리형 서비스인 이유를 설명할 수 있다
- [ ] EKS와 직접 구축한 쿠버네티스의 차이점을 설명할 수 있다
- [ ] EKS의 주요 장점 4가지를 설명할 수 있다
- [ ] EKS 비용 구조를 이해했다

### AWS 콘솔 확인
```bash
# AWS CLI 설치 확인
aws --version

# AWS 계정 설정 확인
aws sts get-caller-identity

# EKS 서비스 확인 (AWS 콘솔)
# https://console.aws.amazon.com/eks/
```

### 실습 준비
- [ ] AWS 계정이 있고 적절한 권한이 설정되어 있다
- [ ] AWS CLI가 설치되고 설정되어 있다
- [ ] EKS 서비스 페이지에서 기본 개념을 확인했다

## 🤔 자주 묻는 질문

**Q: EKS를 사용하면 쿠버네티스를 몰라도 되나요?**
A: 아니요. EKS는 Control Plane 관리를 대신해줄 뿐, 쿠버네티스 자체의 개념과 사용법은 알아야 합니다.

**Q: 기존 쿠버네티스 클러스터를 EKS로 마이그레이션할 수 있나요?**
A: 네, 가능합니다. 애플리케이션과 데이터를 단계적으로 이전할 수 있어요.

**Q: EKS에서 다른 클라우드의 서비스를 사용할 수 있나요?**
A: 기술적으로는 가능하지만, AWS 서비스와의 통합 이점을 잃게 됩니다.

**Q: 개발 환경에서도 EKS를 사용해야 하나요?**
A: 개발 환경에서는 minikube나 kind 같은 로컬 도구를 사용하는 것이 비용 효율적입니다.

---

**다음 단계**: [05-EKS-실습-환경-구성.md](./05-EKS-실습-환경-구성.md)로 이동하여 실제 EKS 클러스터를 생성해보세요.