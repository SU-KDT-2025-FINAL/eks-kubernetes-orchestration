# Helm 가이드

## Helm이란?

Helm은 쿠버네티스를 위한 패키지 매니저입니다. Linux의 `apt`나 `yum`, macOS의 `brew`와 유사하게 쿠버네티스 애플리케이션을 쉽게 설치, 업그레이드, 관리할 수 있도록 도와주는 도구입니다.

Helm을 사용하면 복잡한 쿠버네티스 애플리케이션을 템플릿화하여 재사용 가능한 패키지로 만들 수 있으며, 이를 통해 애플리케이션 배포와 관리를 자동화할 수 있습니다.

## Helm의 3가지 주요 개념

### 1. Chart (차트)
- Helm의 패키지 형태
- 쿠버네티스 리소스를 생성하는 데 필요한 모든 정보가 포함된 파일들의 집합
- 템플릿, 기본값, 메타데이터 등을 포함
- `nginx`, `mysql`, `prometheus` 등의 차트를 Helm Hub나 공식 저장소에서 찾을 수 있음

```
mychart/
├── Chart.yaml          # 차트 메타데이터
├── values.yaml         # 기본 설정값
├── charts/             # 의존성 차트
└── templates/          # 쿠버네티스 매니페스트 템플릿
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

### 2. Release (릴리즈)
- 쿠버네티스 클러스터에 설치된 차트의 인스턴스
- 동일한 차트를 여러 번 설치하면 각각 다른 릴리즈가 생성됨
- 각 릴리즈는 고유한 이름을 가지며 독립적으로 관리됨
- 예: `my-nginx-release`, `production-mysql`, `staging-app`

### 3. Repository (저장소)
- 차트가 저장되고 공유되는 장소
- 공식 Helm Hub, Artifact Hub, 또는 사설 저장소 사용 가능
- `helm repo add` 명령으로 저장소를 추가하고 차트를 검색/설치할 수 있음

## 간단한 사용 예제

### 1. Helm 설치 및 초기 설정
```bash
# Helm 설치 (macOS)
brew install helm

# Helm 설치 (Windows - Chocolatey)
choco install kubernetes-helm

# 공식 저장소 추가
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 2. 차트 검색 및 설치
```bash
# nginx 차트 검색
helm search repo nginx

# nginx 설치
helm install my-nginx bitnami/nginx

# 설치된 릴리즈 확인
helm list
```

### 3. 커스텀 값으로 설치
```bash
# values.yaml 파일 생성
cat > my-values.yaml << EOF
replicaCount: 3
service:
  type: LoadBalancer
  port: 80
EOF

# 커스텀 값으로 설치
helm install my-custom-nginx bitnami/nginx -f my-values.yaml
```

### 4. 릴리즈 관리
```bash
# 릴리즈 업그레이드
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# 릴리즈 상태 확인
helm status my-nginx

# 릴리즈 히스토리 확인
helm history my-nginx

# 이전 버전으로 롤백
helm rollback my-nginx 1

# 릴리즈 삭제
helm uninstall my-nginx
```

### 5. 간단한 차트 생성
```bash
# 새 차트 생성
helm create mychart

# 차트 구조 확인
tree mychart/

# 차트 문법 검증
helm lint mychart/

# 차트 패키징
helm package mychart/

# 로컬에서 차트 설치
helm install test-release ./mychart
```

## Helm의 장점

### 1. 간편한 애플리케이션 관리
- 복잡한 쿠버네티스 매니페스트를 간단한 명령어로 관리
- 원클릭 설치, 업그레이드, 삭제 가능
- 롤백 기능으로 안전한 배포 관리

### 2. 템플릿화와 재사용성
- Go 템플릿 엔진을 사용한 동적 매니페스트 생성
- 환경별(개발/스테이징/프로덕션) 다른 설정값 적용 가능
- 차트 공유를 통한 베스트 프랙티스 활용

### 3. 버전 관리
- 애플리케이션 버전과 설정 변경사항 추적
- 릴리즈 히스토리 관리 및 롤백 지원
- 점진적 업그레이드와 안전한 배포

### 4. 의존성 관리
- 차트 간 의존성 정의 및 자동 해결
- 복잡한 애플리케이션 스택을 하나의 차트로 관리
- 서브차트를 통한 모듈화된 구조

