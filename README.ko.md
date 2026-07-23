# Negative-First Mesh Computing

[**한국어**](README.ko.md) | [English](README.en.md) | [日本語](README.ja.md) | [简体中文](README.zh-CN.md)

## 아니오 우선 그물망 추론 개요 및 설계

`Negative-First Mesh Computing`은 질문을 받자마자 가장 그럴듯한 답을 생성하는 대신, 가능한 후보를 그물망처럼 동시에 활성화한 뒤 **명백히 아닌 후보부터 먼저 끄는 방식**으로 답을 찾는 계산·추론 구조다.

이 구조의 핵심은 다음과 같다.

> 정답을 하나씩 찾지 않는다.  
> 전체 후보를 동시에 깨운 뒤, 아닌 것을 먼저 끄고 남은 것만 답한다.

일반적인 검색이나 생성형 AI는 후보를 순차적으로 탐색하거나, 현재 문맥에서 가장 가능성이 높은 출력을 계속 생성한다. 반면 Negative-First Mesh는 질문을 여러 조건 신호로 분해하고, 전체 후보망에 동시에 전달한 다음 불일치·모순·범위 밖·근거 부족 후보를 먼저 제거한다.

최종적으로 남아 있는 후보만 답변에 사용하며, 아무 후보도 남지 않으면 내용을 만들어내지 않고 다음 중 하나로 판정한다.

- 거짓
- 지정 범위 내에서 찾지 못함
- 자료 부족으로 판단 불가
- 질문 자체의 조건이 모순됨
- 다음 후보 그룹 탐색 필요

---

# 1. 문제 정의

현재 AI와 전통적인 컴퓨터는 많은 작업에서 다음과 같은 한계를 가진다.

## 1.1 순차 탐색 비용

데이터가 많아질수록 후보를 하나씩 비교하거나, 여러 단계의 검색·정렬·재검증을 반복해야 한다.

```text
후보 1 확인 → 제외
후보 2 확인 → 제외
후보 3 확인 → 유지
...
후보 N 확인
```

GPU와 벡터 검색은 많은 비교를 병렬화하지만, 논리적으로는 여전히 후보를 불러오고 점수를 계산하고 정렬하는 과정이 필요하다.

## 1.2 생성 우선 AI의 환각

생성형 AI는 관련 근거가 충분하지 않아도 언어적으로 자연스러운 다음 출력을 만들 수 있다.

```text
질문
  ↓
가능성 높은 문장 생성
  ↓
근거 부족 부분을 확률적으로 채움
  ↓
그럴듯하지만 틀린 답 발생
```

## 1.3 “없음”과 “모름”의 혼동

다음 세 문장은 서로 다르다.

- 실제로 존재하지 않는다.
- 지정한 데이터 범위에서는 찾지 못했다.
- 현재 가진 자료로는 알 수 없다.

기존 시스템은 이 차이를 명확히 구분하지 못하고 첫 검색 실패를 전체 부재처럼 표현하는 경우가 있다.

## 1.4 데이터와 연산의 분리

대부분의 컴퓨터에서는 데이터를 메모리에서 연산장치로 이동해야 한다. AI 모델이 커질수록 계산량뿐 아니라 메모리 이동과 데이터 탐색 비용이 병목이 된다.

Negative-First Mesh는 장기적으로 **데이터가 저장된 위치에서 조건 비교와 후보 억제가 함께 일어나는 구조**를 목표로 한다.

---

# 2. 핵심 개념

## 2.1 후보 그물망

정보를 하나의 파일이나 주소로만 저장하지 않고, 여러 의미와 관계를 가진 노드와 연결로 표현한다.

예를 들어 `사과`라는 정보는 다음 관계를 가질 수 있다.

```text
사과
├─ 분류: 과일
├─ 색상: 빨강, 초록
├─ 형태: 둥근 형태
├─ 기능: 먹을 수 있음
├─ 관계: 나무에서 열림
└─ 속성: 단맛, 신맛
```

질문이 들어오면 전체 문장을 그대로 검색하는 것이 아니라, 질문을 구성하는 조건들이 각각의 그물에 전달된다.

