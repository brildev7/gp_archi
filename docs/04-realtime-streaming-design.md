# 실시간 스트리밍 시스템 설계

## 📋 개요

교육 플랫폼에서 수업 플랜과 수업 컨텐츠를 실시간으로 생성하고 스트리밍하는 시스템을 설계합니다. 이 시스템은 교사와 학생 간의 동적인 상호작용을 지원하고, AI가 실시간으로 교육 컨텐츠를 생성하여 즉시 전달하는 것을 목표로 합니다.

## 🎯 핵심 요구사항

### 실시간 요구사항
- **저지연**: 100ms 이하의 응답 지연시간
- **고처리량**: 동시 1000명 이상 사용자 지원
- **실시간 동기화**: 모든 참여자 간 실시간 상태 동기화
- **적응형 품질**: 네트워크 상태에 따른 자동 품질 조정

### 교육적 요구사항
- **즉석 컨텐츠 생성**: 수업 중 필요에 따른 즉시 자료 생성
- **개인화**: 학생별 맞춤형 컨텐츠 실시간 제공
- **상호작용**: 양방향 실시간 피드백 시스템
- **접근성**: 다양한 디바이스 및 네트워크 환경 지원

## 🏗️ 아키텍처 설계

### 전체 시스템 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    클라이언트 레이어                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   교사 웹앱   │  │  학생 모바일  │  │    관리자 대시보드   │   │
│  │  (React)     │  │    앱        │  │                  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                               │
                        WebSocket/WebRTC
                               │
┌─────────────────────────────────────────────────────────────┐
│                      실시간 게이트웨이                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │             Azure SignalR Service                      │  │
│  │        (Connection Management & Scaling)               │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────┐
│                   실시간 처리 레이어                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ 스트리밍 AI   │  │  세션 관리자  │  │   상태 동기화     │   │
│  │   서비스      │  │              │  │    서비스        │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                               │
┌─────────────────────────────────────────────────────────────┐
│                      데이터 레이어                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ Redis Stream │  │  Event Store │  │   벡터 캐시       │   │
│  │   (실시간)    │  │   (이력)     │  │   (컨텍스트)      │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 핵심 컴포넌트

#### 1. 실시간 연결 관리자
```typescript
interface ConnectionManager {
  // 연결 관리
  handleConnection(userId: string, sessionId: string): Promise<Connection>
  handleDisconnection(connectionId: string): Promise<void>
  
  // 메시지 라우팅
  broadcastToSession(sessionId: string, message: Message): Promise<void>
  sendToUser(userId: string, message: Message): Promise<void>
  
  // 그룹 관리
  joinGroup(connectionId: string, groupId: string): Promise<void>
  leaveGroup(connectionId: string, groupId: string): Promise<void>
}

class AzureSignalRConnectionManager implements ConnectionManager {
  private signalrClient: AzureSignalRClient;
  
  async handleConnection(userId: string, sessionId: string): Promise<Connection> {
    const connection = await this.signalrClient.createConnection({
      userId,
      sessionId,
      timestamp: Date.now()
    });
    
    // 세션 상태 복원
    await this.restoreSessionState(connection, sessionId);
    
    return connection;
  }
  
  async broadcastToSession(sessionId: string, message: Message): Promise<void> {
    await this.signalrClient.sendToGroup(sessionId, message);
  }
}
```

