# CLAIR AI 테스트 가이드

clair-ai 서비스의 RAG Q&A 및 법령 준수 검사 기능 테스트 방법 안내.

---

## 사전 준비

### 1. 환경변수 설정

`clair-ai/.env` 파일에 아래 내용 설정:

```env
GEMINI_API_KEY=발급받은_키_입력
```

> Gemini API 키 발급: https://aistudio.google.com/app/apikey

### 2. 의존성 설치

```bash
cd clair-ai
pip install -e .
pip install chromadb langchain-chroma
```

### 3. 서버 실행

```bash
cd clair-ai
uvicorn src.api.main:app --port 8001
```

서버 시작 로그에 아래가 출력되면 법령 DB 초기화 성공:

```
[clair-ai] 법령 DB 초기화 완료: 21개 조항
```

---

## 테스트 1 — 헬스체크

```bash
curl http://localhost:8001/health
```

**기대 응답:**
```json
{"status": "ok", "service": "clair-ai"}
```

---

## 테스트 2 — 인덱싱된 법령 목록 확인

```bash
curl http://localhost:8001/compliance/laws
```

**기대 응답 (일부):**
```json
{
  "laws": {
    "근로기준법": ["제17조 근로조건의 명시", "제56조 연장·야간 및 휴일 근로", ...],
    "민법": ["제103조 반사회질서의 법률행위", ...],
    ...
  },
  "total_articles": 21
}
```

---

## 테스트 3 — 법령 준수 검사 (핵심 기능)

위반 조항과 적합 조항을 섞어서 테스트.

```bash
curl -X POST http://localhost:8001/compliance \
  -H "Content-Type: application/json" \
  -d '{
    "contract_id": 1,
    "contract_type": "근로계약",
    "clauses": [
      {
        "clause_id": "clause-001",
        "title": "연장근로",
        "text": "연장근로에 대한 별도 수당은 지급하지 아니한다."
      },
      {
        "clause_id": "clause-002",
        "title": "계약기간",
        "text": "본 계약의 기간은 2025년 1월 1일부터 2025년 12월 31일까지로 한다."
      },
      {
        "clause_id": "clause-003",
        "title": "근로시간",
        "text": "1일 근로시간은 12시간으로 하며, 휴게시간 없이 연속 근무한다."
      }
    ]
  }'
```

**기대 응답:**
```json
{
  "contract_id": 1,
  "contract_type": "근로계약",
  "total_clauses_checked": 3,
  "violation_count": 2,
  "warning_count": 0,
  "results": [
    {
      "clause_id": "clause-001",
      "status": "위반",
      "reason": "근로기준법 제56조가 정하는 연장근로 가산수당 지급 의무를 명백히 위반합니다...",
      "law_references": [
        {"law_name": "근로기준법", "article_no": "제56조", ...}
      ]
    },
    {
      "clause_id": "clause-002",
      "status": "적합",
      "reason": "1년의 고정된 계약 기간 명시는 근로계약에서 일반적으로 허용됩니다...",
      ...
    },
    {
      "clause_id": "clause-003",
      "status": "위반",
      "reason": "근로기준법 제50조 위반...",
      ...
    }
  ]
}
```

### 계약 유형별 테스트

`contract_type` 변경으로 다른 법령 적용:

| contract_type | 적용 법령 |
|---|---|
| `근로계약` | 근로기준법, 최저임금법 |
| `용역계약` | 민법, 하도급법 |
| `NDA` | 부정경쟁방지법 |
| `임대차계약` | 주택임대차보호법, 민법 |

---

## 테스트 4 — RAG Q&A

계약서 조항 기반 질의응답. 벡터 검색으로 관련 조항 찾아 답변.