```text
질문: "먹을 수 있는 빨간 과일"

먹을 수 있음 ─┐
빨강 ─────────┼─ 사과 노드 활성
과일 ─────────┘
```

## 2.2 부정 우선 억제

각 후보는 처음에는 잠정 활성 상태로 시작한다. 질문 조건과 명백히 맞지 않는 후보는 즉시 꺼진다.

```text
후보 A: 과일이 아님         → OFF
후보 B: 먹을 수 없음       → OFF
후보 C: 빨간색 아님        → OFF
후보 D: 모든 조건 만족     → ON
```

하나의 필수 조건이라도 충족하지 못하면 후보를 더 계산하지 않는다.

## 2.3 0 감지

활성 후보 수가 0이 되는 순간을 별도의 계산 결과로 취급한다.

```text
활성 후보 > 0
→ 남은 후보 검증

활성 후보 = 0
→ 답 생성 금지
→ 부재·거짓·모름·범위 부족 판정
```

이 구조에서 `0`은 단순히 실패가 아니다. 다음 탐색 그룹으로 넘어가거나, 정확한 불확실성 상태를 출력하기 위한 중요한 신호다.

## 2.4 그룹 전이

정확한 표현에서 후보가 없으면 다음 그룹으로 확장한다.

```text
정확 표현
  ↓ 0
표기 변형
  ↓ 0
동의어·약어
  ↓ 0
상위·하위 개념
  ↓ 0
관계·원인·결과
  ↓ 0
판단 불가 또는 범위 내 없음
```

확장할수록 원래 질문과의 의미 거리가 커지므로, 각 후보에는 질문과의 거리 패널티를 부여한다.

## 2.5 근거 제한 답변

최종 답변은 살아남은 후보 중 근거가 충분한 항목으로만 만든다.

```text
ACTIVE_SUPPORTED
→ 답변에 직접 사용

ACTIVE_TENTATIVE
→ 추정 또는 가능성으로 표시

OFF_UNSUPPORTED
→ 답변에서 제외

UNKNOWN_UNRESOLVED
→ 판단 불가로 표시
```

---

# 3. 현재 기술과의 관계

Negative-First Mesh는 완전히 독립된 단일 기술이라기보다 다음 개념을 하나의 아키텍처로 통합한다.

## 3.1 내용주소 메모리

주소를 지정해 데이터를 읽는 대신, 찾고 싶은 내용을 입력하면 저장된 모든 항목이 동시에 비교되는 구조다.

Negative-First Mesh에서는 내용주소 비교를 단순 키 일치가 아니라 의미·관계·조건·시간·출처까지 확장한다.

## 3.2 연상기억

부분 단서만으로 관련 패턴 전체를 복원한다.

```text
부분 질문
→ 관련 노드 동시 활성
→ 경쟁과 억제
→ 안정된 패턴으로 수렴
```

## 3.3 뉴로모픽 계산

뉴런과 시냅스처럼 분산된 계산 요소가 이벤트에 반응하고, 필요한 부분만 활성화된다.

## 3.4 인메모리 컴퓨팅

메모리 내부 또는 가까운 위치에서 비교와 누산을 수행해 데이터 이동 비용을 줄인다.

## 3.5 파동·간섭 계산

전기 신호, 광 신호, 위상, 주파수 또는 시간차를 이용해 여러 후보의 보강과 상쇄를 동시에 처리한다.

## 3.6 그래프 추론

개체, 속성, 원인, 결과, 출처, 시간 관계를 노드와 연결로 표현한다.

Negative-First Mesh의 차별점은 이들을 다음 순서로 결합한다는 점이다.

```text
전체 후보 활성
→ 부정 조건 우선 전파
→ 후보 억제
→ 0 감지
→ 다음 그룹 전이
→ 근거가 남은 후보만 생성 AI에 전달
```

---

# 4. 목표

## 4.1 1차 목표

소프트웨어 기반으로 다음 기능을 구현한다.

- 질문 조건 자동 분해
- 후보 그룹 생성
- 여러 데이터 소스 동시 검색
- 불일치 후보 우선 제거
- 부재·거짓·판단 불가 구분
- 근거 기반 답변 생성
- 환각 억제
- 후보 제거 로그와 근거 추적

