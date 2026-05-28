# Celery + Redis 마이그레이션 플랜

> 현재 브랜치 `feat/celery-redis`에 구현 완료 상태로 보존돼 있음.
> 지금 당장은 asyncio 인메모리 방식으로 운영하고, 트래픽이 늘어 uvicorn 워커를 여러 개 띄워야 할 때 이 플랜을 적용한다.

---

## 왜 필요한가

| 현재 (asyncio 인메모리) | Celery + Redis 전환 후 |
|---|---|
| uvicorn 워커 1개에서만 동작 | 워커 N개 수평 확장 가능 |
| 서버 재시작 시 처리 중 태스크 유실 | 태스크 큐가 Redis에 영속 |
| 재시도 직접 구현 | 자동 재시도 (max_retries, countdown) |
| 모니터링 없음 | Flower UI로 태스크 현황 실시간 확인 |

---

## 아키텍처

```
[Client]
   │
   ├─ POST /contracts/{id}/analyze
   │       └─ analyze_contract_task.delay(contract_id)  →  [Redis 브로커]
   │                                                              │
   │                                                    [Celery Worker]
   │                                                       asyncio.run(analyze_contract_background())
   │
   ├─ POST /chat/sessions/{id}/messages  (202)
   │       └─ process_chat_message_task.delay(...)  →  [Redis 브로커]
   │                                                          │
   │                                               [Celery Worker]
   │                                                  clair-ai /qa 호출
   │                                                  → redis.publish(f"chat:session:{id}")
   │
   └─ GET /chat/sessions/{id}/stream  (SSE)
           └─ redis.pubsub().subscribe(f"chat:session:{id}")
                   └─ AI 응답 도착 시 SSE 이벤트로 전달
```

---

## 실행 명령어 (전환 시)

```bash
# 1. Redis 실행
brew services start redis

# 2. 패키지 설치 (이미 requirements.txt에 추가돼 있음)
cd clair-backend
pip install -r requirements.txt

# 3. Celery 워커 시작 (터미널 별도 탭)
celery -A app.celery_app worker --loglevel=info

# 4. (선택) Flower 모니터링 UI — http://localhost:5555
celery -A app.celery_app flower

# 5. 백엔드 서버
uvicorn app.main:app --reload --port 8000
```

`.env`에 Redis URL이 기본값과 다르면 추가:
```
REDIS_URL=redis://127.0.0.1:6379/0
```

---

## 전환 체크리스트

- [ ] `feat/celery-redis` 브랜치를 `dev`에 머지
- [ ] 서버에 Redis 설치 및 실행 확인 (`redis-cli ping`)
- [ ] Celery 워커 프로세스 관리 방법 결정 (systemd / supervisor / PM2)
- [ ] 환경변수 `REDIS_URL` 설정
- [ ] Flower 대시보드 접근 권한 설정 (외부 노출 시 인증 추가)
- [ ] uvicorn 워커 수 조정: `uvicorn app.main:app --workers 4 --port 8000`

---

## 주의사항

- **SSE 인증**: `EventSource`는 커스텀 헤더를 지원하지 않아 JWT를 쿼리 파라미터(`?token=...`)로 전달함. HTTPS 환경에서만 운용할 것.
- **asyncio + Celery**: Celery 태스크 내부에서 `asyncio.run()`을 사용해 기존 async 함수를 재사용함. Celery 워커는 기본적으로 sync이므로 이벤트 루프를 직접 생성해야 함.
- **인메모리 → Redis 전환 시 채팅 SSE**: 현재 코드(`tasks/chat.py`)가 이미 Redis Pub/Sub으로 구현돼 있어 코드 수정 없이 브랜치 머지만으로 전환됨.

---

## 관련 파일 (feat/celery-redis 브랜치)

| 파일 | 역할 |
|---|---|
| `clair-backend/app/celery_app.py` | Celery 인스턴스 설정 |
| `clair-backend/app/tasks/analysis.py` | 계약서 분석 태스크 (재시도 2회) |
| `clair-backend/app/tasks/chat.py` | 채팅 Q&A 태스크 + Redis publish |
| `clair-backend/app/api/v1/chat.py` | POST 202 + GET /stream SSE 엔드포인트 |
| `clair-backend/app/api/v1/contracts.py` | BackgroundTask → Celery 교체 |
