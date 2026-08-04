# AX Hub — Creative Memory & Employee KB Design

**Date:** 2026-08-04  
**Status:** Draft (aligned with (주)원정대 AX 채용 방향 + 사전 과제 케이스 분석)  
**Scope:** 입사 전~초기 AX 프로토타입 기준 스펙. 완성 시스템 전체가 아님.

---

## 1. Context

### 1.1 Company

(주)원정대는 숏폼 커머스 콘텐츠·퍼포먼스 마케팅 중심 기업이다.

- 뷰티·리빙·가전·패션 등 브랜드 크리에이티브 기획·촬영
- 성공 지표는 조회수가 아니라 **전환·매출**
- 네이버·카카오 공식 파트너, 커머스 광고·숏폼 운영 규모가 빠르게 성장

### 1.2 AX Team Goal

AX = AI Transformation. 목표는 자동화 도구 N개가 아니라:

> 흩어진 **데이터 · 문서 · 기준 · 처리 이력**을 연결한 **사내 AI 통합시스템 (AX Hub)**

채용 공고가 명시한 예시:

- 광고 소재의 **반려 사유 + 성과**를 참고해 검수·생성 흐름 개선
- 반복 보고/정리 자동화
- 사내 데이터를 AI가 활용할 수 있는 구조로 정리
- Knowledge Base / RAG / Knowledge Graph / AI Agent 실험
- 작은 프로토타입 → 현업 피드백 → 개선

### 1.3 Case Signal (사전 과제)

과제 `ax_pre_interview_case_exercise`는 제출용이 아니라, AX Hub 첫 도메인의 힌트다.

실무자 요청 예:

1. “올해 5~6월 가상브랜드A 관련, 다시 쓸 만한 소재”
2. “가상브랜드C, 얼굴 비교 말고 사용 장면 위주”
3. “성과 좋았던 소재 3개”
4. “직장인 아침 루틴 느낌, 다른 상품에도 참고할 만한 것”

관찰된 병목: 폴더/태그 검색 → 캠페인·심사·성과를 사람이 따로 합침 (중앙값 17분, 재확인·재전달 발생).

---

## 2. Problem Statement

과거 광고 소재·사내 지식을 찾을 때:

- 소재 파일, 태그, 캠페인, 심사, 성과가 **분산**되어 있다
- 태그가 비표준(`diet`/`다이어트`, 공백, `FINAL2`)이라 검색이 깨진다
- “다시 쓸 만함” ≠ “성과 좋음” (매체별 반려·승인 이력이 재사용 가능성에 직결)
- 성과 비교 시 **전환기준**(LPV vs purchase)·**데이터상태**(확정 vs 잠정)를 무시하면 오판한다
- 회사 소개형 RAG만으로는 채용/과제에서 가리킨 병목을 풀지 못한다

---

## 3. Goals & Non-Goals

### 3.1 Goals

1. **Creative Memory**: 소재 × 캠페인 × 심사 × 성과를 join한 재사용 후보 조회
2. **Employee KB**: 사내 매뉴얼·기준·러닝 문서를 Obsidian에 두고 RAG로 질의
3. 현업이 Hermes로 자연어 질의할 수 있는 최소 Agent 툴셋
4. 스키마·태그 사전·사실/추론 분리 규칙을 팀 자산으로 문서화

### 3.2 Non-Goals (초기)

- DeepTutor 등 교육용 제품 통째 fork/도입
- 전체 매체 API 실시간 연동
- Knowledge Graph 풀 구축
- 완벽한 멀티 Agent / 자동 소재 생성 파이프라인
- 월 300소재 스케일 인프라 최적화
- 회사 소개·복지 중심 RAG를 코어로 삼는 것

---

## 4. Architecture Overview

AX Hub는 두 지식 레이어 + 하나의 Agent UX로 구성한다.

```
┌─────────────────────────────────────────────┐
│  Hermes (직원 접점 UX / Agent / Tools·MCP)   │
└─────────────────┬───────────────────────────┘
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
┌───────────────┐     ┌──────────────────────┐
│ Employee KB   │     │ Creative Memory      │
│ (문서 RAG)    │     │ (정형 + 멀티모달)    │
│               │     │                      │
│ Obsidian md   │     │ Asset/Campaign/      │
│      ↓        │     │ Review/Performance/  │
│ LlamaIndex    │     │ Learning             │
│      ↓        │     │      ↓               │
│ Postgres      │     │ Postgres (SQL join)  │
│ + pgvector    │     │ (+ 선택적 임베딩)    │
└───────────────┘     └──────────────────────┘
```

### 4.1 Role Separation