## 4.2 2차 목표

벡터 데이터베이스, 그래프 데이터베이스, 검색엔진, 규칙엔진을 결합해 대규모 병렬 연상 검색을 구현한다.

## 4.3 장기 목표

멤리스터, FPGA, 광학 회로, 뉴로모픽 칩 또는 다층 파동 네트워크에서 후보 활성과 억제를 물리적으로 처리한다.

---

# 5. 전체 아키텍처

```text
┌──────────────────────────────┐
│          사용자 질문          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 1. Question Normalizer       │
│ - 대상·주장·조건·범위 분해     │
│ - 시점·위험도·출력유형 판정     │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 2. Mesh Planner              │
│ - 후보 그룹 생성              │
│ - 검색 축 및 관계 축 생성      │
│ - 병렬 탐색 계획 수립          │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 3. Candidate Activator       │
│ - 정확 후보 활성              │
│ - 유사 후보 활성              │
│ - 관계 후보 활성              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 4. Negative Gate Engine      │
│ - 정의 불일치 제거            │
│ - 필수조건 위반 제거           │
│ - 시점 불일치 제거            │
│ - 모순 후보 제거              │
│ - 근거 부족 후보 제거          │
│ - 안전 기준 미충족 후보 제거    │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 5. Zero Detector             │
│ - 활성 후보 수 계산           │
│ - 부재·거짓·판단불가 분류      │
│ - 다음 그룹 전이 여부 결정      │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 6. Evidence Resolver         │
│ - 출처 신뢰도                │
│ - 최신성                     │
│ - 직접성                     │
│ - 후보 간 모순               │
│ - 질문과의 거리              │
└──────────────┬───────────────┘
               ↓
┌──────────────────────────────┐
│ 7. Answer Composer           │
│ - 살아남은 근거만 사용         │
│ - 확정·추정·모름 구분          │
│ - 범위와 한계 표시            │
└──────────────────────────────┘
```

---

# 6. 주요 컴포넌트

## 6.1 Question Normalizer

질문을 구조화된 프레임으로 변환한다.

```json
{
  "target": "찾으려는 대상",
  "claim": "검증하려는 주장",
  "required_conditions": [],
  "excluded_conditions": [],
  "scope": {
    "sources": [],
    "time_range": null,
    "region": null,
    "closed_world": false
  },
  "freshness_required": false,
  "risk_level": "LOW | MEDIUM | HIGH",
  "answer_type": "FACT | SEARCH | DIAGNOSIS | DESIGN | RECOMMENDATION"
}
```

### 역할

- 애매한 대상을 분리
- 부정 조건과 필수 조건 추출
- 검색 범위가 닫혀 있는지 판정
- 최신 정보가 필요한지 판단
- 잘못된 답의 위험도를 설정

---

## 6.2 Mesh Planner

후보를 탐색할 축과 순서를 만든다.

### 기본 후보 그룹

1. 정확한 일치
2. 표기 변형
3. 동의어·약어
4. 의미 유사
5. 상위 개념
6. 하위 개념
7. 시간 인접
8. 지역 인접
9. 원인·결과
10. 반대 개념
11. 출처 교차검증
12. 사용자 전제 대안

### 그룹 정의 예시

```json
{
  "group_id": "G03",
  "group_type": "SYNONYM",
  "distance": 0.15,
  "activation_policy": "ON_PREVIOUS_ZERO",
  "queries": [
    "오프라인 사용",
    "인터넷 연결 없이 실행",
    "초기 인증 후 로컬 사용"
  ]
}
```

---

## 6.3 Candidate Activator

검색 결과, 규칙 결과, 그래프 경로, 모델 추정치를 후보 노드로 만든다.

```json
{
  "candidate_id": "C1024",
  "label": "후보 설명",
  "group_id": "G03",
  "state": "ACTIVE_TENTATIVE",
  "matched_conditions": [],
  "failed_conditions": [],
  "evidence_ids": [],
  "distance": 0.15
}
```

