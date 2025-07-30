# Helm 리서치

## 개요
Helm은 쿠버네티스 패키지 관리자로, 복잡한 쿠버네티스 애플리케이션을 쉽게 설치, 업그레이드, 롤백할 수 있도록 도와주는 도구입니다.

## Helm의 핵심 개념

### 1. Helm Chart
- 쿠버네티스 리소스들의 모음
- 애플리케이션을 배포하기 위한 모든 정보를 포함
- 템플릿화된 쿠버네티스 매니페스트 파일들의 집합

### 2. Helm Release
- Chart의 실행 중인 인스턴스
- 특정 네임스페이스에서 실행되는 애플리케이션
- 고유한 릴리스 이름으로 식별

### 3. Helm Repository
- Chart들을 저장하고 공유하는 저장소
- 공개 저장소와 프라이빗 저장소 지원

## Helm 아키텍처

### Helm 3.x (현재 버전)
- **Tiller 제거**: 서버 사이드 컴포넌트 제거
- **네임스페이스 기반**: 각 네임스페이스별로 독립적인 릴리스 관리
- **RBAC 통합**: 쿠버네티스 RBAC과 완전 통합
- **Secrets 기반**: 릴리스 정보를 쿠버네티스 Secrets에 저장

## 주요 기능

### 1. 패키지 관리
```bash
# Chart 설치
helm install my-release ./my-chart

# Chart 업그레이드
helm upgrade my-release ./my-chart

# Chart 삭제
helm uninstall my-release
```

### 2. 템플릿 엔진
- Go 템플릿 언어 사용
- 동적 값 주입 및 조건부 렌더링
- 재사용 가능한 템플릿 컴포넌트

### 3. 의존성 관리
- Chart 간의 의존성 정의
- 자동 의존성 해결 및 설치

### 4. 릴리스 관리
- 버전 관리 및 롤백 기능
- 릴리스 히스토리 추적
- 상태 모니터링

## Chart 구조

```
my-chart/
├── Chart.yaml          # Chart 메타데이터
├── values.yaml         # 기본 설정값
├── charts/             # 의존성 Charts
├── templates/          # 템플릿 파일들
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl
└── .helmignore         # 무시할 파일들
```

## 주요 명령어

### Chart 관리
```bash
# Chart 생성
helm create my-chart

# Chart 패키징
helm package my-chart

# Chart 검증
helm lint my-chart

# Chart 테스트
helm test my-release
```

### Repository 관리
```bash
# Repository 추가
helm repo add bitnami https://charts.bitnami.com/bitnami

# Repository 업데이트
helm repo update

# Repository 목록 조회
helm repo list
```

### 릴리스 관리
```bash
# 릴리스 목록 조회
helm list

# 릴리스 상태 확인
helm status my-release

# 릴리스 히스토리 조회
helm history my-release

# 릴리스 롤백
helm rollback my-release 1
```

## Values 관리

### values.yaml 예시
```yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.19.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
```

### 템플릿에서 Values 사용
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
```

## 고급 기능

### 1. Hooks
- 릴리스 라이프사이클의 특정 시점에서 실행되는 작업
- pre-install, post-install, pre-upgrade, post-upgrade 등

### 2. Tests
- Chart가 올바르게 설치되었는지 확인하는 테스트
- test 디렉토리에 정의

### 3. Library Charts
- 다른 Charts에서 재사용할 수 있는 공통 템플릿
- 애플리케이션 로직은 포함하지 않음

## 모범 사례

### 1. Chart 설계
- 단일 책임 원칙: 하나의 Chart는 하나의 애플리케이션
- 모듈화: 복잡한 애플리케이션은 여러 Chart로 분리
- 재사용성: 공통 컴포넌트는 Library Chart로 분리

### 2. Values 설계
- 명확한 기본값 제공
- 환경별 설정 분리
- 민감한 정보는 Secret 사용

### 3. 템플릿 작성
- 가독성 있는 템플릿 작성
- 적절한 주석 추가
- 에러 처리 및 검증 로직 포함

## 보안 고려사항

### 1. RBAC
- 최소 권한 원칙 적용
- 네임스페이스별 권한 분리
- ServiceAccount 사용

### 2. Secrets 관리
- 민감한 정보는 외부 Secret 관리 도구 사용
- Helm Secrets 플러그인 활용
- GitOps 환경에서의 안전한 배포

### 3. Chart 검증
- Chart 보안 스캔
- 취약점 검사
- 정적 분석 도구 활용

## CI/CD 통합

### 1. GitOps
- Chart 버전 관리
- 자동화된 배포 파이프라인
- 승인 워크플로우

### 2. 테스트 자동화
- Chart 유효성 검사
- 배포 후 테스트
- 롤백 자동화

## 모니터링 및 로깅

### 1. Helm 상태 모니터링
- 릴리스 상태 추적
- 배포 성공/실패 알림
- 메트릭 수집

### 2. 로깅
- Helm 작업 로그 수집
- 감사 추적
- 문제 해결을 위한 디버깅 정보

## 결론

Helm은 쿠버네티스 애플리케이션 배포를 단순화하고 표준화하는 강력한 도구입니다. 적절한 설계와 모범 사례를 따르면 복잡한 애플리케이션도 효율적으로 관리할 수 있습니다.

## 참고 자료
- [Helm 공식 문서](https://helm.sh/docs/)
- [Helm GitHub](https://github.com/helm/helm)
- [Helm Hub](https://hub.helm.sh/) 