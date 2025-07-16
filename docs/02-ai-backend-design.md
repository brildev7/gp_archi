# AI 백엔드 상세 설계

## 📋 개요

교육 플랫폼의 AI 백엔드는 교사들의 수업 준비와 진행을 지원하는 5개의 핵심 AI 서비스로 구성됩니다. 각 서비스는 독립적으로 배포되고 확장 가능하며, OpenAI API와 Azure OpenAI Service를 활용합니다.

## 🎯 핵심 AI 서비스

### 1. 레슨플랜 생성 AI 서비스

#### 기능 개요
- 교육과정 표준 기반 수업 계획 자동 생성
- RAG(Retrieval-Augmented Generation) 기반 개인화된 계획 수립
- 실시간 수정 및 개선 제안

#### 기술 스택
```yaml
AI 모델:
  - 주 모델: OpenAI GPT-4 (복잡한 계획 수립)
  - 보조 모델: GPT-3.5-turbo (빠른 수정 작업)
  - 임베딩: text-embedding-ada-002

데이터 저장:
  - Vector DB: Azure Cognitive Search
  - 구조화 데이터: Azure SQL Database
  - 캐시: Redis (계획 템플릿)
```

#### 아키텍처 구성
```
사용자 요청 → 입력 검증 → 커리큘럼 검색 (RAG)
    ↓
컨텍스트 구성 → GPT-4 호출 → 결과 후처리
    ↓
구조화 저장 → 벡터 인덱싱 → 사용자 응답
```

#### 데이터 모델
```python
class LessonPlan:
    id: str
    title: str
    subject: str
    grade_level: int
    duration_minutes: int
    learning_objectives: List[str]
    activities: List[Activity]
    assessment_methods: List[str]
    materials_needed: List[str]
    differentiation_strategies: List[str]
    created_by: str
    created_at: datetime
    updated_at: datetime

class Activity:
    name: str
    description: str
    duration_minutes: int
    activity_type: ActivityType
    materials: List[str]
    instructions: str
```

### 2. 컨텐츠 생성 AI 서비스

#### 기능 개요
- 텍스트 기반 교육 자료 생성 (워크시트, 설명서)
- 이미지 및 시각 자료 생성 (DALL-E 3)
- 퀴즈 및 평가 문항 자동 생성
- 다양한 학습 수준별 맞춤형 컨텐츠

#### 기술 스택
```yaml
AI 모델:
  - 텍스트: GPT-4, GPT-3.5-turbo
  - 이미지: DALL-E 3
  - 퀴즈: GPT-4 (구조화된 출력)

저장소:
  - 생성 컨텐츠: Azure Blob Storage
  - 메타데이터: Azure Cosmos DB
  - 템플릿 캐시: Redis
```

#### 컨텐츠 유형별 생성 전략

##### 텍스트 컨텐츠
```python
class ContentGenerator:
    def generate_worksheet(self, topic, grade_level, difficulty):
        prompt = self._build_worksheet_prompt(topic, grade_level, difficulty)
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "system", "content": WORKSHEET_SYSTEM_PROMPT},
                     {"role": "user", "content": prompt}],
            temperature=0.7
        )
        return self._parse_worksheet_response(response)
```

##### 이미지 컨텐츠
```python
def generate_educational_image(self, description, style="educational illustration"):
    enhanced_prompt = f"Educational illustration for {description}, "
    enhanced_prompt += "clear, simple, child-friendly, high contrast, "
    enhanced_prompt += "suitable for classroom display"
    
    response = openai.images.generate(
        model="dall-e-3",
        prompt=enhanced_prompt,
        size="1024x1024",
        quality="standard",
        n=1
    )
    return response.data[0].url
```

### 3. 평가 AI 서비스

#### 기능 개요
- 자동 과제 채점 (주관식, 서술형)
- 개인화된 피드백 생성
- 학습 진도 분석 및 추천
- 루브릭 기반 평가

#### 기술 스택
```yaml
AI 모델:
  - 채점: GPT-4 (정확도 중시)
  - 피드백: GPT-3.5-turbo (빠른 생성)
  - 분석: Azure OpenAI + Custom Models

평가 데이터:
  - 채점 결과: Azure SQL Database
  - 학습 패턴: Azure Cosmos DB
  - 실시간 분석: Redis
```

#### 자동 채점 알고리즘
```python
class AutoGrader:
    def grade_essay(self, essay_text, rubric, reference_answer):
        grading_prompt = self._build_grading_prompt(
            essay_text, rubric, reference_answer
        )
        
        response = openai.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": GRADING_SYSTEM_PROMPT},
                {"role": "user", "content": grading_prompt}
            ],
            temperature=0.1,  # 일관성을 위해 낮은 temperature
            response_format={"type": "json_object"}
        )
        
        return self._parse_grading_response(response)

    def generate_feedback(self, essay_text, grade, strengths, weaknesses):
        feedback_prompt = self._build_feedback_prompt(
            essay_text, grade, strengths, weaknesses
        )
        
        return openai.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": FEEDBACK_SYSTEM_PROMPT},
                {"role": "user", "content": feedback_prompt}
            ],
            temperature=0.7
        )
```