모든 후보를 실제로 동시에 처리할 수 없는 환경에서도 논리적으로 같은 그룹의 후보를 한 번에 배치 처리한다.

---

## 6.4 Negative Gate Engine

Negative-First Mesh의 핵심 컴포넌트다.

### 정의 게이트

후보와 질문 대상의 정의가 같은지 검사한다.

```text
질문: 정기 구독이 없는 일회성 구매 앱
후보: 매월 결제가 필요한 구독형 서비스

정의 불일치 → OFF_FALSE
```

### 조건 게이트

필수 조건 하나라도 위반하면 후보를 끈다.

```text
조건:
- 100만 원 이하
- 메모리 16GB 이상
- 정기 구독 없음

후보가 월 구독 전용이면 → OFF_FALSE
```

### 시간 게이트

자료 시점이 요청 시점에 적합한지 확인한다.

```text
2024년 출시 가격
현재 판매 가격 질문
→ OFF_OUT_OF_SCOPE
```

### 존재 게이트

닫힌 범위에서는 실제 존재 여부를 판정한다.

```text
업로드한 사용설명서 3개 전체 검색
오프라인 사용 문구 없음
→ NOT_FOUND_IN_SCOPE
```

열린 세계에서는 검색 실패를 부재로 확정하지 않는다.

```text
웹 검색에서 발견 안 됨
→ UNKNOWN_UNRESOLVED
```

### 모순 게이트

더 강한 근거나 내부 모순이 있으면 후보를 제거한다.

### 근거 게이트

답에 포함할 직접 근거가 없으면 `OFF_UNSUPPORTED`로 전환한다.

### 인과 게이트

상관관계를 원인으로 확정하지 않는다.

### 안전 게이트

의료·법률·금융·보안 등 고위험 분야에서는 더 높은 근거 임계값을 적용한다.

---

# 7. 후보 상태 모델

| 상태 | 설명 |
|---|---|
| `ACTIVE_SUPPORTED` | 강한 근거가 있어 답 후보로 유지 |
| `ACTIVE_TENTATIVE` | 가능성이 있으나 추가 검증 필요 |
| `OFF_FALSE` | 질문 조건과 명백히 다름 |
| `OFF_CONTRADICTED` | 더 강한 근거나 모순으로 제거 |
| `OFF_OUT_OF_SCOPE` | 현재 시간·지역·자료 범위 밖 |
| `OFF_UNSUPPORTED` | 주장할 근거가 부족 |
| `UNKNOWN_UNRESOLVED` | 자료 부족으로 참·거짓 판정 불가 |
| `NOT_FOUND_IN_SCOPE` | 닫힌 검증 범위에서 발견되지 않음 |

중요 규칙:

```text
근거가 없음 ≠ 거짓
찾지 못함 ≠ 존재하지 않음
가능성이 낮음 ≠ 불가능
```

---

# 8. 0 감지와 다음 그룹 전이

## 8.1 0 감지 조건

```text
ACTIVE_SUPPORTED 수 = 0
AND
ACTIVE_TENTATIVE 수 = 0
```

## 8.2 0 상태 분류

```text
질문 조건 자체가 모순
→ CONTRADICTED_QUERY

닫힌 데이터 범위에서 전수 확인
→ NOT_FOUND_IN_SCOPE

강한 반례로 주장 불성립
→ FALSE

현재 소스 부족
→ UNKNOWN_UNRESOLVED

다음 유사 그룹 존재
→ EXPAND_NEXT_GROUP
```

## 8.3 그룹 전이 정책

```text
G1 정확 일치
  ↓ 0
G2 표기 변형
  ↓ 0
G3 동의어
  ↓ 0
G4 의미 유사
  ↓ 0
G5 상위·하위 개념
  ↓ 0
G6 관계·원인·결과
  ↓ 0
최종 0 판정
```

각 그룹은 질문과의 의미 거리를 가진다.

```text
distance = 0.00  정확 일치
distance = 0.10  표기 변형
distance = 0.20  동의어
distance = 0.35  의미 유사
distance = 0.50  상위·하위 개념
distance = 0.70  인접 관계
```

거리가 멀수록 답변에서 더 약한 확정 표현을 사용한다.