#### 2. 실시간 AI 스트리밍 서비스
```python
class StreamingAIService:
    def __init__(self):
        self.openai_client = AsyncOpenAI()
        self.redis_client = aioredis.Redis()
        self.session_manager = SessionManager()
    
    async def stream_lesson_content(
        self, 
        session_id: str, 
        prompt: str, 
        context: LessonContext
    ):
        """실시간 수업 컨텐츠 스트리밍"""
        
        # 1. 컨텍스트 준비
        enhanced_prompt = await self._build_contextual_prompt(
            prompt, context, session_id
        )
        
        # 2. 스트리밍 생성 시작
        stream = await self.openai_client.chat.completions.create(
            model="gpt-4-turbo",
            messages=enhanced_prompt,
            stream=True,
            temperature=0.7,
            max_tokens=1000
        )
        
        # 3. 실시간 전송
        buffer = ""
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                content = chunk.choices[0].delta.content
                buffer += content
                
                # 청크 단위로 즉시 전송
                await self._broadcast_chunk(session_id, {
                    "type": "content_chunk",
                    "data": content,
                    "timestamp": time.time(),
                    "chunk_id": str(uuid.uuid4())
                })
                
                # 문장 단위로 처리
                if self._is_sentence_boundary(buffer):
                    await self._process_complete_sentence(
                        session_id, buffer, context
                    )
                    buffer = ""
        
        # 4. 완료 처리
        await self._finalize_content_generation(session_id, buffer)

    async def adapt_content_realtime(
        self, 
        session_id: str, 
        feedback: StudentFeedback
    ):
        """학생 피드백 기반 실시간 컨텐츠 적응"""
        
        # 현재 컨텍스트 분석
        current_context = await self.session_manager.get_context(session_id)
        
        # 적응 전략 결정
        adaptation_strategy = await self._analyze_feedback(
            feedback, current_context
        )
        
        if adaptation_strategy.needs_immediate_action:
            # 즉시 보완 자료 생성
            supplementary_content = await self._generate_supplementary(
                adaptation_strategy, current_context
            )
            
            await self._broadcast_chunk(session_id, {
                "type": "supplementary_content",
                "data": supplementary_content,
                "trigger": "student_feedback",
                "adaptation_type": adaptation_strategy.type
            })
```

#### 3. 세션 상태 관리자
```python
class SessionStateManager:
    def __init__(self):
        self.redis = aioredis.Redis()
        self.event_store = EventStore()
    
    async def initialize_session(self, session_config: SessionConfig):
        """새 세션 초기화"""
        session_state = {
            "id": session_config.session_id,
            "teacher_id": session_config.teacher_id,
            "subject": session_config.subject,
            "grade_level": session_config.grade_level,
            "participants": [],
            "current_topic": session_config.initial_topic,
            "content_history": [],
            "interaction_history": [],
            "ai_context": session_config.ai_context,
            "created_at": datetime.utcnow().isoformat(),
            "status": "active"
        }
        
        # Redis에 세션 상태 저장 (TTL 설정)
        await self.redis.setex(
            f"session:{session_config.session_id}",
            86400,  # 24시간
            json.dumps(session_state)
        )
        
        # 이벤트 기록
        await self.event_store.record_event({
            "type": "session_initialized",
            "session_id": session_config.session_id,
            "data": session_state,
            "timestamp": datetime.utcnow()
        })
    
    async def update_session_context(
        self, 
        session_id: str, 
        context_update: dict
    ):
        """세션 컨텍스트 실시간 업데이트"""
        
        # 원자적 업데이트를 위한 Lua 스크립트
        lua_script = """
        local session_key = KEYS[1]
        local updates = cjson.decode(ARGV[1])
        local timestamp = ARGV[2]
        
        local session_data = redis.call('GET', session_key)
        if session_data then
            local session = cjson.decode(session_data)
            
            -- 업데이트 적용
            for key, value in pairs(updates) do
                session[key] = value
            end
            
            session.last_updated = timestamp
            
            -- 저장
            redis.call('SET', session_key, cjson.encode(session))
            return cjson.encode(session)
        else
            return nil
        end
        """
        
        updated_session = await self.redis.eval(
            lua_script,
            1,
            f"session:{session_id}",
            json.dumps(context_update),
            datetime.utcnow().isoformat()
        )
        
        return json.loads(updated_session) if updated_session else None
```

## 🚀 실시간 컨텐츠 생성 파이프라인

### 1. 즉석 레슨플랜 생성

