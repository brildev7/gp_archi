# API 설계 명세서

## 📋 개요

AI 교육 플랫폼의 API 설계 명세서입니다. RESTful API와 WebSocket API를 통해 프론트엔드, 백엔드, AI 서비스 간의 통신을 정의합니다.

## 🏗️ API 아키텍처

### API Gateway 구조
```
클라이언트 → Azure API Management → 백엔드 서비스
                ↓
           인증/인가/Rate Limiting
                ↓
         라우팅 및 로드 밸런싱
                ↓
            서비스별 엔드포인트
```

### API 버전 관리
- **URL 버전 관리**: `/api/v1/`, `/api/v2/`
- **헤더 버전 관리**: `API-Version: 1.0`
- **하위 호환성**: 최소 2개 버전 동시 지원

## 🔐 인증 및 인가

### JWT 기반 인증
```yaml
헤더 형식:
  Authorization: Bearer <JWT_TOKEN>

토큰 구조:
  Header:
    alg: HS256
    typ: JWT
  
  Payload:
    sub: user_id
    role: teacher|student|admin
    school_id: school_identifier
    exp: expiration_time
    iat: issued_at
```

### 권한 레벨
```yaml
교사 (Teacher):
  - 레슨플랜 생성/수정/삭제
  - 컨텐츠 생성/관리
  - 학생 평가 및 피드백
  - 수업 세션 관리

학생 (Student):
  - 과제 제출
  - 수업 참여
  - 개인 학습 데이터 조회

관리자 (Admin):
  - 시스템 전체 관리
  - 사용자 관리
  - 통계 및 분석 데이터 접근
```

## 📚 핵심 API 엔드포인트

### 1. 인증 API

#### POST /api/v1/auth/login
교사/학생 로그인
```json
Request:
{
  "email": "teacher@school.edu",
  "password": "secure_password",
  "user_type": "teacher"
}

Response:
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
  "expires_in": 3600,
  "user": {
    "id": "user_123",
    "name": "김선생",
    "email": "teacher@school.edu",
    "role": "teacher",
    "school_id": "school_456"
  }
}
```

#### POST /api/v1/auth/refresh
토큰 갱신
```json
Request:
{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g..."
}

Response:
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "expires_in": 3600
}
```

### 2. 레슨플랜 API

#### POST /api/v1/lesson-plans
레슨플랜 생성
```json
Request:
{
  "title": "분수의 덧셈과 뺄셈",
  "subject": "mathematics",
  "grade_level": 4,
  "duration_minutes": 45,
  "learning_objectives": [
    "분수의 개념을 이해한다",
    "분수의 덧셈과 뺄셈을 할 수 있다"
  ],
  "student_characteristics": {
    "class_size": 25,
    "learning_level": "average",
    "special_needs": ["dyslexia"]
  },
  "teaching_environment": "classroom"
}

Response:
{
  "id": "lesson_789",
  "title": "분수의 덧셈과 뺄셈",
  "status": "generated",
  "created_at": "2024-01-15T09:00:00Z",
  "content": {
    "introduction": {
      "duration_minutes": 5,
      "activities": [...]
    },
    "main_activity": {
      "duration_minutes": 30,
      "activities": [...]
    },
    "conclusion": {
      "duration_minutes": 10,
      "activities": [...]
    }
  },
  "materials_needed": ["분수 막대", "화이트보드"],
  "assessment_methods": [...]
}
```

#### GET /api/v1/lesson-plans/{id}
레슨플랜 조회
```json
Response:
{
  "id": "lesson_789",
  "title": "분수의 덧셈과 뺄셈",
  "subject": "mathematics",
  "grade_level": 4,
  "created_by": "user_123",
  "created_at": "2024-01-15T09:00:00Z",
  "updated_at": "2024-01-15T09:30:00Z",
  "content": {...},
  "usage_analytics": {
    "views": 15,
    "implementations": 3,
    "average_rating": 4.5
  }
}
```

#### PUT /api/v1/lesson-plans/{id}
레슨플랜 수정
```json
Request:
{
  "title": "분수의 덧셈과 뺄셈 (심화)",
  "content": {
    "main_activity": {
      "duration_minutes": 35,
      "activities": [...]
    }
  }
}

Response:
{
  "id": "lesson_789",
  "status": "updated",
  "updated_at": "2024-01-15T10:00:00Z",
  "content": {...}
}
```