---

# 9. 신뢰도 계산

후보의 신뢰도는 다음 요소로 계산할 수 있다.

- `D`: 직접 근거 강도
- `S`: 출처 신뢰도
- `C`: 조건 충족도
- `R`: 최신성
- `X`: 모순 패널티
- `G`: 질문과 후보 사이 거리 패널티

```text
confidence =
D × S × C × R × (1 - X) × (1 - G)
```

권장 판정:

| 점수 | 판정 |
|---:|---|
| 0.85 이상 | 확인됨 |
| 0.65~0.84 | 상당히 가능 |
| 0.40~0.64 | 불확실 |
| 0.40 미만 | 답 후보에서 제거하거나 추정으로만 표시 |

이 점수는 수학적으로 보장된 확률이 아니라 내부 비교와 우선순위를 위한 지표다.

---

# 10. 답변 생성 정책

## 10.1 기본 응답 구조

```text
[직접 답]
가장 근거가 강한 결론

[판정]
확인됨 / 상당히 가능 / 불확실 / 범위 내 없음 / 판단 불가 / 거짓

[검증 범위]
어떤 데이터와 시점을 확인했는지

[먼저 제거된 후보]
핵심적으로 아닌 후보와 이유

[남은 불확실성]
아직 확인되지 않은 부분
```

일반 사용자 답변에서는 내부 상태 이름을 그대로 노출하지 않고 자연스러운 언어로 표현한다.

## 10.2 답변 금지 조건

다음 상황에서는 확정 답을 생성하지 않는다.

- 모든 후보가 `OFF_UNSUPPORTED`
- 후보가 남았지만 서로 강하게 모순됨
- 최신 검증이 필요한데 현재 자료가 없음
- 닫힌 범위인지 열린 범위인지 구분되지 않음
- 질문의 핵심 조건이 모호해 결과가 크게 달라짐
- 고위험 문제에서 근거 임계값을 충족하지 못함

---

# 11. 소프트웨어 구현 설계

## 11.1 권장 구성

```text
Frontend
- 질문 입력
- 검증 범위 선택
- 후보망 시각화
- 제거 이유 표시
- 최종 답과 근거 표시

API
- 질문 정규화
- 후보 그룹 생성
- 검색·도구 오케스트레이션
- 부정 게이트 실행
- 0 감지
- 답변 생성

Storage
- PostgreSQL: 질문, 후보, 상태, 근거, 실행 로그
- Redis: 활성 후보, 그룹 큐, 임시 점수
- Vector DB: 의미 유사 검색
- Graph DB 또는 PostgreSQL 그래프 모델: 관계 탐색
- Object Storage: 원문, 첨부파일, 증거 스냅샷

Workers
- exact-search-worker
- semantic-search-worker
- graph-search-worker
- contradiction-worker
- evidence-verification-worker
- freshness-check-worker
```

---

# 12. 데이터 모델

## 12.1 questions

