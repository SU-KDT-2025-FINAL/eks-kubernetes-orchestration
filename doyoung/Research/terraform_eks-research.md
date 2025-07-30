# Terraform을 이용한 AWS EKS 클러스터 구축 및 관리 리서치

## 목차
1. [Terraform 기반 EKS 인프라 설계 원칙](#terraform-기반-eks-인프라-설계-원칙)
2. [주요 리소스 및 모듈 구조](#주요-리소스-및-모듈-구조)
3. [실전 예시: 기본 EKS 클러스터 코드](#실전-예시-기본-eks-클러스터-코드)
4. [운영 자동화 및 관리 팁](#운영-자동화-및-관리-팁)
5. [보안, 비용, 확장성 고려사항](#보안-비용-확장성-고려사항)
6. [실무 적용 모범 사례](#실무-적용-모범-사례)

---

## 1. Terraform 기반 EKS 인프라 설계 원칙
- **코드로 인프라 관리(IaC)**: 모든 리소스 선언적 관리, 변경 이력 추적
- **모듈화**: EKS, VPC, 노드그룹 등 역할별 모듈 분리
- **버전 관리**: provider, module, 리소스 버전 명시
- **변수화**: 환경별 파라미터 분리, 재사용성 강화
- **상태 관리**: 원격 backend(S3+Locking)로 상태파일 관리, 충돌 방지

## 2. 주요 리소스 및 모듈 구조
- **VPC**: 퍼블릭/프라이빗 서브넷, NAT, IGW 등 네트워크 구성
- **EKS 클러스터**: 관리형 제어플레인, IAM 역할, 클러스터 정책
- **노드 그룹**: managed node group, launch template, spot/ondemand 혼합
- **IAM**: IRSA, 노드/컨트롤플레인 권한 분리
- **보안**: SG, NACL, KMS, Secret Manager 연동
- **Add-ons**: CoreDNS, kube-proxy, VPC CNI, ALB Ingress Controller 등

### 예시 모듈 구조
```
modules/
  vpc/
  eks/
  nodegroup/
  iam/
  addons/
main.tf
variables.tf
outputs.tf
```

## 3. 실전 예시: 기본 EKS 클러스터 코드
```hcl
provider "aws" {
  region = var.region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  name    = "eks-vpc"
  cidr    = "10.0.0.0/16"
  azs     = ["ap-northeast-2a", "ap-northeast-2c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.101.0/24", "10.0.102.0/24"]
  enable_nat_gateway = true
}

module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "my-eks-cluster"
  cluster_version = "1.29"
  subnets         = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id
  enable_irsa     = true
  node_groups = {
    default = {
      desired_capacity = 2
      max_capacity     = 4
      min_capacity     = 1
      instance_types   = ["t3.medium"]
    }
  }
}
```

## 4. 운영 자동화 및 관리 팁
- **GitOps 연계**: Terraform 코드 PR 기반 배포, CI/CD 파이프라인 연동
- **Terraform Cloud/Enterprise**: 정책, 승인, 감사, 원격 상태 관리
- **모듈 버전 고정**: module source에 버전 명시, 예) `?ref=v19.21.0`
- **변수 파일 분리**: 환경별(개발/운영) tfvars 파일 관리
- **Output 활용**: kubeconfig, 노드그룹 정보 등 자동 출력

## 5. 보안, 비용, 확장성 고려사항
- **IRSA**: 서비스 계정별 IAM 권한 최소화
- **프라이빗 클러스터**: API endpoint 제한, 보안그룹 최소화
- **스팟 인스턴스**: 비용 절감, mixed node group 활용
- **Auto Scaling**: Cluster Autoscaler, Karpenter 연동
- **리소스 태깅**: 비용 분석, 운영 자동화에 활용
- **로그/모니터링**: CloudWatch, Prometheus, Grafana 연동

## 6. 실무 적용 모범 사례
- **모듈화/재사용**: 공통 모듈로 표준화, 신규 프로젝트 신속 적용
- **상태파일 백업**: S3 버전 관리, 주기적 백업
- **변경 관리**: terraform plan/apply 분리, 승인 프로세스 적용
- **보안 정책 자동화**: Security Group, IAM Policy 코드화
- **운영 자동화**: 배포, 롤백, 모니터링까지 자동화

---

Terraform을 활용하면 AWS EKS 클러스터의 생성, 확장, 운영, 보안까지 코드로 일관성 있게 관리할 수 있습니다. 실무에서는 모듈화, 변수화, 자동화, 보안/비용 최적화, GitOps 등 다양한 Best Practice를 적극적으로 적용하는 것이 중요합니다. 