```python
class RealtimeLessonPlanGenerator:
    async def generate_adaptive_plan(
        self, 
        base_topic: str, 
        student_responses: List[StudentResponse],
        time_remaining: int
    ):
        """학생 반응 기반 적응형 레슨플랜 생성"""
        
        # 1. 학생 이해도 실시간 분석
        comprehension_analysis = await self._analyze_comprehension(
            student_responses
        )
        
        # 2. 남은 시간 고려한 활동 계획
        time_optimized_activities = await self._optimize_for_time(
            base_topic, time_remaining, comprehension_analysis
        )
        
        # 3. 개별화 전략 적용
        individualized_plan = await self._apply_individualization(
            time_optimized_activities, student_responses
        )
        
        # 4. 실시간 스트리밍 시작
        async for plan_section in self._stream_plan_generation(
            individualized_plan
        ):
            yield {
                "type": "lesson_plan_section",
                "section": plan_section.type,
                "content": plan_section.content,
                "estimated_time": plan_section.duration,
                "difficulty_level": plan_section.difficulty,
                "materials_needed": plan_section.materials
            }

class RealtimeContentAdaptation:
    async def adapt_content_difficulty(
        self, 
        original_content: str,
        student_performance_data: dict,
        target_difficulty: str
    ):
        """실시간 난이도 조정"""
        
        adaptation_prompt = f"""
        원본 컨텐츠: {original_content}
        학생 성과 데이터: {student_performance_data}
        목표 난이도: {target_difficulty}
        
        다음 기준으로 컨텐츠를 실시간 조정해주세요:
        1. 어휘 수준 조정
        2. 설명의 상세도 조절
        3. 예시의 복잡도 변경
        4. 시각적 보조 자료 필요성 판단
        """
        
        stream = await openai.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[{"role": "user", "content": adaptation_prompt}],
            stream=True,
            temperature=0.5
        )
        
        adapted_content = ""
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                content_piece = chunk.choices[0].delta.content
                adapted_content += content_piece
                
                # 실시간 전송
                yield {
                    "type": "adapted_content",
                    "chunk": content_piece,
                    "adaptation_type": target_difficulty,
                    "timestamp": time.time()
                }
```

### 2. 실시간 시각 자료 생성

```python
class RealtimeVisualGenerator:
    def __init__(self):
        self.dalle_client = AsyncOpenAI()
        self.image_cache = ImageCache()
    
    async def generate_instant_visual(
        self, 
        description: str, 
        educational_context: dict,
        style_preferences: dict = None
    ):
        """교육용 시각 자료 즉시 생성"""
        
        # 캐시 확인
        cache_key = self._generate_cache_key(description, educational_context)
        cached_image = await self.image_cache.get(cache_key)
        
        if cached_image:
            return {
                "type": "cached_visual",
                "url": cached_image.url,
                "generation_time": 0
            }
        
        # 교육용 프롬프트 강화
        enhanced_prompt = self._enhance_educational_prompt(
            description, educational_context, style_preferences
        )
        
        # 실시간 생성 시작 알림
        yield {
            "type": "visual_generation_started",
            "description": description,
            "estimated_time": 15  # 초
        }
        
        try:
            response = await self.dalle_client.images.generate(
                model="dall-e-3",
                prompt=enhanced_prompt,
                size="1024x1024",
                quality="standard",
                n=1
            )
            
            image_url = response.data[0].url
            
            # 캐시에 저장
            await self.image_cache.store(cache_key, {
                "url": image_url,
                "prompt": enhanced_prompt,
                "context": educational_context,
                "created_at": datetime.utcnow()
            })
            
            yield {
                "type": "visual_generated",
                "url": image_url,
                "description": description,
                "educational_context": educational_context
            }
            
        except Exception as e:
            yield {
                "type": "visual_generation_failed",
                "error": str(e),
                "fallback_suggestions": await self._get_fallback_visuals(
                    description
                )
            }
```

## 📊 성능 최적화 전략

### 1. 다층 캐싱 시스템

```python
class MultiTierCache:
    def __init__(self):
        self.l1_cache = {}  # 메모리 캐시 (가장 빠름)
        self.l2_cache = aioredis.Redis()  # Redis 캐시
        self.l3_cache = AzureBlobStorage()  # 영구 저장소
    
    async def get_cached_content(self, cache_key: str):
        """계층적 캐시 조회"""
        
        # L1: 메모리 캐시
        if cache_key in self.l1_cache:
            return self.l1_cache[cache_key]
        
        # L2: Redis 캐시
        redis_result = await self.l2_cache.get(f"content:{cache_key}")
        if redis_result:
            content = json.loads(redis_result)
            # L1 캐시에 역참조
            self.l1_cache[cache_key] = content
            return content
        
        # L3: 영구 저장소
        blob_result = await self.l3_cache.get_blob(f"cache/{cache_key}")
        if blob_result:
            content = json.loads(blob_result)
            # 상위 캐시들에 역참조
            await self.l2_cache.setex(
                f"content:{cache_key}", 3600, json.dumps(content)
            )
            self.l1_cache[cache_key] = content
            return content
        
        return None
    
    async def store_content(
        self, 
        cache_key: str, 
        content: dict, 
        ttl: int = 3600
    ):
        """계층적 캐시 저장"""
        
        # 모든 레벨에 저장
        self.l1_cache[cache_key] = content
        
        await self.l2_cache.setex(
            f"content:{cache_key}", ttl, json.dumps(content)
        )
        
        await self.l3_cache.store_blob(
            f"cache/{cache_key}", json.dumps(content)
        )
```

