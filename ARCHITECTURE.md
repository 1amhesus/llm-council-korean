# ARCHITECTURE.md - Technical Notes for LLM Council-korean

이 파일은 향후 개발 세션을 위한 기술적 상세, 아키텍처 결정 사항, 그리고 중요한 구현 노트를 포함.

## Project Overview

LLM Council-korean은 여러 LLM들이 협력적으로 사용자 질문에 답하는 3단계 심의(3-stage deliberation) 시스템이고, 혁신적인 것은 2단계(Stage 2)에서 진행되는 익명화된 피어 리뷰(anonymized peer review)를 통해 모델들이 편향을 갖지 못하도록 함.

## Architecture

### Backend Structure (`backend/`)

백엔드는 Python FastAPI 오케스트레이션 레이어를 유지하며, CPU-집약적 ranking/aggregation 로직은 Rust 엔진으로 이전되었음.

전체 데이터 파이프라인은 다음 세 계층으로 구성:

FastAPI → synedrion-py(PyO3) → synedrion-core(Rust logic)

**`config.py`**
- `COUNCIL_MODELS` (OpenRouter model identifier 목록)을 포함
- `CHAIRMAN_MODEL` (최종 답변을 합성-synthesizes final answer-하는 모델)을 포함
- `.env`에 있는 환경 변수 `OPENROUTER_API_KEY`을 사용
- 백엔드는 **port 8001**에서 실행

이 구성은 기존 Karpathy의 바이브 코드에서 변경된것 없음 — Synedrion Rust pipeline은 config 레이어에 영향을 주지 않음.

**`openrouter.py`**
- `query_model()`: 단일(Single) async model query
- `query_models_parallel()`: `asyncio.gather()`로 병렬 질의(parallel query)
- 호출 결과: {content, reasoning_details?}(Returns dict with 'content' and optional 'reasoning_details')
- 실패 모델은 무시하고 나머지 응답 진행(Graceful degradation: returns None on failure, continues with successful responses)

Rust 이전과 무관한 외부 네트워크 계층 → Python 영역 유지가 적절함(LLM 호출은 Rust 성능 이점이 거의 없는 I/O bound 영역)

**`council.py`** - 핵심 로직(The Core Logic)

- `stage1_collect_responses()`: Parallel queries to all council models
- `stage2_collect_rankings()`:
  - Anonymizes responses as "Response A, B, C, etc."
  - Creates `label_to_model` mapping for de-anonymization
  - Prompts models to evaluate and rank (with strict format requirements)
  - Returns tuple: (rankings_list, label_to_model_dict)
  - Each ranking includes both raw text and `parsed_ranking` list
- `stage3_synthesize_final()`: Chairman synthesizes from all responses + rankings
- `parse_ranking_from_text()`: Extracts "FINAL RANKING:" section, handles both numbered lists and plain format
- `calculate_aggregate_rankings()`: Computes average rank position across all peer evaluations


**council/ 핵심 모듈 구조 변화**

이 영역이 러스트 기반 Synedrion 중심 구조 변화 반영 핵심

이전 구조: 모든 ranking logic이 council.py 내부 Python 함수에 직접 구현됨

변경 후:Python은 orchestration만 수행하고 모든 CPU ranking/aggregation 연산은 Rust로 위임됨

**council.py (Python Orchestration Layer) - 핵심 로직(The Core Logic)**

- 유지되는 Stage 로직: stage1_collect_responses() - 모든 council 모델에 대해 병렬 응답 수집,
                       stage2_collect_rankings()  - Response 라벨링 (A/B/C…),
                                                  - label_to_model 매핑 생성,
                                                  - 모델에게 peer 평가 요청
                                                  - 결과: stage2_results + label_to_model
                       stage3_synthesize_final()  - 의장(Chairman)이 Stage1 + Stage2 자료 기반으로 최종 응답 합성

- 변경되는 내부 역할: 다음 두 함수가 제거됨 - parse_ranking_from_text(), calculate_aggregate_rankings()
    → 이제 Python이 아니라 Rust에 있음.

- 대신 Python쪽에서는: from synedrion_py import parse_ranking_from_text_rs, calculate_aggregate_rankings_rs
                       을 통해 JSON 문자열을 전달하고 결과를 받아오는 lightweight wrapper 역할 수행.

**New File: council/ranking.py 새로운 파일**

모든 ranking CPU 로직이 Python에서 제거됨

책임: - Rust binding 호출
      - JSON 직렬화/역직렬화
      - FastAPI 레이어에 Rust 결과 전달

내용 요약: -ranking.py
            ├─ parse_ranking_from_text(text) → Rust 호출
            └─ calculate_aggregate_rankings(stage2_results, label_to_model) → Rust 호출

**Rust Core Engine (rust/ workspace)**

- synedrion-core/ : - 완전한 business logic 보유,
                    - Python/HTTP 개념 없음
                    - 모듈 구조: - parser.rs → ranking 문자열 파싱
                                 - aggregator.rs → ranking 평균/정렬 계산
                                 - types.rs → Stage2Result, AggregateRanking struct

                    - serde/regex 사용

- synedrion-py/ : - PyO3 binding crate
                  - Rust 로직을 Python에서 사용할 수 있도록 노출
                  - maturin develop 후 Python import 가능