| Layer | Role |
|-------|------|
| **Obsidian** | 사람이 쓰는 원본 문서 저장소 (md vault) |
| **LlamaIndex** | 문서 청킹·임베딩·인덱싱·검색 |
| **Postgres + pgvector** | 벡터 저장 + 정형 메타/업무 데이터 |
| **Hermes** | 질의 UX, 툴 호출, 메모리/스킬 (DeepTutor UX 대체) |

Obsidian vs LlamaIndex는 경쟁이 아니라 보완이다.  
저장소 = Obsidian, 인덱싱/벡터 = LlamaIndex + pgvector.

### 4.2 Why This Stack

- DeepTutor: RAG 엔진 아이디어만 참고. 본체·교육 UX 비채택
- LangChain 구형 Chain: 비채택. (Agent는 Hermes가 담당)
- LlamaIndex: 2026 기준 문서 RAG에 현역. 전사 KB에 적합
- 벡터DB: 초기 **pgvector** (회사 실무 기본값에 가깝고, 소재 메타 join에 유리). 규모·필터 한계 시 Qdrant 검토
- Qdrant/Pinecone/Weaviate/Milvus는 대안이지 필수 아님

---

## 5. Data Model — Creative Memory

연결 키: `소재번호`(asset_id), `캠페인번호`(campaign_id).  
케이스 CSV와 호환되는 최소 스키마.

### 5.1 Entities

#### Asset (소재)

- asset_id, file_name, file_type (`image`|`video`), storage_path
- registered_at, preview_path
- legacy_tags (원본 비정제, 참고용)
- ops_memo
- controlled tags: format, angle, hook, audience (아래 사전)

#### Campaign (캠페인)

- campaign_id, name, brand, category, target_audience
- objective, conversion_type (`landing_page_view`|`purchase`|…)
- platform, start_date, end_date

#### Review (심사 이력)

- review_id, asset_id, campaign_id, platform
- status (`approved`|`rejected`|`revision_required`)
- reject_reason, remediation, reviewed_at
- **현재 상태 컬럼은 별도 없음** → 동일 키의 최신 이벤트로 해석

#### Performance (성과)

- perf_id, asset_id, campaign_id
- impressions, clicks, ctr, conversions, spend
- period_start, period_end
- data_status (`확정`|`잠정`)
- 행 없음 ≠ 미집행/0 (표본·누락 가능)

#### Learning (러닝)

- asset_id 또는 campaign_id (optional FK)
- hypothesis, result, next_test, notes
- created_at
- RAG 문서화 시 Obsidian `learnings/`에도 mirror 가능

### 5.2 Controlled Tag Taxonomy

자유 태그 금지. 소수 통제 값만 사용.

| Field | Example values |
|-------|----------------|
| format | `image`, `video_9x16`, `video_1x1`, `carousel` |
| angle | `product_cut`, `usage_scene`, `before_after`, `ugc`, `lookbook`, `promo` |
| hook | `morning_routine`, `social_proof`, `offer`, `urgency`, `feature` |
| audience | `20s_female`, `office_30s`, `1person_household`, … |
| reuse_status | 매체별 최신 심사에서 유도: `approved` / `rejected` / `conditional` |
| perf_tier | `champion` / `average` / `weak` / `unknown` (동일 전환기준·확정만 비교) |

태그 사전 문서를 Obsidian에 두고 신규 값은 사전 합의 후에만 추가.

### 5.3 Fact / Inference / Hypothesis

| Layer | Example | Storage |
|-------|---------|---------|
| Fact | CTR 3.8%, Meta approved | DB numeric/status |
| Inference | “아침 루틴이 먹힌다” | Learning + `inferred` 표시 |
| Hypothesis | “다른 상품에도 전이” | Learning only; 랭킹에 과신 금지 |

---

## 6. Data Model — Employee KB

### 6.1 Source

- Obsidian vault (사내 매뉴얼, 검수 기준, 프롬프트, 온보딩, 트러블슈팅)
- 권한 티어(후속): 전체공개 / 팀 / 리더 — 초기 MVP는 공개·반공개만

### 6.2 Indexing

- LlamaIndex가 vault 경로를 **직접** 읽음 (불필요한 복사본 지양)
- 변화 범위가 크지 않으므로 **증분 upsert**
  - 파일 해시 또는 mtime으로 변경 감지
  - 신규·수정 → 재임베딩 후 upsert
  - 삭제 → 벡터/메타 delete
  - 전체 rebuild: 임베딩 모델·청크 스키마 변경 시에만

### 6.3 Storage

- Postgres + pgvector
- 메타데이터 예: `path`, `title`, `updated_at`, `doc_type`, `access_tier`

---

## 7. Agent Tools (Hermes)

최소 툴셋:

| Tool | Purpose |
|------|---------|
| `kb_search` | Employee KB 의미 검색 (LlamaIndex + pgvector) |
| `search_assets` | 소재 필터/자연어 → 후보 리스트 |
| `get_asset_card` | 미리보기 + 캠페인 + 최신심사 + 성과 카드 |
| `rank_by_kpi` | 전환기준·확정만으로 CPA/CTR 등 랭킹 |
| `check_reuse_risk` | 매체별 최신 반려/승인 요약 |

