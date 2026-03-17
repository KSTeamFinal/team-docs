# 계약서 분석 AI 프로젝트 구조 문서

## 1. 문서 목적

이 문서는 계약서 분석 AI 시스템 개발을 위한 권장 저장소 구조와 디렉터리 책임을 정의한다. 기준 아키텍처는 `React + PWA + FastAPI + MySQL + Gemini API + LangChain + PaddleOCR + YOLOv8`이다.

## 2. 저장소 운영 방식

권장 방식은 모노레포다.

```text
contract-ai/
  frontend/
  backend/
  ml/
  infra/
  docs/
  .github/
```

## 3. 루트 디렉터리 구조

```text
contract-ai/
  frontend/
  backend/
  ml/
  infra/
  docs/
  .github/
  .editorconfig
  .gitignore
  README.md
```

### 디렉터리 책임

- `frontend/`: React PWA 사용자 화면
- `backend/`: FastAPI API 및 분석 오케스트레이션
- `ml/`: OCR, YOLO, 데이터셋, 실험 설정
- `infra/`: Docker, DB 초기화, 배포 설정
- `docs/`: 기획, 프롬프트, 테스트케이스 문서
- `.github/`: CI 및 협업 템플릿

## 4. 프론트엔드 구조

```text
frontend/
  src/
    app/
      upload/
      documents/
      documents/[documentId]/
      layout.tsx
      page.tsx
    components/
      ui/
      document/
      detection/
      qa/
    features/
      upload/
      document-list/
      document-detail/
      qa/
    lib/
      api/
      utils/
      constants/
    hooks/
    styles/
  public/
  tests/
  package.json
```

## 5. 백엔드 구조

```text
backend/
  app/
    api/
      v1/
        documents.py
        qa.py
        detections.py
    core/
      config.py
      logging.py
    schemas/
      document.py
      extraction.py
      qa.py
      risk.py
      detection.py
    services/
      document_service.py
      analysis_service.py
      qa_service.py
    parsers/
      clause_parser.py
      field_extractor.py
      ocr_normalizer.py
    rules/
      risk_detector.py
      risk_rules.yaml
    integrations/
      gemini_client.py
      mysql_client.py
      paddleocr_client.py
      yolo_client.py
      langchain_pipeline.py
    workers/
      analysis_worker.py
    tests/
      test_clause_parser.py
      test_field_extractor.py
      test_risk_detector.py
      test_documents_api.py
    main.py
```

## 6. ML 구조

```text
ml/
  datasets/
    roboflow/
  models/
    yolo/
  notebooks/
  training/
    lightning/
    configs/
```

### 책임

- `datasets/roboflow/`: 라벨링 데이터셋 메타정보
- `models/yolo/`: 학습된 가중치 관리
- `training/lightning/`: 학습 코드
- `training/configs/`: 학습 설정 파일

## 7. 데이터 구조

MySQL은 아래 테이블 중심으로 구성한다.

```text
documents
clauses
extractions
risk_flags
qa_logs
vision_detections
analysis_jobs
```

## 8. 인프라 구조

```text
infra/
  docker/
    backend.Dockerfile
  db/
    init.sql
  env/
    frontend.env.example
    backend.env.example
  ml/
    training-config.yaml
```

## 9. 개발 시작 체크리스트

1. React 프로젝트 생성 및 PWA 설정
2. FastAPI 기본 서버 구성
3. MySQL 초기 스키마 작성
4. Gemini API 키 연결
5. PaddleOCR, YOLOv8 실행 환경 구성
6. Roboflow 데이터셋 준비
7. 샘플 계약서 3종 기준 업로드/분석 흐름 고정