### 3. 컨텐츠 생성 API

#### POST /api/v1/content/generate
교육 컨텐츠 생성
```json
Request:
{
  "type": "worksheet",
  "topic": "분수의 덧셈",
  "grade_level": 4,
  "difficulty": "medium",
  "options": {
    "problem_count": 10,
    "include_visual_aids": true,
    "language": "ko"
  }
}

Response:
{
  "id": "content_456",
  "type": "worksheet",
  "status": "generating",
  "estimated_completion": "2024-01-15T09:05:00Z",
  "webhook_url": "/api/v1/webhooks/content-ready"
}
```

#### POST /api/v1/content/images/generate
교육용 이미지 생성
```json
Request:
{
  "description": "분수를 표현하는 피자 조각 그림",
  "style": "educational_illustration",
  "grade_level": 4,
  "cultural_context": "korean"
}

Response:
{
  "id": "image_789",
  "status": "generating",
  "estimated_completion": "2024-01-15T09:02:00Z",
  "preview_url": null
}
```

#### GET /api/v1/content/{id}
생성된 컨텐츠 조회
```json
Response:
{
  "id": "content_456",
  "type": "worksheet",
  "status": "completed",
  "content": {
    "title": "분수 덧셈 연습 문제",
    "problems": [
      {
        "id": 1,
        "question": "1/4 + 1/4 = ?",
        "visual_aid": "image_url",
        "answer": "2/4 = 1/2"
      }
    ]
  },
  "download_urls": {
    "pdf": "https://storage.../worksheet.pdf",
    "docx": "https://storage.../worksheet.docx"
  },
  "created_at": "2024-01-15T09:05:00Z"
}
```

### 4. 평가 API

#### POST /api/v1/assessments/auto-grade
자동 채점
```json
Request:
{
  "submission_id": "submission_123",
  "assignment_id": "assignment_456",
  "student_id": "student_789",
  "answers": [
    {
      "question_id": "q1",
      "answer": "2/4 = 1/2",
      "type": "text"
    },
    {
      "question_id": "q2",
      "answer": "학생이 작성한 에세이 내용...",
      "type": "essay"
    }
  ],
  "rubric_id": "rubric_101"
}

Response:
{
  "submission_id": "submission_123",
  "total_score": 85,
  "max_score": 100,
  "grading_details": [
    {
      "question_id": "q1",
      "score": 10,
      "max_score": 10,
      "feedback": "정답입니다!",
      "correct": true
    },
    {
      "question_id": "q2",
      "score": 75,
      "max_score": 90,
      "feedback": "논리적 구성이 잘 되어 있으나, 결론 부분을 보강하면 좋겠습니다.",
      "strengths": ["논리적 구성", "예시 활용"],
      "improvements": ["결론 보강", "어휘 다양성"]
    }
  ],
  "overall_feedback": "전반적으로 잘 작성되었습니다...",
  "graded_at": "2024-01-15T14:30:00Z"
}
```

#### GET /api/v1/assessments/analytics/{student_id}
학생 학습 분석
```json
Response:
{
  "student_id": "student_789",
  "analytics_period": "2024-01-01T00:00:00Z/2024-01-31T23:59:59Z",
  "performance_summary": {
    "average_score": 78.5,
    "improvement_trend": "positive",
    "completed_assignments": 15,
    "total_assignments": 18
  },
  "subject_performance": [
    {
      "subject": "mathematics",
      "average_score": 82,
      "trend": "stable",
      "weak_areas": ["분수 계산", "문제 해석"],
      "strong_areas": ["기본 연산", "도형 이해"]
    }
  ],
  "learning_recommendations": [
    "분수 개념 보강 학습 권장",
    "문제 해석 능력 향상을 위한 독해 연습"
  ]
}
```

### 5. 실시간 수업 API

#### POST /api/v1/sessions
수업 세션 생성
```json
Request:
{
  "lesson_plan_id": "lesson_789",
  "class_id": "class_101",
  "scheduled_start": "2024-01-15T09:00:00Z",
  "duration_minutes": 45,
  "session_type": "live_class"
}

Response:
{
  "session_id": "session_456",
  "status": "created",
  "join_urls": {
    "teacher": "https://platform.../session/456?role=teacher&token=...",
    "student": "https://platform.../session/456?role=student&token=..."
  },
  "websocket_endpoint": "wss://platform.../session/456/ws",
  "created_at": "2024-01-15T08:55:00Z"
}
```

