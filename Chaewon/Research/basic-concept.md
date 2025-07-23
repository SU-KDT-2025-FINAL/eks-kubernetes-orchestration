# Kubernetes & EKS 기본 개념 정리

## 1. Kubernetes 개요

### Kubernetes란?
- **컨테이너 오케스트레이션 플랫폼**으로, 컨테이너화된 애플리케이션의 배포, 확장, 관리를 자동화
- Google에서 개발하여 오픈소스로 공개
- 마이크로서비스 아키텍처의 핵심 기술

### 주요 특징
- **자동 배포 및 롤백**: 애플리케이션 버전 관리 및 무중단 배포
- **서비스 디스커버리**: 서비스 간 자동 연결 및 로드 밸런싱
- **스토리지 오케스트레이션**: 다양한 스토리지 시스템 통합
- **자동 확장**: CPU, 메모리 사용량에 따른 자동 스케일링
- **자가 치유**: 장애 발생 시 자동 복구

## 2. Kubernetes 핵심 구성 요소

### 클러스터 아키텍처
```
┌─────────────────┐    ┌─────────────────┐
│   Master Node   │    │   Worker Node   │
│                 │    │                 │
│  - API Server   │◄──►│   - Kubelet     │
│  - etcd         │    │   - Kube-proxy  │
│  - Scheduler    │    │   - Container   │
│  - Controller   │    │     Runtime     │
└─────────────────┘    └─────────────────┘
```

### Master Node 구성 요소
- **API Server**: 클러스터의 모든 통신 중심점
- **etcd**: 클러스터 상태 정보를 저장하는 분산 키-값 저장소
- **Scheduler**: Pod를 적절한 노드에 할당
- **Controller Manager**: 클러스터 상태를 원하는 상태로 유지

### Worker Node 구성 요소
- **Kubelet**: 각 노드에서 Pod를 관리하는 에이전트
- **Kube-proxy**: 네트워크 프록시 및 로드 밸런서
- **Container Runtime**: 컨테이너 실행 환경 (Docker, containerd 등)

## 3. Kubernetes 주요 리소스

### Pod
- Kubernetes의 **최소 배포 단위**
- 하나 이상의 컨테이너를 포함
- 같은 Pod 내 컨테이너들은 네트워크와 스토리지 공유

### Deployment
- Pod의 **선언적 배포**를 관리
- 레플리카 수, 롤링 업데이트 전략 정의
- 자동 롤백 기능 제공

### Service
- Pod 그룹에 대한 **네트워크 접근점** 제공
- 로드 밸런싱 및 서비스 디스커버리
- 타입: ClusterIP, NodePort, LoadBalancer, ExternalName

### ConfigMap & Secret
- **ConfigMap**: 설정 데이터 저장
- **Secret**: 민감한 정보 (비밀번호, 토큰 등) 저장

### Namespace
- 클러스터 내 **논리적 분리** 단위
- 리소스 격리 및 권한 관리

## 4. Amazon EKS (Elastic Kubernetes Service)

### EKS란?
- AWS에서 제공하는 **관리형 Kubernetes 서비스**
- Kubernetes 마스터 노드를 AWS가 완전 관리
- AWS 서비스와의 원활한 통합

### EKS 주요 특징
- **완전 관리형**: 마스터 노드 운영, 패치, 업그레이드 자동화
- **고가용성**: 다중 AZ에 마스터 노드 배포
- **보안**: AWS IAM과 통합된 인증/인가
- **확장성**: Auto Scaling 그룹과 연동

### EKS 구성 요소
- **EKS 클러스터**: 관리형 Kubernetes 컨트롤 플레인
- **Node Group**: EC2 인스턴스 그룹 (워커 노드)
- **Fargate**: 서버리스 컨테이너 실행 환경

## 5. EKS 네트워킹

### VPC 통합
- EKS 클러스터는 VPC 내에 생성
- 서브넷을 통한 노드 배치
- 보안 그룹으로 네트워크 트래픽 제어

### CNI (Container Network Interface)
- **AWS VPC CNI**: Pod에 VPC IP 주소 직접 할당
- 네이티브 AWS 네트워킹 지원
- 보안 그룹을 Pod 레벨에서 적용 가능

### 로드 밸런싱
- **Application Load Balancer (ALB)**: HTTP/HTTPS 트래픽
- **Network Load Balancer (NLB)**: TCP/UDP 트래픽
- **AWS Load Balancer Controller**: 자동 로드 밸런서 프로비저닝

## 6. EKS 보안

### IAM 통합
- **RBAC**: Kubernetes 역할 기반 접근 제어
- **IAM 역할**: AWS 서비스 접근 권한
- **IRSA (IAM Roles for Service Accounts)**: Pod 레벨 IAM 권한

### 보안 모범 사례
- 최소 권한 원칙 적용
- 네트워크 정책을 통한 트래픽 제어
- 시크릿 관리 (AWS Secrets Manager 연동)
- 이미지 취약점 스캔

## 7. EKS 모니터링 및 로깅

### CloudWatch 통합
- **Container Insights**: 클러스터 및 애플리케이션 메트릭
- **로그 수집**: CloudWatch Logs로 컨테이너 로그 전송

### 추가 모니터링 도구
- **Prometheus**: 메트릭 수집 및 모니터링
- **Grafana**: 대시보드 및 시각화
- **Jaeger**: 분산 추적

## 8. EKS 배포 도구

### kubectl
- Kubernetes 클러스터 관리를 위한 CLI 도구
- 리소스 생성, 수정, 삭제 및 상태 확인

### Helm
- Kubernetes **패키지 매니저**
- 복잡한 애플리케이션 배포 및 관리 단순화
- 차트를 통한 템플릿 기반 배포

### AWS CLI & eksctl
- **eksctl**: EKS 클러스터 생성 및 관리 전용 CLI
- AWS CLI를 통한 EKS 리소스 관리

## 9. EKS 비용 최적화

### 인스턴스 타입 선택
- **온디맨드 인스턴스**: 예측 가능한 워크로드
- **스팟 인스턴스**: 비용 절약 (최대 90% 할인)
- **예약 인스턴스**: 장기 사용 시 할인

### 오토스케일링
- **Cluster Autoscaler**: 노드 자동 확장/축소
- **Horizontal Pod Autoscaler**: Pod 수평 확장
- **Vertical Pod Autoscaler**: Pod 리소스 자동 조정

### Fargate vs EC2
- **Fargate**: 서버리스, 사용한 만큼 과금
- **EC2**: 전통적인 방식, 더 많은 제어권

## 10. 프로젝트 시작 전 체크리스트

### 사전 준비
- [ ] AWS 계정 및 IAM 권한 설정
- [ ] kubectl, eksctl, AWS CLI 설치
- [ ] Docker 이해 및 컨테이너 이미지 준비
- [ ] VPC 및 네트워크 아키텍처 설계

### 아키텍처 고려사항
- [ ] 클러스터 규모 및 노드 구성 계획
- [ ] 네트워킹 및 보안 정책 수립
- [ ] 모니터링 및 로깅 전략
- [ ] CI/CD 파이프라인 설계
- [ ] 백업 및 재해 복구 계획

### 학습 우선순위
1. Kubernetes 기본 개념 및 리소스
2. EKS 클러스터 생성 및 관리
3. 애플리케이션 배포 및 서비스 노출
4. 모니터링 및 트러블슈팅
5. 보안 및 네트워킹 고급 설정