### 2. 적응형 품질 관리

```typescript
class AdaptiveQualityManager {
  private networkMonitor: NetworkMonitor;
  private qualityLevels: QualityLevel[];
  
  constructor() {
    this.qualityLevels = [
      { name: 'high', videoRes: '1080p', audioKbps: 128, maxLatency: 100 },
      { name: 'medium', videoRes: '720p', audioKbps: 96, maxLatency: 200 },
      { name: 'low', videoRes: '480p', audioKbps: 64, maxLatency: 500 }
    ];
  }
  
  async adaptQuality(connectionId: string): Promise<QualityLevel> {
    const networkState = await this.networkMonitor.getConnectionState(connectionId);
    
    // 네트워크 상태 기반 품질 결정
    if (networkState.bandwidth > 5000 && networkState.latency < 100) {
      return this.qualityLevels.find(q => q.name === 'high')!;
    } else if (networkState.bandwidth > 2000 && networkState.latency < 300) {
      return this.qualityLevels.find(q => q.name === 'medium')!;
    } else {
      return this.qualityLevels.find(q => q.name === 'low')!;
    }
  }
  
  async monitorAndAdapt(connectionId: string): Promise<void> {
    setInterval(async () => {
      const optimalQuality = await this.adaptQuality(connectionId);
      await this.applyQualitySettings(connectionId, optimalQuality);
    }, 5000); // 5초마다 체크
  }
}
```

### 3. 병렬 처리 및 배치 최적화

```python
class ParallelContentProcessor:
    async def process_multiple_requests(
        self, 
        requests: List[ContentRequest]
    ):
        """여러 컨텐츠 요청 병렬 처리"""
        
        # 요청 유형별 그룹화
        text_requests = [r for r in requests if r.type == 'text']
        image_requests = [r for r in requests if r.type == 'image']
        audio_requests = [r for r in requests if r.type == 'audio']
        
        # 각 유형별 배치 처리
        text_task = asyncio.create_task(
            self._process_text_batch(text_requests)
        )
        image_task = asyncio.create_task(
            self._process_image_batch(image_requests)
        )
        audio_task = asyncio.create_task(
            self._process_audio_batch(audio_requests)
        )
        
        # 모든 작업 완료 대기
        text_results, image_results, audio_results = await asyncio.gather(
            text_task, image_task, audio_task
        )
        
        # 결과 통합 및 반환
        return self._combine_results(
            text_results, image_results, audio_results
        )
    
    async def _process_text_batch(self, requests: List[ContentRequest]):
        """텍스트 요청 배치 처리"""
        batch_size = 10
        results = []
        
        for i in range(0, len(requests), batch_size):
            batch = requests[i:i + batch_size]
            
            # OpenAI API 배치 호출
            batch_prompts = [req.prompt for req in batch]
            
            responses = await asyncio.gather(*[
                self._generate_single_text(prompt) 
                for prompt in batch_prompts
            ])
            
            results.extend(responses)
        
        return results
```

## 🔄 실시간 상호작용 패턴

### 1. 양방향 실시간 피드백

```typescript
interface RealtimeInteraction {
  // 교사 → 학생
  broadcastQuestion(question: Question): Promise<void>;
  sendIndividualHint(studentId: string, hint: string): Promise<void>;
  
  // 학생 → 교사
  submitAnswer(answer: Answer): Promise<void>;
  requestHelp(question: string): Promise<void>;
  
  // 양방향
  startPollSession(poll: Poll): Promise<PollSession>;
  updateCollaborativeBoard(update: BoardUpdate): Promise<void>;
}

class RealtimeInteractionHandler implements RealtimeInteraction {
  private signalr: SignalRConnection;
  private aiAssistant: AIAssistant;
  
  async broadcastQuestion(question: Question): Promise<void> {
    // 즉시 전체 전송
    await this.signalr.sendToGroup('students', {
      type: 'question_broadcast',
      question: question,
      timestamp: Date.now(),
      responseTime: question.timeLimit
    });
    
    // AI 답변 힌트 미리 준비
    const hints = await this.aiAssistant.prepareHints(question);
    await this.cacheHints(question.id, hints);
  }
  
  async submitAnswer(answer: Answer): Promise<void> {
    // 즉시 접수 확인
    await this.signalr.sendToUser(answer.studentId, {
      type: 'answer_received',
      answerId: answer.id,
      timestamp: Date.now()
    });
    
    // 실시간 AI 평가
    const evaluation = await this.aiAssistant.evaluateAnswer(answer);
    
    // 교사에게 즉시 결과 전송
    await this.signalr.sendToUser('teacher', {
      type: 'answer_evaluated',
      studentId: answer.studentId,
      evaluation: evaluation,
      suggestedFeedback: evaluation.feedback
    });
  }
}
```