#### POST /api/v1/sessions/{id}/start
수업 세션 시작
```json
Response:
{
  "session_id": "session_456",
  "status": "active",
  "started_at": "2024-01-15T09:00:00Z",
  "participants": {
    "teacher_count": 1,
    "student_count": 23
  }
}
```

## 🔄 WebSocket API

### 실시간 통신 엔드포인트
```
연결: wss://api.platform.com/ws/session/{session_id}?token={jwt_token}
```

### 메시지 형식
```json
{
  "type": "message_type",
  "data": {...},
  "timestamp": "2024-01-15T09:00:00Z",
  "sender_id": "user_123",
  "session_id": "session_456"
}
```

### 주요 메시지 타입

#### 수업 컨텐츠 스트리밍
```json
{
  "type": "content_stream",
  "data": {
    "content_type": "lesson_explanation",
    "chunk": "분수는 전체를 나눈 부분을 나타내는 수입니다...",
    "chunk_id": "chunk_001",
    "is_final": false
  }
}
```

#### 학생 응답 수집
```json
{
  "type": "student_response",
  "data": {
    "question_id": "q1",
    "answer": "1/2",
    "response_time": 15,
    "confidence_level": "high"
  }
}
```

#### 실시간 피드백
```json
{
  "type": "instant_feedback",
  "data": {
    "target_student": "student_789",
    "feedback": "훌륭합니다! 정답입니다.",
    "feedback_type": "positive",
    "display_duration": 3000
  }
}
```

#### AI 어시스턴트 메시지
```json
{
  "type": "ai_assistant",
  "data": {
    "suggestion": "현재 30%의 학생이 어려워하고 있습니다. 추가 설명을 제공하는 것이 좋겠습니다.",
    "action_type": "teaching_suggestion",
    "priority": "medium",
    "auto_dismiss": false
  }
}
```

## 📊 AI 서비스 간 내부 API

### AI 서비스 공통 인터페이스
```yaml
Base URL: http://internal-ai-gateway:8000
헤더: 
  Content-Type: application/json
  X-Service-Token: <internal_service_token>
  X-Request-ID: <unique_request_id>
```

### 레슨플랜 AI 서비스

#### POST /ai/lesson-plan/generate
```json
Request:
{
  "context": {
    "subject": "mathematics",
    "grade_level": 4,
    "topic": "분수의 덧셈",
    "curriculum_standards": ["4-가-03-01", "4-가-03-02"],
    "class_profile": {
      "size": 25,
      "learning_styles": ["visual", "kinesthetic"],
      "special_needs": []
    }
  },
  "options": {
    "include_differentiation": true,
    "assessment_type": "formative",
    "technology_integration": true
  }
}

Response:
{
  "plan_id": "ai_plan_123",
  "generation_status": "completed",
  "content": {
    "lesson_structure": {...},
    "activities": [...],
    "assessment": {...}
  },
  "metadata": {
    "generation_time": 12.5,
    "model_version": "gpt-4-turbo",
    "confidence_score": 0.92
  }
}
```

### 컨텐츠 생성 AI 서비스

#### POST /ai/content/generate-text
```json
Request:
{
  "prompt": "4학년 수준의 분수 덧셈 설명 자료를 만들어주세요",
  "parameters": {
    "max_tokens": 500,
    "temperature": 0.7,
    "grade_level": 4,
    "content_type": "explanation"
  },
  "context": {
    "previous_content": "...",
    "learning_objectives": [...]
  }
}

Response:
{
  "content_id": "ai_content_456",
  "generated_text": "분수는 전체를 같은 크기로 나눈 것 중에서...",
  "metadata": {
    "word_count": 245,
    "reading_level": "grade_4",
    "generation_time": 3.2
  }
}
```

### 평가 AI 서비스

