# EKS-Kubernetes Orchestration 스터디 개요

## 1. 주제 개요

- **EKS(Elastic Kubernetes Service)**: AWS에서 제공하는 완전관리형 Kubernetes 서비스로, 인프라 관리 부담 없이 컨테이너 오케스트레이션이 가능합니다.
- **Kubernetes Orchestration**: 컨테이너화된 애플리케이션의 배포, 확장, 관리를 자동화하는 기술로, Pod, Service, Deployment, Ingress, ConfigMap, Secret 등 다양한 리소스를 활용합니다.
- **DevOps와의 연계**: CI/CD, IaC(Terraform, CloudFormation), 모니터링, 롤링 업데이트, 블루/그린 배포 등 자동화된 운영 환경 구축에 필수적입니다.

## 2. 주요 학습 목표

- EKS 클러스터 생성 및 관리
- Kubernetes 핵심 리소스와 오브젝트 구조 이해
- 실전형 CI/CD 파이프라인 구축 (GitHub Actions, ArgoCD 등)
- IaC를 통한 인프라 자동화
- 실시간 모니터링 및 로깅 (Prometheus, Grafana, CloudWatch)
- 실전 배포 전략(롤링, 카나리, 블루/그린)
- 보안(네트워크 정책, IAM, Secret 관리)
- 3차 프로젝트에 필요한 실전 예제 및 실습

## 3. 스터디 진행 방식 예시

### 예시 1: 실습형 워크플로우
- eksctl로 EKS 클러스터 생성
- kubectl로 Nginx Deployment 및 Service 생성
- LoadBalancer 타입으로 외부 노출
- 배포 상태 확인 및 트러블슈팅

### 예시 2: 아키텍처 다이어그램 & 설명
- VPC, Subnet, EKS Control Plane, Node Group, ALB, External DNS, CI/CD 파이프라인, 모니터링 스택(Prometheus, Grafana)
- 각 컴포넌트의 역할 및 상호작용 설명

### 예시 3: 실전 시나리오 Q&A
- Pod가 CrashLoopBackOff 상태일 때 원인 진단 및 해결 방법
- 롤링 업데이트 중 장애 발생 시 롤백 방법
- EKS에서 Secret을 안전하게 관리하는 방법

## 4. 3차 프로젝트에 필요한 심화 내용

- Helm Chart를 활용한 배포 자동화
- ArgoCD를 통한 GitOps 파이프라인 구축
- Terraform으로 EKS 및 관련 리소스 IaC 관리
- Service Mesh (Istio, Linkerd) 도입 및 트래픽 관리
- Kubernetes RBAC 및 네트워크 정책
- EKS에서의 비용 최적화 전략
- 실시간 모니터링/알림 연동 (Slack, PagerDuty 등)
- Multi-Cluster 관리 및 DR(Disaster Recovery) 전략

## 5. 실습/토론 프롬프트 예시

- eksctl을 사용해 EKS 클러스터를 생성하는 명령어와 주요 옵션을 설명하세요.
- Kubernetes에서 Deployment와 StatefulSet의 차이점은 무엇인가요? 각각의 사용 사례를 들어 설명해보세요.
- ArgoCD를 활용한 GitOps 배포 파이프라인을 설계하고, 실제로 적용하는 과정을 단계별로 정리해보세요.
- Helm Chart를 사용해 Nginx Ingress Controller를 배포하는 실습을 진행해보세요.
- EKS에서 Pod 간 네트워크 통신을 제한하는 네트워크 정책(NetworkPolicy) 예시를 작성해보세요.
- Terraform을 이용해 EKS 클러스터와 Node Group을 코드로 관리하는 예시 코드를 작성해보세요.
- 실시간 모니터링을 위해 Prometheus와 Grafana를 EKS에 설치하고, 대시보드를 구성하는 방법을 설명하세요.
- EKS에서 발생할 수 있는 비용을 최적화하는 방안에는 어떤 것들이 있는지 토론해보세요.
- Service Mesh를 도입했을 때 얻을 수 있는 이점과, Istio의 주요 기능을 설명해보세요.
- EKS 환경에서 장애 복구(Disaster Recovery) 전략을 어떻게 수립할 수 있을지 논의해보세요. 