### 4. 교수 지원 에이전트

#### 기능 개요
- 실시간 교육 질문 응답
- 수업 중 즉석 도움말 제공
- 학생 질문에 대한 답변 제안
- 교육학적 조언 및 팁 제공

#### 기술 스택
```yaml
AI 모델:
  - 대화형 AI: GPT-4 (복잡한 교육 질문)
  - 빠른 응답: GPT-3.5-turbo (간단한 질문)
  - 메모리: LangChain ConversationBufferMemory

지식 베이스:
  - 교육학 지식: Azure Cognitive Search
  - 대화 히스토리: Redis
  - 교사 프로필: Azure SQL Database
```

#### 대화 관리 시스템
```python
class TeachingAgent:
    def __init__(self):
        self.memory = ConversationBufferMemory(
            memory_key="chat_history",
            return_messages=True
        )
        self.knowledge_base = AzureCognitiveSearchRetriever()
    
    async def handle_teacher_query(self, teacher_id, query, context):
        # 1. 컨텍스트 정보 수집
        teacher_profile = await self.get_teacher_profile(teacher_id)
        relevant_knowledge = await self.knowledge_base.search(query)
        
        # 2. 대화 히스토리 로드
        chat_history = self.memory.load_memory_variables({})
        
        # 3. AI 응답 생성
        enhanced_prompt = self._build_enhanced_prompt(
            query, teacher_profile, relevant_knowledge, context
        )
        
        response = await openai.chat.completions.acreate(
            model="gpt-4",
            messages=enhanced_prompt,
            temperature=0.7,
            max_tokens=1000
        )
        
        # 4. 메모리 업데이트
        self.memory.save_context(
            {"input": query},
            {"output": response.choices[0].message.content}
        )
        
        return response.choices[0].message.content
```

### 5. 실시간 스트리밍 AI 서비스

#### 기능 개요
- 수업 중 실시간 컨텐츠 생성
- 학생 반응 기반 적응형 컨텐츠 조정
- 즉석 시각 자료 생성
- 실시간 번역 및 자막 제공

#### 기술 스택
```yaml
실시간 처리:
  - WebSocket: 양방향 통신
  - Azure SignalR: 확장성 있는 실시간 연결
  - Redis Streams: 실시간 데이터 파이프라인

AI 모델:
  - Streaming: OpenAI GPT-4 Streaming
  - 즉석 생성: GPT-3.5-turbo (저지연)
  - 번역: Azure Translator Text API
```

#### 스트리밍 구현
```python
class StreamingAIService:
    async def stream_content_generation(self, websocket, prompt):
        stream = await openai.chat.completions.acreate(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            stream=True,
            temperature=0.8
        )
        
        async for chunk in stream:
            if chunk.choices[0].delta.content is not None:
                content = chunk.choices[0].delta.content
                await websocket.send_text(json.dumps({
                    "type": "content_chunk",
                    "data": content,
                    "timestamp": time.time()
                }))
        
        await websocket.send_text(json.dumps({
            "type": "generation_complete",
            "timestamp": time.time()
        }))

    async def adapt_content_based_on_feedback(self, original_content, feedback):
        adaptation_prompt = f"""
        원본 컨텐츠: {original_content}
        학생 피드백: {feedback}
        
        피드백을 바탕으로 컨텐츠를 개선해주세요.
        """
        
        return await openai.chat.completions.acreate(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": adaptation_prompt}],
            temperature=0.7,
            max_tokens=500
        )
```

## 🔄 서비스 간 통신

### API Gateway 패턴
```python
class AIServiceOrchestrator:
    def __init__(self):
        self.lesson_plan_service = LessonPlanService()
        self.content_service = ContentGenerationService()
        self.assessment_service = AssessmentService()
        self.teaching_agent = TeachingAgentService()
        self.streaming_service = StreamingAIService()
    
    async def create_comprehensive_lesson(self, lesson_request):
        # 1. 레슨플랜 생성
        lesson_plan = await self.lesson_plan_service.generate(lesson_request)
        
        # 2. 관련 컨텐츠 생성 (병렬 처리)
        tasks = [
            self.content_service.generate_worksheet(lesson_plan),
            self.content_service.generate_presentation(lesson_plan),
            self.assessment_service.generate_quiz(lesson_plan)
        ]
        
        worksheet, presentation, quiz = await asyncio.gather(*tasks)
        
        return {
            "lesson_plan": lesson_plan,
            "materials": {
                "worksheet": worksheet,
                "presentation": presentation,
                "quiz": quiz
            }
        }
```