```bash
curl -X POST http://localhost:8001/qa \
  -H "Content-Type: application/json" \
  -d '{
    "contract_id": 1,
    "question": "연장근로 수당은 얼마나 받을 수 있나요?",
    "clauses": [
      {
        "clause_id": "clause-001",
        "title": "연장근로",
        "text": "연장근로에 대한 별도 수당은 지급하지 아니한다."
      },
      {
        "clause_id": "clause-002",
        "title": "급여",
        "text": "월 급여는 300만원으로 하며, 매월 25일에 지급한다."
      },
      {
        "clause_id": "clause-003",
        "title": "근로시간",
        "text": "1일 8시간, 주 40시간을 기준 근로시간으로 한다."
      }
    ],
    "use_rag": true
  }'
```

**기대 응답:**
```json
{
  "question": "연장근로 수당은 얼마나 받을 수 있나요?",
  "answer": "본 계약서에는 연장근로 수당을 지급하지 않는다고 명시되어 있습니다...",
  "evidence_clause_ids": ["clause-001", "clause-002"]
}
```

> `use_rag: false` 로 변경하면 전체 조항을 LLM에 직접 전달하는 기존 방식 사용

---

## 테스트 5 — 계약서 파일 전체 분석

실제 파일을 업로드하여 OCR → 분석 → 법령 준수 검사까지 전체 파이프라인 테스트.

```bash
curl -X POST http://localhost:8001/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "contract_id": 1,
    "file_path": "/절대경로/계약서.txt",
    "file_type": "txt",
    "document_id": "test-doc-001"
  }'
```

샘플 파일 위치: `team-docs/samples/`
- `sample_service_contract.txt` — 용역계약서
- `sample_service_contract.pdf` — 용역계약서 (PDF)
- `sample_nda.txt` — NDA
- `sample_employment_contract.txt` — 근로계약서

**응답에 포함되는 항목:**
```json
{
  "document_id": "test-doc-001",
  "clauses": [...],
  "extraction": { "contract_type": {...}, "signing_date": {...}, ... },
  "risks": [...],
  "summary": "계약서 요약...",
  "compliance": [
    {"clause_id": "...", "status": "위반", "reason": "...", "law_references": [...]}
  ],
  "ocr_raw_text": "..."
}
```

---

## Python 직접 테스트 (서버 없이)

서버 없이 모듈 단위로 테스트할 때:

```bash
cd clair-ai
```

### 임베딩 테스트
```python
from src.rag.embedder import embed_query, embed_texts

v = embed_query("계약 기간")
print("벡터 차원:", len(v))  # 3072
```

### 법령 검색 테스트
```python
from src.legal.law_store import get_law_store, ensure_law_db_initialized

ensure_law_db_initialized()  # 최초 1회 초기화 필요
store = get_law_store()
results = store.search("연장근로 수당 지급", top_k=3)
for r in results:
    print(f"[{r.score:.3f}] {r.law_name} {r.article_no} {r.article_title}")
```

### 법령 준수 검사 테스트
```python
from src.chains.contract_analysis_chain import Clause
from src.legal.compliance_chain import get_compliance_chain

clauses = [
    Clause(
        clause_id="clause-001",
        title="연장근로",
        text="연장근로에 대한 별도 수당은 지급하지 아니한다.",
        page_refs=[],
        order=1,
    )
]

chain = get_compliance_chain()
results = chain.check_clauses(clauses, contract_type="근로계약")
for r in results:
    print(f"[{r.status}] {r.clause_title}: {r.reason}")
```

---

## 트러블슈팅

| 오류 | 원인 | 해결 |
|------|------|------|
| `GEMINI_API_KEY가 설정되지 않았습니다` | .env 파일 미설정 또는 경로 오류 | `clair-ai/.env` 에 키 입력 확인 |
| `법령 DB 초기화 건너뜀` | API 키 미인식 | 서버 재시작 후 로그 확인 |
| `404 models/gemini-2.0-flash is no longer available` | 구버전 모델 | `gemini.py` 에서 `gemini-2.5-flash` 로 변경 |
| `404 text-embedding-004 is not found` | 임베딩 모델 미지원 | `embedder.py` 에서 `gemini-embedding-001` 로 변경 |
| 법령 검사 결과 0개 | 유사도 임계값(0.4) 미달 | 조항 텍스트가 너무 짧거나 법령과 무관한 내용 |
