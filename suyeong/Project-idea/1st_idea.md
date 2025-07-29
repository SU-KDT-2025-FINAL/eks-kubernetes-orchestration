# 핵심 서비스 아이디어
- 중고거래 플랫폼 경매버전

## MSA 서비스 구성

### 🔐 인증/사용자 관리 서비스
- **User Service**: 회원가입, 로그인, 프로필 관리
- **Auth Service**: JWT 토큰 발급, 인증/인가 처리
- **Notification Service**: 이메일, SMS, 푸시 알림

### 🛒 상품 관리 서비스
- **Product Service**: 상품 등록, 수정, 조회, 카테고리 관리
- **Image Service**: 상품 이미지 업로드, 리사이징, CDN 연동
- **Search Service**: Elasticsearch 기반 상품 검색, 필터링

### 💰 경매 관리 서비스
- **Auction Service**: 경매 생성, 입찰, 낙찰 처리
- **Bidding Service**: 실시간 입찰 처리, WebSocket 연동
- **Payment Service**: 결제 처리, PG사 연동, 환불 관리

### 📦 거래 관리 서비스
- **Order Service**: 주문 생성, 상태 관리, 주문 이력
- **Shipping Service**: 배송 정보 관리, 택배사 연동
- **Review Service**: 거래 후기, 평점 관리

### 🔍 부가 서비스
- **Chat Service**: 판매자-구매자 실시간 채팅
- **Analytics Service**: 사용자 행동 분석, 통계
- **Admin Service**: 관리자 대시보드, 신고 처리

## MSA 시스템 유형

### 🏗️ 아키텍처 패턴
- **API Gateway**: Kong 또는 AWS API Gateway
- **Service Mesh**: Istio를 통한 서비스 간 통신 관리
- **Event-Driven Architecture**: 경매 입찰, 결제 등 이벤트 기반 처리

### 💾 데이터 저장소
- **MySQL**: 사용자, 상품, 주문 등 관계형 데이터
- **Redis**: 세션, 캐시, 실시간 입찰 데이터
- **MongoDB**: 채팅 메시지, 로그 데이터
- **S3**: 상품 이미지, 파일 저장

### 🔄 메시징 시스템
- **Apache Kafka**: 이벤트 스트리밍, 서비스 간 비동기 통신
- **RabbitMQ**: 실시간 알림, 이메일 발송 큐
- **WebSocket**: 실시간 입찰, 채팅

### 📊 모니터링 & 로깅
- **Prometheus + Grafana**: 메트릭 수집 및 대시보드
- **ELK Stack**: 로그 수집, 검색, 분석
- **Jaeger**: 분산 트레이싱

### 🚀 배포 & 운영
- **Container**: Docker 기반 컨테이너화
- **Orchestration**: Kubernetes (EKS)
- **CI/CD**: GitLab CI 또는 GitHub Actions
- **IaC**: Terraform을 통한 인프라 관리

## Kubernetes 중심 아키텍처 설계

### 🎯 Kubernetes 리소스 구성

#### Core Workloads
- **Deployment**: 각 마이크로서비스 Pod 관리
  - User Service, Product Service, Auction Service 등 각각 독립 Deployment
  - Rolling Update 전략으로 무중단 배포
  - Resource Limits/Requests로 리소스 관리

- **StatefulSet**: 상태가 있는 서비스
  - MySQL, MongoDB 클러스터
  - 순서 보장 및 안정적인 네트워크 식별자

- **DaemonSet**: 모든 노드에서 실행되는 서비스
  - 로그 수집 Agent (Fluent Bit)
  - 모니터링 Agent (Node Exporter)

#### Networking
- **Service**: 서비스 디스커버리 및 로드밸런싱
  - ClusterIP: 내부 서비스 간 통신
  - LoadBalancer: 외부 접근이 필요한 API Gateway
  - NodePort: 개발/테스트 환경 접근

- **Ingress**: HTTP/HTTPS 라우팅
  - 도메인별 서비스 라우팅 (api.auction.com, admin.auction.com)
  - SSL/TLS 인증서 관리
  - Rate Limiting 및 보안 정책