성공 응답에는 가능하면 **근거**(문서 path, review_id, perf_id)를 붙인다.

---

## 8. Product Surfaces (MVP)

### 8.1 Must

1. 케이스 12소재 정규화 적재 (또는 동등 샘플)
2. 소재 카드 조회 (미리보기 + 맥락 + 심사 + 성과)
3. 실무자 요청 4종에 답하는 검색/추천
4. Obsidian 샘플 문서 + 증분 인덱싱 + Hermes `kb_search`
5. “성과 좋음” / “다시 쓸 만함” 정의 문서

### 8.2 Suggested Definitions (초기)

- **성과 좋음 (purchase)**: `data_status=확정` AND 동일 카테고리·전환기준 내 CPA 상위 20% (또는 팀 합의 임계값)
- **다시 쓸 만함**: 성과 조건 + 대상 매체 최신 심사 `approved` (또는 revision 후 approved) + 잠정 성과 제외
- LPV와 purchase CPA는 **절대 섞어 랭킹하지 않음**

### 8.3 Later

- 이미지 임베딩(유사 소재)
- 검수 보조(반려 위험 표시)
- 매체 API 연동
- ACL·부서별 vault
- Knowledge Graph

---

## 9. Success Metrics

케이스 `work_context` 기준을 프로토타입 검증에 재사용:

| Metric | Baseline (case) | Target (MVP 가정) |
|--------|-----------------|-------------------|
| 첫 후보 전달 중앙값 | 17분 | ≤ 8분 (또는 대화형 즉시 후보) |
| 조건 재확인율 | 11/38 | 절반 이하 |
| 재전달 | 6건 | 감소 |
| 요청 4종 커버 | — | 각각 근거 포함 응답 가능 |

정량 측정이 어려우면: 과제 요청 4종에 대한 **골든 답안 체크리스트**로 대체.

---

## 10. Implementation Order

1. Postgres 스키마 (5 entities + pgvector 문서 테이블)
2. 태그 사전 1페이지 (Obsidian)
3. 케이스 데이터 정규화 적재
4. 소재 카드 API/조회
5. 필터 검색 → 그다음 자연어/RAG
6. Obsidian vault + LlamaIndex 증분 upsert
7. Hermes 툴 연결
8. (선택) 이미지 임베딩

**정형 join 조회가 RAG보다 먼저**다. 검색이 깨진 채 RAG만 올리지 않는다.

---

## 11. Risks & Open Questions

| Risk / Question | Handling |
|-----------------|----------|
| 성과 표본 커버리지 불명 | “행 없음 ≠ 0” 명시, unknown tier |
| “성과 좋음” 조직 표준 부재 | 초기 정의 문서화 후 현업 합의 |
| 영상은 대표 프레임만 | MVP는 썸네일+메타; 원본 분석은 later |
| 사내 문서 권한 | MVP는 공개·반공개; ACL later |
| Hermes는 업무 화면이 아님 | 소재 카드는 툴 결과/얇은 UI로 보완 |
| DeepTutor/유행 프레임워크 과신 | 엔진만 차용, UX·본체는 직접 조합 |

---

## 12. Decisions Log

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Hub 첫 도메인 | Creative Memory + Employee KB | 채용 멘트·과제와 일치 |
| UX | Hermes | DeepTutor 교육 UX 비적합; Agent·MCP에 가깝음 |
| 문서 저장 | Obsidian | 사람 편집·원본 vault |
| RAG | LlamaIndex | 문서 RAG 현역; DeepTutor가 감싸던 엔진 층 |
| Vector DB | Postgres + pgvector | join·운영 단순; 초기 규모 충분 |
| Indexing | 증분 upsert | 변화량 작음 |
| LangChain classic | 비사용 | Agent는 Hermes; RAG는 LlamaIndex |
| DeepTutor full | 비사용 | 패턴만 참고 |

---

## 13. References

- (주)원정대 AX 채용 공고 문구 (AX Hub, KB/RAG/Agent, 반려+성과 예시)
- 사전 과제: `ax_pre_interview_case_exercise` (README, work_context, assets/campaigns/review/performance, thumbnails)
- 업계 참고: creative library = performance-tagged knowledge system; fact/inference 분리
- 스택 참고: LlamaIndex / pgvector / Hermes Agent (Nous Research); DeepTutor는 멀티엔진 RAG 패턴만

---

## 14. Next Step

이 스펙 승인 후 implementation plan 작성:

- DB 마이그레이션·시드
- LlamaIndex upsert 파이프라인
- Hermes tool 스키마
- MVP 검증용 요청 4종 시나리오
