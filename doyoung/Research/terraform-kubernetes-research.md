# 쿠버네티스 구축을 위한 테라폼 리서치

## 테라폼이란?

테라폼(Terraform)은 **Infrastructure as Code(IaC)** 도구로, 인프라를 코드로 정의하고 관리할 수 있게 해주는 도구입니다.

### 핵심 특징
- **선언적 언어**: 원하는 상태를 정의하면 테라폼이 알아서 구현
- **멱등성**: 여러 번 실행해도 같은 결과
- **상태 관리**: 현재 인프라 상태를 파일로 추적
- **멀티 클라우드**: AWS, Azure, GCP 등 다양한 클라우드 지원

## 테라폼으로 쿠버네티스 구축하는 이유

### 1. 일관성 보장
```hcl
# 동일한 설정으로 여러 환경 구축 가능
resource "aws_eks_cluster" "main" {
  name     = "my-kubernetes-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.28"
}
```

### 2. 버전 관리
- Git으로 인프라 코드 버전 관리
- 변경 이력 추적 가능
- 롤백 기능

### 3. 자동화
- CI/CD 파이프라인과 연동
- 반복 작업 자동화
- 실수 방지

## 테라폼 기본 구조

### 1. Provider 설정
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}
```

### 2. 리소스 정의
```hcl
# VPC 생성
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "kubernetes-vpc"
  }
}

# 서브넷 생성
resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  
  tags = {
    Name = "private-subnet"
  }
}
```

### 3. 출력값 정의
```hcl
output "cluster_endpoint" {
  value = aws_eks_cluster.main.endpoint
}

output "cluster_name" {
  value = aws_eks_cluster.main.name
}
```

## 쿠버네티스 구축 단계별 예시

### 1단계: 네트워크 인프라
```hcl
# VPC 생성
resource "aws_vpc" "eks_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "eks-vpc"
  }
}

# 퍼블릭 서브넷
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.eks_vpc.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# 프라이빗 서브넷
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.eks_vpc.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}
```

### 2단계: EKS 클러스터
```hcl
# EKS 클러스터
resource "aws_eks_cluster" "main" {
  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids = concat(aws_subnet.public[*].id, aws_subnet.private[*].id)
  }

  depends_on = [
    aws_iam_role_policy_attachment.eks_cluster_policy
  ]
}

# 노드 그룹
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main-node-group"
  node_role_arn   = aws_iam_role.eks_node_group.arn
  subnet_ids      = aws_subnet.private[*].id

  scaling_config {
    desired_size = 2
    max_size     = 4
    min_size     = 1
  }

  instance_types = ["t3.medium"]

  depends_on = [
    aws_iam_role_policy_attachment.eks_node_group_policy
  ]
}
```

### 3단계: IAM 역할
```hcl
# EKS 클러스터 역할
resource "aws_iam_role" "eks_cluster" {
  name = "eks-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "eks.amazonaws.com"
        }
      }
    ]
  })
}

# 노드 그룹 역할
resource "aws_iam_role" "eks_node_group" {
  name = "eks-node-group-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}
```

## 주요 테라폼 명령어

### 기본 명령어
```bash
# 초기화
terraform init

# 계획 확인
terraform plan

# 인프라 생성/변경
terraform apply

# 인프라 삭제
terraform destroy

# 상태 확인
terraform show
```

### 상태 관리
```bash
# 상태 파일 확인
terraform state list

# 특정 리소스 상태 확인
terraform state show aws_eks_cluster.main

# 상태 파일 백업
terraform state pull > terraform.tfstate.backup
```

## 모듈화 예시

### 메인 설정 파일
```hcl
# main.tf
module "vpc" {
  source = "./modules/vpc"
  
  vpc_cidr = "10.0.0.0/16"
  environment = "production"
}

module "eks" {
  source = "./modules/eks"
  
  cluster_name = "my-kubernetes-cluster"
  vpc_id = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

### 변수 정의
```hcl
# variables.tf
variable "cluster_name" {
  description = "EKS 클러스터 이름"
  type        = string
  default     = "my-kubernetes-cluster"
}

variable "kubernetes_version" {
  description = "쿠버네티스 버전"
  type        = string
  default     = "1.28"
}

variable "node_group_instance_types" {
  description = "노드 그룹 인스턴스 타입"
  type        = list(string)
  default     = ["t3.medium"]
}
```

## 모범 사례

### 1. 상태 관리
- **원격 상태 저장**: S3 + DynamoDB 사용
- **상태 파일 분리**: 환경별로 분리
- **상태 백업**: 정기적인 백업

### 2. 보안
```hcl
# 보안 그룹 예시
resource "aws_security_group" "eks_cluster" {
  name_prefix = "eks-cluster-"
  vpc_id      = aws_vpc.eks_vpc.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 3. 태깅 전략
```hcl
locals {
  common_tags = {
    Environment = "production"
    Project     = "kubernetes-cluster"
    ManagedBy   = "terraform"
  }
}

resource "aws_eks_cluster" "main" {
  # ... 기타 설정 ...
  
  tags = local.common_tags
}
```

## CI/CD 통합

### GitHub Actions 예시
```yaml
name: Terraform EKS

on:
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      
    - name: Terraform Init
      run: terraform init
      
    - name: Terraform Plan
      run: terraform plan
      
    - name: Terraform Apply
      run: terraform apply -auto-approve
```

## 장점과 단점

### 장점
- **재현 가능**: 동일한 인프라를 여러 번 생성 가능
- **버전 관리**: Git으로 인프라 코드 관리
- **자동화**: 수동 작업 최소화
- **일관성**: 환경별 차이 최소화

### 단점
- **학습 곡선**: 새로운 언어와 개념 학습 필요
- **상태 의존성**: 상태 파일 관리 중요
- **복잡성**: 복잡한 인프라의 경우 설정이 복잡

## 결론

테라폼을 사용하면 쿠버네티스 클러스터를 코드로 관리할 수 있어, 일관성 있고 재현 가능한 인프라를 구축할 수 있습니다. 적절한 모듈화와 상태 관리 전략을 사용하면 효율적인 쿠버네티스 환경을 구축할 수 있습니다.

## 참고 자료
- [Terraform 공식 문서](https://www.terraform.io/docs)
- [AWS EKS with Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eks_cluster)
- [Terraform Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html) 