```sql
CREATE TABLE questions (
    id BIGSERIAL PRIMARY KEY,
    raw_question TEXT NOT NULL,
    normalized_target TEXT,
    normalized_claim TEXT,
    answer_type VARCHAR(32),
    risk_level VARCHAR(16),
    closed_world BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.2 candidate_groups

```sql
CREATE TABLE candidate_groups (
    id BIGSERIAL PRIMARY KEY,
    question_id BIGINT NOT NULL REFERENCES questions(id),
    group_type VARCHAR(32) NOT NULL,
    semantic_distance NUMERIC(6,5) NOT NULL DEFAULT 0,
    sequence_no INTEGER NOT NULL,
    activated_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ
);
```

## 12.3 candidates

```sql
CREATE TABLE candidates (
    id BIGSERIAL PRIMARY KEY,
    group_id BIGINT NOT NULL REFERENCES candidate_groups(id),
    label TEXT NOT NULL,
    state VARCHAR(32) NOT NULL,
    confidence NUMERIC(6,5),
    matched_conditions JSONB NOT NULL DEFAULT '[]',
    failed_conditions JSONB NOT NULL DEFAULT '[]',
    disabled_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 12.4 evidence

```sql
CREATE TABLE evidence (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    source_type VARCHAR(32) NOT NULL,
    source_uri TEXT,
    source_title TEXT,
    published_at TIMESTAMPTZ,
    retrieved_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    directness NUMERIC(6,5),
    reliability NUMERIC(6,5),
    freshness NUMERIC(6,5),
    excerpt TEXT
);
```

## 12.5 gate_results

```sql
CREATE TABLE gate_results (
    id BIGSERIAL PRIMARY KEY,
    candidate_id BIGINT NOT NULL REFERENCES candidates(id),
    gate_type VARCHAR(32) NOT NULL,
    passed BOOLEAN NOT NULL,
    resulting_state VARCHAR(32),
    reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

# 13. 핵심 API

## 질문 실행

```http
POST /api/mesh/questions
Content-Type: application/json
```

```json
{
  "question": "업로드한 안내서에 오프라인 사용 기능이 명시되어 있는가?",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "mode": "negative_first"
}
```

## 실행 결과

```json
{
  "status": "CONFIRMED",
  "answer": "초기 인증 후 오프라인으로 사용할 수 있다는 문구가 확인되었습니다.",
  "scope": {
    "document_ids": [101, 102, 103],
    "closed_world": true
  },
  "active_candidates": [
    {
      "candidate": "최초 활성화 이후에는 인터넷 연결 없이 실행 가능",
      "confidence": 0.93
    }
  ],
  "disabled_candidates": [
    {
      "candidate": "클라우드 동기화 기능",
      "state": "OFF_FALSE",
      "reason": "오프라인 실행 가능 여부를 증명하지 않음"
    }
  ],
  "uncertainties": []
}
```

---

# 14. 핵심 알고리즘

```text
function answer_with_negative_first_mesh(question, sources):

    frame = normalize_question(question)
    groups = build_candidate_groups(frame)

    for group in groups:

        candidates = activate_candidates(group, sources)

        parallel_for candidate in candidates:

            if not definition_matches(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if violates_required_condition(candidate, frame):
                disable(candidate, OFF_FALSE)
                continue

            if outside_time_or_scope(candidate, frame):
                disable(candidate, OFF_OUT_OF_SCOPE)
                continue

            if contradicted(candidate):
                disable(candidate, OFF_CONTRADICTED)
                continue

            if lacks_required_evidence(candidate):
                disable(candidate, OFF_UNSUPPORTED)
                continue

            candidate.state = score_candidate(candidate)

        survivors = get_survivors(candidates)

        if survivors is not empty:
            verified = cross_check(survivors)

            if verified is not empty:
                return compose_answer_from_verified_only(verified)

        zero_state = classify_zero_state(candidates, frame)

        if zero_state == EXPAND_NEXT_GROUP:
            continue

        return report_zero_state(zero_state)

    return {
        "status": "UNKNOWN",
        "answer": "현재 확인 가능한 범위에서는 판단할 근거가 부족합니다."
    }
```

---

# 15. AI 모델과의 결합

Negative-First Mesh는 생성 AI를 제거하기 위한 구조가 아니다. 생성 AI의 역할을 줄이고, 근거가 있는 결과를 설명하는 단계로 제한한다.

## 기존 구조

```text
질문
→ 대형 언어모델
→ 검색 또는 추정
→ 답 생성
```

## 제안 구조

```text
질문
→ 조건 분해
→ 후보망 전체 활성
→ 아닌 후보 우선 제거
→ 근거 검증
→ 남은 후보만 소형 또는 대형 언어모델에 전달
→ 설명 생성
```

### 기대 효과

- 환각 감소
- 불필요한 토큰 사용 감소
- 검색 범위 축소
- 답변 근거 추적 가능
- “모름” 판정 향상
- 폐쇄형 기업 데이터에서 정확한 부재 확인
- 여러 AI 모델 간 교차검증 가능

---

# 16. 물리 그물망 확장

소프트웨어 구현이 안정화되면 일부 연산을 물리적 그물망으로 이전할 수 있다.

## 16.1 가능한 매체

- 멤리스터 크로스바
- FPGA 병렬 비교망
- 광학 간섭기
- 초전도 회로
- 스핀파 소자
- 뉴로모픽 칩
- 다층 PCB 전송망
- 주파수·위상 다중화 회로

## 16.2 물리 노드 역할

```text
입력 신호
→ 여러 경로로 동시 전파
→ 조건 불일치 경로 상쇄
→ 조건 일치 경로 보강
→ 임계값 이상 노드만 유지
→ 출력 센서가 활성 패턴 측정
```

## 16.3 가상 다층화

실제 기판을 무한히 쌓는 대신 하나의 물리 경로에 여러 논리 층을 겹친다.

- 서로 다른 주파수
- 서로 다른 위상
- 서로 다른 시간 슬롯
- 서로 다른 광파장
- 서로 다른 편광
- 서로 다른 코드 패턴
- 서로 다른 방향

이를 통해 제한된 물리 자원으로 더 큰 논리 상태공간을 구성할 수 있다.

---

# 17. 성능 목표

## 소프트웨어 1단계

- 100만 후보 내 정확 조건 병렬 제거
- 후보 그룹별 배치 검색
- 1초 이내 1차 0 판정
- 근거 추적 100%
- 부재·모름 오판 테스트 통과

## 소프트웨어 2단계

- 벡터·그래프·규칙 결합
- 다중 데이터 소스 동시 검색
- 후보 수천만 건 분산 처리
- 실시간 활성 후보 시각화
- LLM 호출 전 후보 90% 이상 제거

## 하드웨어 실험 단계

- 8×8 또는 16×16 그물망
- 동일 위상 보강 검증
- 반대 위상 상쇄 검증
- 다중 입력 임계값 판정
- 후보 0 상태 전기적 감지
- FPGA와 연계한 그룹 전이 제어

---

# 18. 테스트 기준

## 사실성

- 근거 부족을 거짓으로 판단하지 않는가
- 오래된 정보를 현재 사실로 사용하지 않는가
- 상관관계를 원인으로 단정하지 않는가

## 범위성

- 닫힌 범위에서만 부재를 확정하는가
- 외부 범위를 확인하지 않았음을 표시하는가
- 질문에서 너무 멀어진 후보를 답으로 둔갑시키지 않는가

## 0 처리

- 후보가 없을 때 내용을 생성하지 않는가
- 다음 그룹을 합리적으로 탐색하는가
- `거짓`, `범위 내 없음`, `판단 불가`를 구분하는가

## 답변 품질

- 결론이 먼저 나오는가
- 제거한 핵심 후보를 간단히 설명하는가
- 내부 추론 전체를 불필요하게 노출하지 않는가
- 근거와 불확실성이 함께 표시되는가

---

# 19. 한계

## 19.1 실제 무한 상태는 불가능

유한한 장치에 무한한 독립 정보를 저장할 수는 없다. 다만 주파수·위상·시간·공간·연결 조합을 이용해 매우 큰 상태공간을 만들 수 있다.

## 19.2 물리 자원 비용

완전 병렬 처리는 계산 시간을 줄이는 대신 더 많은 메모리, 연결, 전력, 칩 면적을 사용한다.

## 19.3 모든 노드 완전 연결 문제

노드 수가 증가하면 연결 수가 급격히 늘어난다. 따라서 실제 구현은 다음과 같은 계층형 구조가 필요하다.

- 지역 그물
- 전문 분야 그물
- 상위 요약 그물
- 희소 연결
- 필요할 때만 활성화되는 경로
- 파장·주파수 기반 가상 연결

## 19.4 의미 비교의 불확실성

정확 키 비교와 달리 의미 비교는 확률적이다. 따라서 의미 유사 후보는 확정 답이 아니라 잠정 후보로 유지해야 한다.

## 19.5 부재 증명의 어려움

열린 세계에서는 검색하지 못했다고 존재하지 않는다고 말할 수 없다. Negative-First Mesh는 이 한계를 숨기지 않고 판정 상태로 명시한다.

---

# 20. 개발 단계

## Phase 1. 스킬 및 규칙 엔진

- 질문 정규화
- 후보 상태 모델
- Negative Gate
- 0 감지
- 그룹 전이
- 답변 템플릿
- 테스트 시나리오

## Phase 2. 검색 오케스트레이터

- 정확 검색
- 의미 검색
- 그래프 검색
- 파일 검색
- 웹 검색
- 데이터베이스 검색
- 교차검증

## Phase 3. 운영 플랫폼

- 질문 실행 이력
- 후보망 시각화
- 게이트 통과·제거 이유
- 근거 관리
- 관리자 임계값 설정
- 분야별 정책 프로필
- 감사 로그

## Phase 4. AI 통합

- 다중 LLM 후보 생성
- 후보 간 모순 검사
- 근거 없는 후보 자동 OFF
- 살아남은 컨텍스트만 LLM에 전달
- 답변 신뢰도 및 부재 판정

## Phase 5. 하드웨어 실험

- FPGA 기반 후보 병렬 비교
- 아날로그 신호 보강·상쇄
- 멤리스터 또는 광학 그물 실험
- 소프트웨어 Mesh Engine과 하드웨어 가속기 연동

---

# 21. 프로젝트 구성

```text
negative-first-mesh-skill/
├── README.md
├── README.ko.md
├── README.en.md
├── README.ja.md
├── README.zh-CN.md
├── SKILL.md
├── SKILL.ko.md
├── SKILL.en.md
├── SKILL.ja.md
├── SKILL.zh-CN.md
├── INSTALL.md
├── INSTALL.ko.md
├── INSTALL.en.md
├── INSTALL.ja.md
├── INSTALL.zh-CN.md
├── EXAMPLES.md
├── EXAMPLES.ko.md
├── EXAMPLES.en.md
├── EXAMPLES.ja.md
├── EXAMPLES.zh-CN.md
├── TESTS.md
├── TESTS.ko.md
├── TESTS.en.md
├── TESTS.ja.md
├── TESTS.zh-CN.md
└── packages/
    ├── negative-first-mesh-ko.zip
    ├── negative-first-mesh-en.zip
    ├── negative-first-mesh-ja.zip
    └── negative-first-mesh-zh-CN.zip
```

---

# 22. 한 문장 정의

> Negative-First Mesh Computing은 질문을 전체 후보망에 동시에 전파하고, 불일치·모순·범위 밖·근거 부족 후보를 먼저 억제한 뒤, 남은 근거만으로 답하거나 활성 후보가 0인 이유를 정확히 판정하는 계산·추론 아키텍처다.

---

# 23. 핵심 원칙 요약

```text
답을 먼저 만들지 않는다.
후보를 먼저 펼친다.
아닌 것을 먼저 끈다.
0을 실패로 보지 않는다.
0이면 다음 그룹으로 간다.
끝까지 없으면 만들지 않는다.
근거가 남은 것만 말한다.
거짓, 없음, 모름을 구분한다.
```
---

# 24. 현존 LLM에 적용하기

이 스킬은 Codex, Claude Code, Gemini CLI에 직접 설치할 수 있다.

| 대상 | 프로젝트 설치 위치 | 사용자 전체 설치 위치 |
|---|---|---|
| Codex | `.agents/skills/negative-first-mesh/SKILL.md` | `~/.agents/skills/negative-first-mesh/SKILL.md` |
| Claude Code | `.claude/skills/negative-first-mesh/SKILL.md` | `~/.claude/skills/negative-first-mesh/SKILL.md` |
| Gemini CLI | `.gemini/skills/negative-first-mesh/SKILL.md` 또는 `.agents/skills/...` | `~/.gemini/skills/negative-first-mesh/SKILL.md` 또는 `~/.agents/skills/...` |

언어별 `SKILL.ko.md`, `SKILL.en.md`, `SKILL.ja.md`, `SKILL.zh-CN.md` 중 하나를 선택하고 설치 위치에서 정확히 `SKILL.md`로 복사한다. 네이티브 스킬을 지원하지 않는 LLM에서는 프로젝트 지침이나 시스템·개발자 프롬프트로 조건부 적용한다.

상세 방법은 [INSTALL.ko.md](INSTALL.ko.md)를 참고한다.