**`storage.py`**
- JSON-based conversation storage in `data/conversations/`
- Each conversation: `{id, created_at, messages[]}`
- Assistant messages contain: `{role, stage1, stage2, stage3}`
- Note: metadata (label_to_model, aggregate_rankings) is NOT persisted to storage, only returned via API

- JSON conversation persistence는 유지되지만 Ranking/aggregation 메타데이터는 저장하지 않음
- 저장 구조: data/conversations/{conversation_id}.json

Synedrion Rust pipeline은 저장 계층을 수정하지 않음.

**`main.py`**
- CORS 설정, routing 방식(FastAPI app with CORS enabled for localhost:5173 and localhost:3000) 모두 이전과 동일.
- POST `/api/conversations/{id}/message` stage outputs + Rust aggregate ranking metadata 리턴
- 메타 데이터 포함(Metadata includes): label_to_model mapping and aggregate_rankings

### Frontend Structure (`frontend/src/`)

**`App.jsx`**
- Main orchestration: manages conversations list and current conversation
- Handles message sending and metadata storage
- Important: metadata is stored in the UI state for display but not persisted to backend JSON

**`components/ChatInterface.jsx`**
- Multiline textarea (3 rows, resizable)
- Enter to send, Shift+Enter for new line
- User messages wrapped in markdown-content class for padding

**`components/Stage1.jsx`**
- Tab view of individual model responses
- ReactMarkdown rendering with markdown-content wrapper

**`components/Stage2.jsx`**
- **Critical Feature**: Tab view showing RAW evaluation text from each model
- De-anonymization happens CLIENT-SIDE for display (models receive anonymous labels)
- Shows "Extracted Ranking" below each evaluation so users can validate parsing
- Aggregate rankings shown with average position and vote count
- Explanatory text clarifies that boldface model names are for readability only

**`components/Stage3.jsx`**
- Final synthesized answer from chairman
- Green-tinted background (#f0fff0) to highlight conclusion

**Styling (`*.css`)**
- Light mode theme (not dark mode)
- Primary color: #4a90e2 (blue)
- Global markdown styling in `index.css` with `.markdown-content` class
- 12px padding on all markdown content to prevent cluttered appearance

## Key Design Decisions

### Stage 2 Prompt Format
The Stage 2 prompt is very specific to ensure parseable output:
```
1. Evaluate each response individually first
2. Provide "FINAL RANKING:" header
3. Numbered list format: "1. Response C", "2. Response A", etc.
4. No additional text after ranking section
```

This strict format allows reliable parsing while still getting thoughtful evaluations.

### De-anonymization Strategy
- Models receive: "Response A", "Response B", etc.
- Backend creates mapping: `{"Response A": "openai/gpt-5.1", ...}`
- Frontend displays model names in **bold** for readability
- Users see explanation that original evaluation used anonymous labels
- This prevents bias while maintaining transparency

### Error Handling Philosophy
- Continue with successful responses if some models fail (graceful degradation)
- Never fail the entire request due to single model failure
- Log errors but don't expose to user unless all models fail

### UI/UX Transparency
- All raw outputs are inspectable via tabs
- Parsed rankings shown below raw text for validation
- Users can verify system's interpretation of model outputs
- This builds trust and allows debugging of edge cases

## Important Implementation Details

### Relative Imports
All backend modules use relative imports (e.g., `from .config import ...`) not absolute imports. This is critical for Python's module system to work correctly when running as `python -m backend.main`.

### Port Configuration
- Backend: 8001 (changed from 8000 to avoid conflict)
- Frontend: 5173 (Vite default)
- Update both `backend/main.py` and `frontend/src/api.js` if changing

### Markdown Rendering
All ReactMarkdown components must be wrapped in `<div className="markdown-content">` for proper spacing. This class is defined globally in `index.css`.

### Model Configuration
Models are hardcoded in `backend/config.py`. Chairman can be same or different from council members. The current default is Gemini as chairman per user preference.

## Common Gotchas

1. **Module Import Errors**: Always run backend as `python -m backend.main` from project root, not from backend directory
2. **CORS Issues**: Frontend must match allowed origins in `main.py` CORS middleware
3. **Ranking Parse Failures**: If models don't follow format, fallback regex extracts any "Response X" patterns in order
4. **Missing Metadata**: Metadata is ephemeral (not persisted), only available in API responses

## Future Enhancement Ideas

- Configurable council/chairman via UI instead of config file
- Streaming responses instead of batch loading
- Export conversations to markdown/PDF
- Model performance analytics over time
- Custom ranking criteria (not just accuracy/insight)
- Support for reasoning models (o1, etc.) with special handling

## Testing Notes

Use `test_openrouter.py` to verify API connectivity and test different model identifiers before adding to council. The script tests both streaming and non-streaming modes.

## Data Flow Summary

```
User Query
    ↓
Stage 1: Parallel queries → [individual responses]
    ↓
Stage 2: Anonymize → Parallel ranking queries → [evaluations + parsed rankings]
    ↓
Aggregate Rankings Calculation → [sorted by avg position]
    ↓
Stage 3: Chairman synthesis with full context
    ↓
Return: {stage1, stage2, stage3, metadata}
    ↓
Frontend: Display with tabs + validation UI
```

The entire flow is async/parallel where possible to minimize latency.