### 2. 적응형 컨텐츠 딜리버리

```python
class AdaptiveContentDelivery:
    async def deliver_personalized_content(
        self, 
        session_id: str,
        base_content: str,
        student_profiles: List[StudentProfile]
    ):
        """학생별 맞춤형 컨텐츠 동시 전달"""
        
        # 학생 그룹별 컨텐츠 변형 생성
        content_variants = await self._generate_content_variants(
            base_content, student_profiles
        )
        
        # 각 학생에게 맞춤형 컨텐츠 스트리밍
        for student_id, variant in content_variants.items():
            asyncio.create_task(
                self._stream_personalized_content(
                    session_id, student_id, variant
                )
            )
    
    async def _stream_personalized_content(
        self, 
        session_id: str,
        student_id: str, 
        content_variant: ContentVariant
    ):
        """개별 학생 맞춤형 스트리밍"""
        
        # 학생 연결 상태 확인
        connection = await self.connection_manager.get_connection(student_id)
        if not connection or not connection.is_active:
            await self._queue_for_later_delivery(student_id, content_variant)
            return
        
        # 청크 단위로 스트리밍
        async for chunk in content_variant.stream():
            await connection.send({
                "type": "personalized_content",
                "chunk": chunk,
                "difficulty_level": content_variant.difficulty,
                "learning_style": content_variant.learning_style,
                "timestamp": time.time()
            })
            
            # 전송 간격 조정 (학생 처리 속도 고려)
            await self._adaptive_delay(student_id)
```

## 📈 확장성 및 고가용성

### 1. 수평적 확장 설계

```yaml
확장 전략:
  Connection Pool:
    - Azure SignalR Service 다중 인스턴스
    - 연결 분산 로드 밸런싱
    - 자동 스케일링 정책
  
  Processing Layer:
    - Container Instances 자동 확장
    - 큐 기반 작업 분산
    - 지역별 서비스 배포
  
  Data Layer:
    - Redis Cluster (샤딩)
    - Database Read Replicas
    - CDN 캐시 최적화

모니터링 지표:
  - 동시 연결 수
  - 메시지 처리 지연시간
  - AI 응답 생성 시간
  - 네트워크 대역폭 사용량
  - 오류율 및 재연결율
```

### 2. 장애 복구 메커니즘

```python
class FailoverManager:
    async def handle_connection_failure(self, connection_id: str):
        """연결 장애 처리"""
        
        # 1. 상태 백업
        session_state = await self.backup_session_state(connection_id)
        
        # 2. 대체 연결 시도
        alternative_endpoint = await self.find_alternative_endpoint()
        
        # 3. 상태 복원
        new_connection = await self.establish_connection(
            alternative_endpoint, session_state
        )
        
        # 4. 클라이언트 재연결 안내
        await self.notify_client_reconnection(connection_id, {
            "new_endpoint": alternative_endpoint,
            "session_token": new_connection.token,
            "restore_data": session_state
        })
    
    async def graceful_degradation(self, system_load: float):
        """점진적 성능 저하"""
        
        if system_load > 0.9:
            # 고급 AI 기능 비활성화
            await self.disable_advanced_ai_features()
            
        if system_load > 0.8:
            # 실시간 시각 자료 생성 제한
            await self.limit_visual_generation()
            
        if system_load > 0.7:
            # 스트리밍 품질 자동 조정
            await self.reduce_streaming_quality()
```

이 실시간 스트리밍 시스템은 교육 환경의 특수한 요구사항을 고려하여 설계되었으며, 높은 확장성과 안정성을 제공하도록 구성되었습니다. 