#### Configuration & Secrets
- **ConfigMap**: 환경별 설정 관리
  - Database 연결 정보, API 엔드포인트
  - 각 서비스별 환경 변수

- **Secret**: 민감한 정보 관리
  - JWT 시크릿 키, 데이터베이스 비밀번호
  - PG사 API 키, 외부 서비스 인증 정보

#### Storage
- **PersistentVolume**: 영구 저장소
  - 데이터베이스 데이터 볼륨
  - 파일 업로드 임시 저장소

- **StorageClass**: 동적 볼륨 프로비저닝
  - SSD, HDD 성능별 스토리지 클래스 구분

### 🔄 Kubernetes 네이티브 기능 활용

#### Auto Scaling
- **Horizontal Pod Autoscaler (HPA)**
  - CPU/Memory 사용률 기반 Pod 자동 확장
  - 경매 시간대 트래픽 증가 대응

- **Vertical Pod Autoscaler (VPA)**
  - 리소스 사용 패턴 분석 후 적정 크기 추천

- **Cluster Autoscaler**
  - 노드 부족 시 자동으로 워커 노드 추가

#### Service Mesh (Istio)
- **Traffic Management**
  - 카나리 배포, A/B 테스트
  - Circuit Breaker 패턴으로 장애 전파 차단

- **Security**
  - mTLS로 서비스 간 암호화 통신
  - 네트워크 정책 기반 접근 제어

- **Observability**
  - 서비스 간 호출 추적
  - 메트릭 자동 수집

### 📊 Kubernetes 모니터링 스택

#### Prometheus 생태계
- **Prometheus Operator**: 선언적 모니터링 설정
- **ServiceMonitor**: 각 마이크로서비스 메트릭 수집 정의
- **AlertManager**: SLA 위반 시 알림 발송

#### 로깅 파이프라인
- **Fluent Bit**: 각 노드에서 로그 수집 (DaemonSet)
- **Elasticsearch**: 로그 저장 및 인덱싱 (StatefulSet)
- **Kibana**: 로그 검색 및 대시보드 (Deployment)

### 🎁 Helm Chart 구조

#### Umbrella Chart
```
auction-platform/
├── Chart.yaml
├── values.yaml
├── charts/
│   ├── user-service/
│   ├── auction-service/
│   ├── payment-service/
│   └── shared-services/
└── templates/
    ├── namespace.yaml
    ├── ingress.yaml
    └── monitoring/
```

#### 환경별 배포
- **Development**: 단일 노드, 최소 리소스
- **Staging**: 프로덕션 유사 환경, 부하 테스트
- **Production**: 멀티 AZ, 고가용성 구성

### 🔐 보안 고려사항

#### Pod Security
- **Pod Security Standards**: Baseline/Restricted 정책 적용
- **Network Policies**: 서비스 간 통신 제한
- **RBAC**: 서비스별 최소 권한 원칙

#### Secrets 관리
- **External Secrets Operator**: AWS Secrets Manager 연동
- **Sealed Secrets**: GitOps를 위한 암호화된 Secret

### 🎯 Kubernetes 학습 포인트

#### 실습할 수 있는 시나리오
1. **서비스 배포**: Deployment → Service → Ingress 순서로 배포
2. **설정 관리**: ConfigMap 변경 후 Pod 재시작 확인
3. **스케일링**: 경매 시작 시 Auction Service HPA 동작 확인
4. **장애 복구**: Pod 삭제 후 자동 복구 과정 관찰
5. **업데이트**: Rolling Update vs Blue-Green 배포 비교
6. **모니터링**: Prometheus 메트릭으로 서비스 상태 확인

#### Kubernetes 고급 기능 활용
- **CRD**: 경매 상태를 관리하는 Custom Resource 정의
- **Operator**: 경매 생명주기를 자동 관리하는 Operator 개발
- **Admission Controller**: 배포 시 보안 정책 자동 검증 