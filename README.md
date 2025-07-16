# AI 기반 교육 플랫폼 아키텍처

## 📋 프로젝트 개요

Azure 기반의 AI 교육 플랫폼으로, OpenAI API를 활용하여 교사들에게 지능형 교육 도구를 제공합니다.

## 🏗️ 아키텍처 구성

### 1. 전체 시스템 아키텍처
- **프론트엔드**: React/Next.js (웹), React Native (모바일)
- **백엔드**: Azure App Services (마이크로서비스 아키텍처)
- **AI 백엔드**: Azure Container Instances + FastAPI
- **데이터베이스**: Azure SQL Database, Cosmos DB, Redis Cache
- **스토리지**: Azure Blob Storage
- **AI API**: OpenAI GPT-4/3.5-turbo, Azure OpenAI Service

### 2. 핵심 AI 기능
- 🎯 **레슨플랜 생성**: RAG 기반 커리큘럼 표준 활용
- 📝 **교육 컨텐츠 생성**: 텍스트, 이미지, 퀴즈 자동 생성
- 📊 **자동 평가**: AI 기반 과제 채점 및 피드백
- 🤖 **실시간 교수 지원**: 수업 중 AI 어시스턴트
- 🌊 **실시간 스트리밍**: 동적 컨텐츠 생성 및 배포

### 3. 실시간 스트리밍 전략
- **WebSocket + SignalR**: 실시간 양방향 통신
- **GPT Streaming API**: 점진적 컨텐츠 생성
- **Redis Pipeline**: 고성능 캐싱 및 세션 관리
- **Azure Media Services**: 멀티미디어 스트리밍

## 🚀 핵심 기술 스택

### Backend Services
```
- FastAPI (Python 3.11+)
- Azure Container Instances
- Celery + Redis (비동기 작업)
- SQLAlchemy (ORM)
- Pydantic (데이터 검증)
```

### AI & ML
```
- OpenAI API (GPT-4, GPT-3.5-turbo, DALL-E 3)
- Azure OpenAI Service
- Azure Cognitive Search (벡터 검색)
- LangChain (RAG 구현)
- Sentence Transformers (임베딩)
```

### Infrastructure
```
- Azure API Management
- Azure Application Gateway
- Azure SQL Database
- Azure Cosmos DB
- Azure Redis Cache
- Azure Blob Storage
```

## 📂 프로젝트 구조

```
gp_ai_archi/
├── frontend/                 # React/Next.js 프론트엔드
├── mobile/                   # React Native 모바일 앱
├── backend/                  # 백엔드 마이크로서비스
│   ├── user-service/         # 사용자 관리 서비스
│   ├── content-service/      # 컨텐츠 관리 서비스
│   ├── lesson-service/       # 레슨 관리 서비스
│   └── streaming-service/    # 스트리밍 서비스
├── ai-backend/               # AI 백엔드 서비스
│   ├── lesson-plan-ai/       # 레슨플랜 생성 AI
│   ├── content-gen-ai/       # 컨텐츠 생성 AI
│   ├── assessment-ai/        # 평가 AI
│   ├── teaching-agent/       # 교수 지원 AI
│   └── streaming-ai/         # 실시간 스트리밍 AI
├── shared/                   # 공통 라이브러리
│   ├── database/             # 데이터베이스 모델
│   ├── auth/                 # 인증/인가
│   ├── monitoring/           # 모니터링
│   └── utils/                # 유틸리티
├── infrastructure/           # Infrastructure as Code
│   ├── terraform/            # Terraform 설정
│   ├── docker/               # Docker 설정
│   └── kubernetes/           # K8s 매니페스트
└── docs/                     # 문서
```

## 🔧 개발 환경 설정

### 1. 환경 변수 설정
```env
# OpenAI API
OPENAI_API_KEY=your_openai_api_key
OPENAI_ORG_ID=your_organization_id

# Azure Services
AZURE_OPENAI_ENDPOINT=your_azure_openai_endpoint
AZURE_OPENAI_API_KEY=your_azure_openai_key
AZURE_STORAGE_CONNECTION_STRING=your_storage_connection
AZURE_SQL_CONNECTION_STRING=your_sql_connection

# Redis
REDIS_URL=your_redis_connection_string

# Other Services
JWT_SECRET_KEY=your_jwt_secret
API_BASE_URL=your_api_base_url
```

### 2. 로컬 개발 실행
```bash
# AI 백엔드 서비스 실행
cd ai-backend
docker-compose up -d

# 백엔드 서비스 실행  
cd backend
docker-compose up -d

# 프론트엔드 실행
cd frontend
npm install
npm run dev
```

## 📊 성능 최적화 전략

### 1. AI API 최적화
- **응답 캐싱**: Redis를 활용한 다층 캐싱
- **배치 처리**: Celery를 활용한 비동기 작업
- **스트리밍**: 점진적 결과 반환
- **로드 밸런싱**: 여러 AI 서비스 인스턴스 운영

### 2. 실시간 스트리밍 최적화
- **WebSocket 풀링**: 연결 재사용
- **컨텐츠 사전 생성**: 예상 가능한 컨텐츠 미리 생성
- **CDN 활용**: Azure CDN을 통한 전역 배포
- **압축**: 실시간 데이터 압축

## 🔒 보안 고려사항

### 1. API 보안
- Azure API Management를 통한 Rate Limiting
- JWT 기반 인증/인가
- HTTPS 강제 적용
- CORS 정책 설정

### 2. 데이터 보안
- Azure Key Vault를 통한 시크릿 관리
- 데이터 암호화 (전송/저장)
- 개인정보 익명화
- GDPR 준수

## 📈 모니터링 및 로깅

### 1. 애플리케이션 모니터링
- Azure Application Insights
- 사용자 정의 메트릭 수집
- 성능 알림 설정

### 2. AI 서비스 모니터링
- OpenAI API 사용량 모니터링
- 응답 시간 추적
- 오류율 모니터링
- 비용 최적화 알림

## 🚦 배포 전략

### 1. CI/CD 파이프라인
- GitHub Actions를 활용한 자동 배포
- 단계별 배포 (dev → staging → production)
- 자동화된 테스트 실행

### 2. Blue-Green 배포
- 무중단 배포 구현
- 롤백 전략 수립
- 카나리 배포 활용

## 📚 추가 문서

- [API 문서](./docs/api.md)
- [AI 모델 가이드](./docs/ai-models.md)
- [배포 가이드](./docs/deployment.md)
- [문제 해결 가이드](./docs/troubleshooting.md)

## 🤝 기여 가이드

1. Fork the repository
2. Create a feature branch
3. Commit your changes  
4. Push to the branch
5. Create a Pull Request

## 📝 라이선스

MIT License - 자세한 내용은 [LICENSE](LICENSE) 파일을 참조하세요. 