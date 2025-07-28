# AWS EKS (Elastic Kubernetes Service) 리서치

## EKS란?

AWS EKS는 Amazon에서 제공하는 **관리형 Kubernetes 서비스**입니다. Kubernetes 클러스터의 제어플레인(마스터 노드)을 AWS가 관리해주어, 개발자가 워커 노드와 애플리케이션에만 집중할 수 있게 해줍니다.

## 주요 특징

### 1. 완전 관리형 제어플레인
- AWS가 마스터 노드(API 서버, etcd, 스케줄러 등)를 관리
- 고가용성 보장 (99.95% SLA)
- 자동 보안 패치 및 업데이트

### 2. AWS 서비스와의 완벽 통합
- IAM과 Kubernetes RBAC 연동
- VPC 네트워킹 활용
- CloudWatch, ALB, EBS 등 AWS 서비스와 통합

### 3. 다중 AZ 지원
- 여러 가용영역에 걸친 고가용성
- 자동 장애 복구
- 지역별 배포 지원

## 왜 EKS를 사용할까?

### 장점
- **운영 부담 감소**: 마스터 노드 관리 불필요
- **보안 강화**: AWS 보안 기능 활용
- **확장성**: Auto Scaling으로 자동 확장
- **비용 효율성**: 사용한 만큼만 비용 지불
- **생태계**: 풍부한 AWS 서비스 연동

### 단점
- **비용**: 자체 구축보다 비용이 높을 수 있음
- **벤더 종속**: AWS에 종속될 수 있음
- **복잡성**: 초기 학습 곡선

## EKS 구성 요소

### 1. 제어플레인 (AWS 관리)
```
┌─────────────────┐
│   EKS Control   │
│     Plane       │
│                 │
│  ┌───────────┐  │
│  │ API Server│  │
│  └───────────┘  │
│  ┌───────────┐  │
│  │   etcd    │  │
│  └───────────┘  │
│  ┌───────────┐  │
│  │Scheduler  │  │
│  └───────────┘  │
└─────────────────┘
```

### 2. 워커 노드 (사용자 관리)
- **EC2 인스턴스**: 애플리케이션 Pod 실행
- **노드 그룹**: 관리형 또는 자체 관리
- **Auto Scaling**: 수요에 따른 자동 확장

### 3. 네트워킹
- **VPC**: 격리된 네트워크 환경
- **서브넷**: 퍼블릭/프라이빗 분리
- **보안 그룹**: 트래픽 제어

## 실무 활용 사례

### 1. 마이크로서비스 아키텍처
```yaml
# 서비스 분해 예시
Services:
  - User Service (3 replicas)
  - Order Service (5 replicas)  
  - Payment Service (2 replicas)
  - Inventory Service (3 replicas)
```

### 2. CI/CD 파이프라인
- Jenkins, GitLab CI, GitHub Actions 연동
- 자동 배포 및 롤백
- Blue/Green, Canary 배포

### 3. 하이브리드 클라우드
- 온프레미스와 클라우드 환경 통합
- EKS Anywhere로 온프레미스 확장

## 비용 구조

### 주요 비용 요소
1. **EKS 제어플레인**: 시간당 요금
2. **EC2 인스턴스**: 워커 노드 비용
3. **네트워킹**: 데이터 전송, NAT Gateway
4. **스토리지**: EBS, EFS
5. **로드 밸런서**: ALB, NLB

### 비용 최적화 팁
- 스팟 인스턴스 활용 (최대 90% 절약)
- Auto Scaling으로 과도한 리소스 방지
- 예약 인스턴스 활용
- 리소스 태깅으로 비용 추적

## 보안 고려사항

### 1. 네트워크 보안
- VPC 격리
- 보안 그룹 설정
- 프라이빗 서브넷 활용

### 2. 접근 제어
- IAM Roles for Service Accounts (IRSA)
- Kubernetes RBAC
- 네트워크 정책

### 3. 데이터 보안
- EBS 암호화
- 트랜짓 암호화 (TLS)
- 시크릿 관리

## 모니터링 및 로깅

### 1. CloudWatch 통합
- 메트릭 수집
- 로그 집계
- 알림 설정

### 2. Prometheus + Grafana
- 커스텀 메트릭
- 대시보드 구성
- 알림 규칙

### 3. 로그 관리
- Fluent Bit/Fluentd
- Elasticsearch + Kibana
- 로그 보존 정책

## 시작하기

### 1. 기본 요구사항
- AWS 계정
- kubectl 설치
- AWS CLI 설정

### 2. 클러스터 생성
```bash
# eksctl 사용 (가장 간단)
eksctl create cluster --name my-cluster --region ap-northeast-2

# 또는 Terraform 사용
terraform init
terraform plan
terraform apply
```

### 3. 애플리케이션 배포
```yaml[.gitignore](../../.idea/.gitignore)
# 간단한 애플리케이션 배포
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:latest
        ports:
        - containerPort: 80
```

## 결론

AWS EKS는 Kubernetes의 복잡성을 줄이면서도 엔터프라이즈급 기능을 제공하는 강력한 플랫폼입니다. 특히 AWS 생태계와의 완벽한 통합, 관리형 서비스의 편의성, 고가용성 등이 큰 장점입니다.

실무에서는 적절한 아키텍처 설계, 보안 구성, 비용 최적화, 모니터링 설정을 통해 안정적이고 확장 가능한 컨테이너 환경을 구축할 수 있습니다. 