# AX Hub — Knowledge Base Design Spec

**Date:** 2026-08-04  
**Status:** Draft  
**Owner:** AX (입사 전 프로토타입 / 입사 후 Hub 기반)

---

## 1. Purpose

(주)원정대 AX팀 목표에 맞춰, 사내 업무에 흩어진 데이터·문서·기준·처리 이력을 AI가 쓸 수 있게 연결하는 **AX Hub**의 지식 기반(KB)을 설계한다.

핵심은 단순 챗봇이 아니라:

1. **Employee KB** — 전사 직원이 쓰는 문서 지식 (매뉴얼, 기준, 러닝)
2. **Creative Memory** — 광고 소재·성과·검수 맥락이 붙은 운영 지식

둘을 같은 Hub 아래에서 분리해 쌓고, Hermes로 질의한다.

---

## 2. Company & AX Context

### 원정대

- 숏폼 커머스 콘텐츠를 만드는 퍼포먼스·콘텐츠 기업
- 뷰티·리빙·가전·패션 등 브랜드 크리에이티브 기획·촬영
- 조회수가 아니라 **전환·매출**로 이어지는 콘텐츠가 성공 정의

### AX팀

- AI Transformation: 반복 업무·병목을 AI·시스템으로 전환
- 최종 지향: **AX Hub** (데이터·문서·기준·처리 이력 연결)
- 광고 운영, 보고, 콘텐츠 제작/검수, 소재 분석·생성 등에 AI 적용
- 작은 프로토타입 → 현업 피드백 → 개선
- KB / RAG / Agent 구조를 실험·문서화해 팀 자산으로 남김

채용·팀 방향에서 강조된 예시: 광고 소재의 **반려 사유와 성과**를 참고한 검수·생성 개선, 사내 데이터의 AI 활용 구조화.

---

## 3. Goals & Non-Goals

### Goals

- 전사 문서 KB를 Obsidian에 쌓고, LlamaIndex + pgvector로 검색·질의
- 광고 소재 관련 정형 지식(소재·캠페인·심사·성과·러닝)을 Postgres에 구조화
- Hermes를 직원 접점 UX/Agent로 사용
- 문서 변경량이 크지 않으므로 **증분 upsert** 인덱싱
- 사실 / 추론 / 가설을 섞지 않는 저장 규칙

### Non-Goals (초기)

- DeepTutor 등 교육용 제품 통째 도입 (RAG 엔진 아이디어만 참고 가능)
- LangChain 구형 Chain 스택
- 전용 벡터DB(Qdrant 등)를 처음부터 도입
- 전체 매체 API 실시간 연동, KG 풀구축, 대규모 인프라

---

## 4. Architecture

```
┌─────────────────────────────────────────────┐
│  Hermes (UX / Agent / Tools·MCP)            │
└─────────────────┬───────────────────────────┘
                  │
      ┌───────────┴───────────┐
      ▼                       ▼
┌───────────────┐     ┌──────────────────────┐
│ Employee KB   │     │ Creative Memory      │
│               │     │                      │
│ Obsidian md   │     │ Asset / Campaign /   │
│      ↓        │     │ Review / Performance │
│ LlamaIndex    │     │ / Learning           │
│      ↓        │     │      ↓               │
│ Postgres      │     │ Postgres (SQL join)  │
│ + pgvector    │     │                      │
└───────────────┘     └──────────────────────┘
```

### Role split

| Component | Role |
|-----------|------|
| **Obsidian** | 사람이 쓰는 원본 문서 저장소 (md vault) |
| **LlamaIndex** | 청킹·임베딩·인덱싱·검색 |
| **Postgres + pgvector** | 벡터 + 정형 메타/업무 데이터 |
| **Hermes** | 직원 질의 UX, 툴 호출 |

Obsidian과 LlamaIndex는 경쟁이 아니라 보완이다.  
**저장 = Obsidian, 인덱싱/벡터 = LlamaIndex + pgvector, UX = Hermes.**

---

## 5. Stack Decisions

| Topic | Decision | Why |
|-------|----------|-----|
| UX | Hermes | Agent·툴·MCP에 가깝고, 교육용 UX(DeepTutor)보다 사내용에 맞음 |
| 문서 원본 | Obsidian | 단순 저장소·사람 편집 |
| RAG | LlamaIndex | 문서 RAG에 현역. DeepTutor가 감싸던 엔진 층만 차용하는 방향과 일치 |
| Vector DB | **pgvector (Postgres)** | 회사 실무에서 흔한 기본값. 소재·메타 join, 운영 단순. 초기 규모(수천만 벡터 이하)에 충분 |
| Indexing | 파일 해시/mtime 기반 **증분 upsert** | 변화 범위가 크지 않음 |
| Agent orchestration | Hermes (LangChain classic 비사용) | RAG는 LlamaIndex, Agent는 Hermes로 역할 분리 |
| Qdrant 등 | later 후보 | 필터·규모 한계가 보일 때 검토 |

---

## 6. Employee KB