### 이벤트 기반 아키텍처
```python
class AIEventHandler:
    async def handle_lesson_created(self, event):
        lesson_id = event.data["lesson_id"]
        
        # 비동기적으로 관련 작업 실행
        await asyncio.gather(
            self._generate_supplementary_materials(lesson_id),
            self._index_lesson_for_search(lesson_id),
            self._notify_relevant_teachers(lesson_id)
        )
    
    async def handle_student_submission(self, event):
        submission = event.data["submission"]
        
        # 자동 채점 및 피드백 생성
        grading_task = asyncio.create_task(
            self.assessment_service.grade_submission(submission)
        )
        
        # 학습 패턴 분석
        analysis_task = asyncio.create_task(
            self.analytics_service.analyze_learning_pattern(submission)
        )
        
        await asyncio.gather(grading_task, analysis_task)
```

## 🎨 프롬프트 엔지니어링

### 시스템 프롬프트 설계
```python
LESSON_PLAN_SYSTEM_PROMPT = """
당신은 경험이 풍부한 교육과정 전문가입니다. 다음 지침을 따라 수업 계획을 작성해주세요:

1. 학습 목표는 구체적이고 측정 가능해야 합니다
2. 다양한 학습 스타일을 고려한 활동을 포함하세요
3. 학생 참여를 극대화하는 상호작용적 요소를 포함하세요
4. 평가 방법은 학습 목표와 일치해야 합니다
5. 차별화된 교수법을 제안하세요

출력 형식: JSON
"""

GRADING_SYSTEM_PROMPT = """
당신은 공정하고 전문적인 교사입니다. 다음 기준으로 학생 답안을 평가해주세요:

1. 정확성: 내용의 정확도
2. 완성도: 요구사항 충족도  
3. 논리성: 논리적 구성과 흐름
4. 창의성: 독창적 아이디어 (해당되는 경우)

건설적이고 격려적인 피드백을 제공하세요.
출력 형식: JSON (점수, 강점, 개선점, 피드백)
"""
```

### 동적 프롬프트 생성
```python
class PromptBuilder:
    def build_lesson_plan_prompt(self, subject, grade, topic, requirements):
        base_prompt = f"""
        과목: {subject}
        학년: {grade}
        주제: {topic}
        
        요구사항:
        """
        
        for req in requirements:
            base_prompt += f"- {req}\n"
        
        if grade <= 3:
            base_prompt += "\n저학년 수준에 맞는 간단한 활동을 포함하세요."
        elif grade >= 10:
            base_prompt += "\n고등학교 수준의 심화 학습을 포함하세요."
        
        return base_prompt
```

## 📊 성능 최적화

### 캐싱 전략
```python
class AIResponseCache:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.default_ttl = 3600  # 1시간
    
    async def get_cached_response(self, prompt_hash):
        cached = await self.redis.get(f"ai_response:{prompt_hash}")
        if cached:
            return json.loads(cached)
        return None
    
    async def cache_response(self, prompt_hash, response, ttl=None):
        ttl = ttl or self.default_ttl
        await self.redis.setex(
            f"ai_response:{prompt_hash}",
            ttl,
            json.dumps(response)
        )
```

### 배치 처리
```python
class BatchProcessor:
    async def process_multiple_assessments(self, submissions):
        batch_size = 10
        results = []
        
        for i in range(0, len(submissions), batch_size):
            batch = submissions[i:i + batch_size]
            batch_tasks = [
                self.grade_single_submission(submission)
                for submission in batch
            ]
            batch_results = await asyncio.gather(*batch_tasks)
            results.extend(batch_results)
        
        return results
```

## 🔒 보안 및 데이터 보호

### API 키 관리
```python
class SecureAIClient:
    def __init__(self):
        self.openai_client = openai.AsyncOpenAI(
            api_key=self._get_secret("OPENAI_API_KEY")
        )
        self.azure_client = AzureOpenAI(
            endpoint=self._get_secret("AZURE_OPENAI_ENDPOINT"),
            api_key=self._get_secret("AZURE_OPENAI_KEY")
        )
    
    def _get_secret(self, secret_name):
        # Azure Key Vault에서 시크릿 가져오기
        return azure_keyvault_client.get_secret(secret_name).value
```

### 데이터 익명화
```python
class DataAnonymizer:
    def anonymize_student_data(self, submission):
        # 개인 식별 정보 제거
        anonymized = submission.copy()
        anonymized["student_id"] = self._hash_id(submission["student_id"])
        anonymized["student_name"] = "[익명]"
        return anonymized
```

이 AI 백엔드 설계는 교육 환경의 요구사항을 충족하면서 확장성과 안정성을 보장하도록 구성되었습니다. 