### 5. 커뮤니티 생태계
- 수천 개의 공개 차트 사용 가능
- 검증된 베스트 프랙티스 활용
- 활발한 커뮤니티 지원

## 쿠버네티스와 Helm

### Helm이 쿠버네티스를 보완하는 방식

#### 1. 복잡성 해결
```yaml
# 기존 쿠버네티스 방식 (여러 개의 YAML 파일)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
        image: nginx:1.20
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
  - port: 80
    targetPort: 80
  type: ClusterIP
```

```bash
# Helm 방식 (한 줄 명령어)
helm install my-nginx bitnami/nginx
```

#### 2. 환경별 설정 관리
```yaml
# values-dev.yaml
replicaCount: 1
image:
  tag: "latest"
resources:
  requests:
    cpu: 100m
    memory: 128Mi

# values-prod.yaml
replicaCount: 5
image:
  tag: "1.20-stable"
resources:
  requests:
    cpu: 500m
    memory: 512Mi
```

```bash
# 개발 환경 배포
helm install dev-app ./mychart -f values-dev.yaml

# 프로덕션 환경 배포
helm install prod-app ./mychart -f values-prod.yaml
```

#### 3. 쿠버네티스 네이티브 통합
- Helm은 쿠버네티스 API를 직접 사용
- 생성된 리소스는 표준 쿠버네티스 객체
- `kubectl`로도 관리 가능하며 완전 호환

### Helm과 쿠버네티스의 관계

#### Helm 2 vs Helm 3 아키텍처
**Helm 2 (Deprecated)**
```
[Helm Client] → [Tiller (Server)] → [Kubernetes API]
```

**Helm 3 (Current)**
```
[Helm Client] → [Kubernetes API]
```

Helm 3는 Tiller를 제거하여 보안성을 개선하고 RBAC와 완전히 호환됩니다.

#### 주요 통합 포인트

1. **ConfigMap/Secret을 통한 릴리즈 정보 저장**
   ```bash
   # 릴리즈 정보 확인
   kubectl get configmap -l owner=helm
   kubectl get secret -l owner=helm
   ```

2. **네임스페이스 격리**
   ```bash
   # 특정 네임스페이스에 설치
   helm install my-app ./chart --namespace production --create-namespace
   ```

3. **RBAC 통합**
   ```yaml
   # ServiceAccount와 RBAC 설정
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: helm-user
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: helm-user-binding
   roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: ClusterRole
     name: cluster-admin
   subjects:
   - kind: ServiceAccount
     name: helm-user
     namespace: default
   ```

### 모범 사례

#### 1. 차트 개발 모범 사례
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.0"
description: My application Helm chart
keywords:
  - web
  - application
maintainers:
  - name: Your Name
    email: your-email@example.com
```

#### 2. 보안 고려사항
```bash
# 차트 서명 및 검증
helm package --sign --key mykey --keyring ~/.gnupg/secring.gpg mychart/
helm install --verify ./mychart-1.0.0.tgz

# RBAC 최소 권한 원칙
helm install my-app ./chart --create-namespace --namespace restricted
```

#### 3. CI/CD 통합
```yaml
# GitLab CI 예제
deploy:
  stage: deploy
  script:
    - helm upgrade --install $APP_NAME ./chart 
        --namespace $NAMESPACE 
        --set image.tag=$CI_COMMIT_SHA
        --wait
  only:
    - main
```

## 결론

Helm은 쿠버네티스 애플리케이션 관리를 혁신적으로 개선하는 도구입니다. 복잡한 매니페스트 관리부터 환경별 배포, 버전 관리까지 쿠버네티스 운영의 모든 측면을 간소화합니다.

특히 마이크로서비스 아키텍처나 복잡한 애플리케이션 스택을 관리할 때 Helm의 진가가 발휘되며, 개발팀의 생산성과 운영 안정성을 크게 향상시킬 수 있습니다.

Helm을 도입할 때는 조직의 쿠버네티스 성숙도와 요구사항을 고려하여 점진적으로 적용하는 것을 권장하며, 공식 문서와 커뮤니티 차트를 적극 활용하여 베스트 프랙티스를 학습하고 적용하는 것이 중요합니다.