### Source

- Obsidian vault: 매뉴얼, 검수 기준, 운영 가이드, 프롬프트, 온보딩, 트러블슈팅, 러닝 노트

### Pipeline

1. LlamaIndex가 vault 경로를 **직접** 읽음 (불필요 복사본 지양)
2. 변경 감지(해시 또는 mtime)
3. 신규·수정 → 재임베딩 후 **upsert**
4. 삭제 → 벡터/메타 delete
5. 전체 rebuild는 임베딩 모델·청크 스키마 변경 시에만

### Vector store

- Postgres + pgvector
- 메타 예: `path`, `title`, `updated_at`, `doc_type`, (후속) `access_tier`

### Access (later)

- 전체공개 / 팀 / 리더 ACL은 후속. 초기는 공개·반공개 문서 위주.

---

## 7. Creative Memory (광고 KB 코어)

광고 관련 KB의 중심은 회사 소개가 아니라  
**소재 × 캠페인 맥락 × 심사(재사용 가능) × 성과 × 러닝** 연결이다.

### Entities

| Entity | 담을 것 |
|--------|---------|
| **Asset** | 소재 ID, 유형, 경로, 미리보기, 통제 태그, 메모 |
| **Campaign** | 브랜드, 카테고리, 타깃, 목표, **전환기준**, 매체, 기간 |
| **Review** | 소재×캠페인×매체, 승인/반려/수정요청, 사유, 조치, 일자 (현재 상태는 최신 이벤트로 해석) |
| **Performance** | 기간 성과, spend/CTR/전환 등, **확정/잠정** |
| **Learning** | 가설 → 결과 → 다음 테스트 (짧은 텍스트; Obsidian mirror 가능) |

연결 키: `asset_id`, `campaign_id`.

### Controlled tags (자유 태그 지양)

- `format`: image / video_9x16 / …
- `angle`: product_cut / usage_scene / before_after / ugc / lookbook / promo
- `hook`: morning_routine / social_proof / offer / …
- `audience`: 합의된 소수 값
- `reuse_status`: 매체별 최신 심사에서 유도
- `perf_tier`: 동일 전환기준·확정 데이터만으로 부여

### Fact / Inference / Hypothesis

| Layer | Example | Rule |
|-------|---------|------|
| Fact | CTR, approved | DB에 숫자·상태로 |
| Inference | “아침 루틴이 먹힌다” | Learning + 추론 표시 |
| Hypothesis | “다른 상품에도 전이” | Learning만, 랭킹 과신 금지 |

### KPI 해석 규칙

- 전환기준이 다르면 CPA 등을 **섞어 랭킹하지 않음**
- `잠정` 성과는 의사결정·보고에 넣을 때 명시하거나 제외
- “성과 좋음” / “다시 쓸 만함” 정의는 태그 사전과 함께 Obsidian에 문서로 고정 (현업 합의로 갱신)

초기 예시 정의:

- **성과 좋음**: 확정 + 동일 전환기준·카테고리 내 합의 KPI(예: CPA 상위 구간)
- **다시 쓸 만함**: 성과 조건 + 대상 매체 최신 심사 approved (+ 잠정 제외)

---

## 8. Hermes Tools (최소)

| Tool | Purpose |
|------|---------|
| `kb_search` | Employee KB 의미 검색 |
| `search_assets` | 소재 후보 검색 |
| `get_asset_card` | 미리보기 + 캠페인 + 최신심사 + 성과 |
| `rank_by_kpi` | 전환기준·확정만 반영한 랭킹 |
| `check_reuse_risk` | 매체별 반려/승인 요약 |

응답에는 가능하면 근거(문서 path, review/perf id)를 붙인다.

Hermes는 범용 Agent UX이므로, 소재 카드형 업무 화면은 툴 결과 또는 얇은 UI로 보완한다.

---

## 9. Implementation Order

1. Postgres 스키마 (Creative Memory 엔티티 + pgvector 문서 테이블)
2. Obsidian 태그/정의 사전
3. Creative Memory 샘플 적재 + 소재 카드 조회
4. 필터 검색 → 자연어/툴 연결
5. Obsidian + LlamaIndex 증분 upsert
6. Hermes 툴 연결
7. (later) 이미지 임베딩, ACL, 매체 API, Qdrant 이전 검토

**정형 join 조회를 RAG보다 먼저** 만든다.

---

## 10. Decisions Log

| Decision | Choice |
|----------|--------|
| Hub 구성 | Employee KB + Creative Memory |
| UX | Hermes |
| 문서 저장 | Obsidian |
| RAG | LlamaIndex |
| Vector DB | Postgres + pgvector |
| Indexing | 증분 upsert |
| DeepTutor | 엔진 패턴만 참고, UX/본체 비사용 |
| LangChain classic | 비사용 |

---

## 11. Next

스펙 확정 후 implementation plan:

- DB 마이그레이션·시드
- LlamaIndex upsert 파이프라인
- Hermes tool 스키마
- MVP 검증 시나리오