#### POST /ai/assessment/grade
```json
Request:
{
  "submission": {
    "student_answer": "1/4 + 1/4 = 2/4 = 1/2",
    "question": "1/4 + 1/4를 계산하세요",
    "expected_answer": "1/2"
  },
  "rubric": {
    "criteria": [
      {
        "name": "calculation_accuracy",
        "weight": 0.7,
        "max_score": 10
      },
      {
        "name": "process_shown",
        "weight": 0.3,
        "max_score": 10
      }
    ]
  }
}

Response:
{
  "grading_result": {
    "total_score": 9.5,
    "max_score": 10,
    "criteria_scores": [
      {
        "name": "calculation_accuracy",
        "score": 10,
        "feedback": "정확한 계산입니다"
      },
      {
        "name": "process_shown",
        "score": 9,
        "feedback": "과정을 잘 보여주었으나 약분 과정을 더 명확히 하면 좋겠습니다"
      }
    ],
    "overall_feedback": "훌륭한 답안입니다..."
  }
}
```

## 🚨 에러 처리

### 표준 에러 응답 형식
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 데이터가 올바르지 않습니다",
    "details": {
      "field": "grade_level",
      "reason": "1-12 사이의 값이어야 합니다"
    },
    "timestamp": "2024-01-15T09:00:00Z",
    "request_id": "req_123456"
  }
}
```

### HTTP 상태 코드
```yaml
200 OK: 성공적인 요청
201 Created: 리소스 생성 성공
202 Accepted: 비동기 작업 수락
400 Bad Request: 잘못된 요청
401 Unauthorized: 인증 실패
403 Forbidden: 권한 없음
404 Not Found: 리소스 없음
429 Too Many Requests: 속도 제한 초과
500 Internal Server Error: 서버 오류
502 Bad Gateway: AI 서비스 오류
503 Service Unavailable: 서비스 일시 중단
```

### 에러 코드 분류
```yaml
인증/인가 에러:
  - AUTH_REQUIRED: 인증 필요
  - TOKEN_EXPIRED: 토큰 만료
  - INSUFFICIENT_PERMISSIONS: 권한 부족

요청 데이터 에러:
  - VALIDATION_ERROR: 데이터 검증 실패
  - MISSING_REQUIRED_FIELD: 필수 필드 누락
  - INVALID_FORMAT: 잘못된 형식

AI 서비스 에러:
  - AI_SERVICE_UNAVAILABLE: AI 서비스 이용 불가
  - CONTENT_GENERATION_FAILED: 컨텐츠 생성 실패
  - QUOTA_EXCEEDED: API 할당량 초과

시스템 에러:
  - INTERNAL_ERROR: 내부 서버 오류
  - DATABASE_ERROR: 데이터베이스 오류
  - EXTERNAL_SERVICE_ERROR: 외부 서비스 오류
```

## 📈 API 성능 및 제한

### Rate Limiting
```yaml
일반 사용자:
  - 분당 100 요청
  - 시간당 1,000 요청
  - 일일 10,000 요청

프리미엄 사용자:
  - 분당 500 요청
  - 시간당 5,000 요청
  - 일일 50,000 요청

AI 컨텐츠 생성:
  - 분당 10 요청 (무료)
  - 분당 50 요청 (프리미엄)
```

### 응답 시간 목표
```yaml
일반 API:
  - 평균: < 200ms
  - 95%: < 500ms
  - 99%: < 1000ms

AI 생성 API:
  - 텍스트 생성: < 10초
  - 이미지 생성: < 30초
  - 레슨플랜 생성: < 60초

실시간 WebSocket:
  - 메시지 지연: < 100ms
  - 연결 설정: < 2초
```

## 🔧 API 테스트

### 테스트 환경
```yaml
개발 환경:
  Base URL: https://api-dev.platform.com
  
스테이징 환경:
  Base URL: https://api-staging.platform.com
  
프로덕션 환경:
  Base URL: https://api.platform.com
```

### Postman 컬렉션
```json
{
  "info": {
    "name": "AI Education Platform API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "auth": {
    "type": "bearer",
    "bearer": [
      {
        "key": "token",
        "value": "{{auth_token}}",
        "type": "string"
      }
    ]
  },
  "variable": [
    {
      "key": "base_url",
      "value": "https://api-dev.platform.com"
    }
  ]
}
```

이 API 설계는 확장 가능하고 유지보수가 용이하며, 교육 환경의 특수한 요구사항을 충족하도록 구성되었습니다. 