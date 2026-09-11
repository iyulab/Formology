# 도출 원리

> **"양식의 구조는 그대로 데이터 구조가 된다"**

문서에서 데이터 모델·API·인터페이스가 어떻게 도출되는지를 다루는 **기술 중립적 원리** 문서입니다. 이것이 사상 규칙의 **정본**입니다.

> 용어 정본: [용어 사전](glossary.md). `FormType(서식) / Form(양식) / Document(문서) / Record(레코드)`

**핵심 원칙**:
1. **기술 중립성** — 특정 기술에 종속되지 않는 보편적 원리
2. **구조 보존** — 양식 구조 ≅ 구현 구조 (동형 사상)
3. **검증 가능성** — 현업이 도출 결과를 확인·검증 가능
4. **위임** — 스택 선택·성능·배포는 구현체의 몫이며 이 문서가 다루지 않는다

> **Formology에는 구현체가 없습니다.** 아래 코드는 규칙이 실제로 작동함을 보이기 위한 **예시**이며, 권장 스택도 템플릿도 아닙니다. 예시가 SQL인 것은 가장 널리 읽히기 때문이지 PostgreSQL을 권해서가 아닙니다.

---

## 1. 사상의 원리

### 1.1 기본 사상 규칙

#### 계층 사상

```
서식 (FormType)  → 스키마/인터페이스
양식 (Form)      → 구조/클래스
문서 (Document)  → 인스턴스/레코드
```

**예시**:
```
[양식] 품질검사의뢰서: 제품명, LOT번호, 수량
  ↓
[구현] QCRequest: product_id, lot_number, quantity
```

### 1.2 구조적 사상

#### 양식 → 엔티티

**규칙**: 하나의 양식 = 하나의 주요 엔티티

```
[양식]                [엔티티]
품질검사의뢰서         QCRequest
├─ 의뢰번호           ├─ request_number
├─ 제품명 [선택]      ├─ product_id (FK)
├─ LOT번호            ├─ lot_number
└─ 수량               └─ quantity
```

**예외**:
- 복합 문서: 여러 엔티티로 분해
- 섹션이 많은 문서: 정규화 고려

#### 섹션 → 엔티티 분해

**판별 기준**:
1. **반복성**: 섹션이 여러 번 나타나는가?
2. **독립성**: 독자적 생명주기를 가지는가?
3. **재사용성**: 다른 양식에서도 사용되는가?

```
[주문서]
헤더
  ├─ 주문번호
  └─ 고객명
품목 (반복) ← 별도 엔티티
  ├─ 제품명
  └─ 수량

[사상]
Order (1)
  └─ OrderItem (N)
```

#### 필드 → 속성

| 필드 유형 | 데이터 타입 | 제약조건 |
|-----------|-------------|----------|
| [입력] | VARCHAR, TEXT | - |
| [숫자] | INTEGER, DECIMAL | - |
| [날짜] | DATE, TIMESTAMP | - |
| [선택] | FK 또는 ENUM | NOT NULL |
| [체크박스] | BOOLEAN | - |
| 본질적 | (위 타입) | NOT NULL |
| 부가적 | (위 타입) | NULLABLE |

### 1.3 관계 사상

#### 참조 (Reference) — 살아있는 링크

```
양식: 문서 A --참조--> 문서 B
  ↓
구현: Entity_A.b_id → Entity_B.id (FK, 실시간)
```

- 외래키로 구현, 참조 무결성 제약
- JOIN으로 최신 정보 조회
- 원본 수정 시 자동 반영

#### 붙임 (Attachment) — 고정된 사본

```
양식: 문서 A --붙임--> 문서 B의 사본
  ↓
구현: Entity_A.attached_b_snapshot (JSON/별도 테이블)
```

- 값 복사 또는 스냅샷
- 독립적 생명주기, 원본 변경 무관

#### 스냅샷 (Snapshot) — 버전 관리

```
양식: 문서 A --스냅샷--> 문서 B@시점T
  ↓
구현: Entity_A_Snapshot (버전 테이블)
```

- 시점별 조회, 이력 추적

#### 마스터-트랜잭션 패턴

```
양식: 제품명 [선택 ▼]
  ↓ "어디서 선택?" → "제품 마스터에서"
  ↓
구현: products (마스터)
       ↑ FK
     qc_requests (트랜잭션)
```

**규칙**: 선택 가능한 값 = 마스터 엔티티, 선택하는 문서 = 트랜잭션 엔티티

### 1.4 시간성 사상

#### 생명주기 → 상태 관리

```
양식: 작성중 → 제출됨 → 승인됨 → 완료됨
  ↓
구현: status ENUM('draft', 'submitted', 'approved', 'completed')
     state_changed_at TIMESTAMP
     state_changed_by UUID
```

**필수 속성**: 상태 필드, 상태 변경 시각, 상태 변경자

**기본 시점 필드**:
```
created_at    : 생성 시각        created_by  : 생성자
updated_at    : 수정 시각        updated_by  : 최종 수정자
submitted_at  : 제출 시각 (해당 시)
approved_at   : 승인 시각 (해당 시)
```

### 1.5 무결성 원리

#### 완비성 보장

```
양식: 품질검사의뢰서
     제품명 (본질적)    → NOT NULL
     LOT번호 (본질적)   → NOT NULL
     특이사항 (부가적)   → NULLABLE
     설명적 요소         → 주석/문서
```

#### 참조 무결성

| 제약 타입 | 의미 | 사용 시기 |
|-----------|------|-----------|
| CASCADE | 원본 삭제 시 함께 삭제 | 강한 의존 |
| RESTRICT | 관련 문서 있으면 삭제 불가 | 증거 보존 |
| SET NULL | 참조만 해제 | 선택적 관계 |

#### 업무 규칙

```
양식 규칙: "검사 수량은 0보다 커야 한다"
  ↓
구현: quantity INTEGER NOT NULL CHECK (quantity > 0)
```

**규칙 위치**: 단순 규칙 → DB 제약조건, 복잡 규칙 → 애플리케이션 로직, 도메인 규칙 → 도메인 레이어

### 1.6 사상 프로세스

**체계적 절차**:

| 단계 | 활동 | 산출물 |
|------|------|--------|
| 1단계 | 양식 → 엔티티 | 엔티티 목록 |
| 2단계 | 필드 → 속성 (본질적/부가적 분류, 타입 결정, 제약조건) | 속성 명세 |
| 3단계 | 관계 식별 ([선택] → FK, 참조/붙임/스냅샷 선택) | 관계 다이어그램 |
| 4단계 | 생명주기 (상태 필드, 시점 필드) | 상태 모델 |
| 5단계 | 현업 검증 (모든 필드 반영? 규칙 표현? 관계 정확?) | 검증 체크리스트 |

**반복적 개선**:
```
V1: 초기 사상 (기본 구조) → V2: 제약조건 추가 → V3: 정규화/역정규화 → V4: 구조 진화
```

### 1.7 기술 중립성

| 패러다임 | 사상 |
|----------|------|
| **관계형 DB** | 양식→테이블, 필드→컬럼, 관계→FK |
| **문서 DB** | 양식→컬렉션, 문서→JSON, 관계→임베디드 |
| **객체지향** | 양식→클래스, 문서→객체, 필드→속성 |
| **그래프 DB** | 양식→노드타입, 문서→노드, 관계→엣지 |

**불변 원리**: 기술이 바뀌어도 양식 구조는 보존, 관계 의미는 유지, 무결성은 보장, 생명주기는 추적

### 1.8 실전 예제: 품질관리 시스템

**양식 도메인**:
```
품질검사의뢰서 --참조--> 제품마스터
검사기록지 --참조--> 품질검사의뢰서
불량기록지 --참조--> 검사기록지
```

**사상 결과**:
```
Product (마스터)                 QCRecord (트랜잭션)
  ├─ id, name, code               ├─ id
                                   ├─ request_id → QCRequest
QCRequest (트랜잭션)               └─ result
  ├─ id
  ├─ product_id → Product        DefectRecord (트랜잭션)
  ├─ quantity (CHECK > 0)          ├─ id
  └─ status                        ├─ record_id → QCRecord
                                   └─ defect_type
```

**생명주기**:
```
QCRequest:    draft → submitted → inspecting → completed
QCRecord:     created → inspected → approved
DefectRecord: reported → verified → closed
```

### 1.9 품질 체크리스트

- 완전성: 모든 양식→엔티티, 모든 필드→속성, 모든 관계 명시, 모든 업무 규칙 표현
- 충실성: 양식 구조 보존, 필드 의미 유지, 관계 특성 유지, 생명주기 반영
- 실용성: 현업 이해 가능, 개발자 구현 가능, 성능 고려, 유지보수 용이
- 응답 가능성: 워크숍이 보존한 [역량질문](glossary.md#역량질문-competency-question) 목록([워크숍 산출물 3.7](workshop.md#37-역량질문-목록-2차-문서-목록))의 각 질문을 도출된 엔티티·관계로 답할 수 있는가 — 앞의 세 항목은 도출이 양식에 충실한지를 묻고, 이 항목만 도출이 **쓸모 있는지**를 묻습니다. 답하지 못하는 질문 하나가 빠진 엔티티나 관계 하나입니다.

---

## 2. 사상 규칙

### 2.1 규칙 요약 — 정본

| 규칙 | From | To | 무리 |
|------|------|-----|------|
| 규칙 1 | Main Section | **새 Entity** (주 테이블) | 정의 |
| 규칙 2 | Field | Attribute (컬럼) | — |
| 규칙 3 | Child Section | **새 Entity** + 1:N (parent FK) | 정의 |
| 규칙 4 | Reference Section | **기존 Entity로의 FK** — 지금 참 | 투영 |
| 규칙 5 | Attachment Section | **그때 값의 고정** — 문서 스냅샷이면 값 복사, 파일이면 파일 테이블 | 투영 |

> **규칙 1은 Main Section에만 적용됩니다.** 모든 섹션이 테이블이 되는 것이 아닙니다. 정의하는 섹션(Main·Child)만 새 엔티티를 만들고, 투영하는 섹션(Reference·Attachment)은 이미 있는 엔티티에서 필요한 속성만 골라 옵니다 — [방법론 「정의하는 섹션과 투영하는 섹션」](methodology.md#정의하는-섹션과-투영하는-섹션).
>
> **규칙 4·5는 관계의 시간 결합 축을 규정합니다.** 관계의 **의미 축** — 무엇 다음에 무엇이 오는가, 무엇을 근거로 쓰였는가 — 는 섹션이 아니라 [흐름](methodology.md#2-문서의-흐름)이 공급합니다.

이 표가 사상 규칙의 정본입니다. 다른 문서에 같은 표가 보이면 오류입니다.

### 2.2 설계 철학

Formology에서 데이터베이스 설계는 **양식 구조에서 자연스럽게 도출**됩니다. 사전 설계가 아니라 **발견**입니다.

| 단계 | 전통 ERD 모델링 | Formology 접근 |
|------|----------------|--------------|
| 시작 | Entity 정의 | 서식(FormType) 나열 |
| 분석 | 속성 도출 | 양식(Form) 구조 스케치 (Section/Field) |
| 관계 | Cardinality 분석 | Section 유형 관찰 (Reference/Child) |
| 검증 | 정규화 이론 | 현업 확인 |
| 오류 발견 시점 | 구현 후 (현업이 화면을 볼 때) | 설계 중 (현업이 그 자리에 있으므로) |

### 2.3 규칙별 상세

아래 소제목의 괄호는 **[2.1 정본 표](#21-규칙-요약--정본)의 규칙 번호**입니다. 이 절은 그 규칙들이 실제 스키마에서 어떤 모양이 되는지를 보이는 예시이며, 규칙 자체를 다시 정의하지 않습니다.

> SQL은 **예시**입니다. 특정 데이터베이스를 권하는 것이 아닙니다.

#### Main Section → 새 Entity, Reference Section의 Selection → FK  *(규칙 1·4)*

**워크숍 양식 구조**:
```
┌─────────────────────────┐
│ [Main Section] 의뢰정보 │
├─────────────────────────┤
│ 제품명: [Selection ▼]  │ ← Reference Section Field
│ LOT번호: [Text]         │ ← Main Section Field
│ 수량: [Numeric]         │ ← Main Section Field
└─────────────────────────┘
```

**현업 질문 → 도출**:
```
개발자: "제품명은 어디서 선택하나요?"
현업: "제품 마스터에서요"
```

**DDL 패턴**:
```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  code VARCHAR(50) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE qc_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_number VARCHAR(20) UNIQUE NOT NULL,
  product_id UUID NOT NULL,                     -- Reference Section → FK
  lot_number VARCHAR(50) NOT NULL,              -- Text Field
  quantity INTEGER NOT NULL,                    -- Numeric Field
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id)
);
```

#### Text·Numeric·Date Field → 컬럼  *(규칙 2)*

```sql
CREATE TABLE work_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  worker_name VARCHAR(50) NOT NULL,      -- Text Field
  start_time TIME NOT NULL,              -- Time Field
  end_time TIME NOT NULL,                -- Time Field
  notes TEXT,                            -- TextArea Field
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**패턴**: `Main Section Field (Text/Numeric/Date)` → VARCHAR/INTEGER/DATE 컬럼

#### Child Section → 새 Entity + 1:N  *(규칙 3)*

**워크숍 양식 구조**:
```
┌─────────────────────────┐
│ [Main Section] 의뢰정보 │
├─────────────────────────┤
│ 제품명: [Selection]     │
│ LOT번호: [Text]         │
├─────────────────────────┤
│ [Child Section] 검사항목│ ← 1:N 관계
│ ☑ 외관검사             │
│ ☑ 치수검사             │
│ ☐ 중량검사             │
└─────────────────────────┘
```

**DDL 패턴**:
```sql
CREATE TABLE qc_request_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  request_id UUID NOT NULL,                  -- Parent FK
  inspection_type_id UUID NOT NULL,          -- Reference Section Field
  is_required BOOLEAN DEFAULT false,
  FOREIGN KEY (request_id) REFERENCES qc_requests(id) ON DELETE CASCADE,
  FOREIGN KEY (inspection_type_id) REFERENCES inspection_types(id)
);
```

#### "참조:" 칸 → 최신 버전으로의 FK  *(규칙 4)*

```sql
CREATE TABLE work_standards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL,
  version VARCHAR(20) NOT NULL,
  is_latest BOOLEAN DEFAULT false,
  effective_date DATE NOT NULL,
  supersedes_id UUID,                -- 이전 버전 참조
  FOREIGN KEY (supersedes_id) REFERENCES work_standards(id),
  UNIQUE (product_id, version)
);

CREATE VIEW latest_work_standards AS
SELECT * FROM work_standards WHERE is_latest = true;
```

#### "붙임:" 칸 → 파일 테이블  *(규칙 5 — 파일인 경우)*

```sql
CREATE TABLE defect_attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  record_id UUID NOT NULL,
  file_name VARCHAR(255) NOT NULL,
  file_path VARCHAR(500) NOT NULL,
  file_size INTEGER,
  mime_type VARCHAR(100),
  uploaded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (record_id) REFERENCES defect_records(id) ON DELETE CASCADE
);
```

**붙임이 문서 스냅샷인 경우**는 파일이 아니라 **값 복사**가 됩니다 — 원본 문서의 그때 값이 이 문서의 칸으로 굳습니다. 두 경우 모두 규칙 5이며, 갈리는 지점은 붙이는 것이 파일이냐 다른 문서의 내용이냐입니다. 구체적인 모습은 [2.5](#25-관계와-시간-결합의-구현)에 있습니다.

### 2.4 Field 타입 사상

| Field 타입 | DB 컬럼 타입 | 예시 |
|-----------|-------------|------|
| Text | VARCHAR/TEXT | `document_no VARCHAR(50)` |
| Numeric | INTEGER/DECIMAL | `quantity INTEGER` |
| Date/Time | DATE/TIMESTAMP | `created_at TIMESTAMP` |
| Selection (Single) | FK | `product_id FK` |
| Selection (Multiple) | 1:N 테이블 | `qc_items` |
| Boolean | BOOLEAN | `is_urgent BOOLEAN` |

### 2.5 관계와 시간 결합의 구현

#### Reference Section 관계 (실시간 링크)

```sql
-- 작업지시서 1:N 작업일지 (Reference Section)
CREATE TABLE work_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  work_order_id UUID NOT NULL,          -- Reference Section → Parent Document FK
  log_date DATE NOT NULL,
  FOREIGN KEY (work_order_id) REFERENCES work_orders(id) ON DELETE CASCADE
);
```

#### Attachment Section 관계 (스냅샷)

```sql
-- 작업일지에 작업지시서 데이터를 스냅샷 복사 (과거 시점 고정)
CREATE TABLE work_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  work_order_id UUID NOT NULL,          -- Reference Section (living link)
  product_name VARCHAR(100) NOT NULL,   -- 스냅샷: 그 시점의 제품명 (frozen)
  product_code VARCHAR(50) NOT NULL,    -- 스냅샷: 그 시점의 제품 코드 (frozen)
  target_quantity INTEGER NOT NULL,     -- 스냅샷: 그 시점의 목표 수량 (frozen)
  log_date DATE NOT NULL,
  FOREIGN KEY (work_order_id) REFERENCES work_orders(id)
);
```

**한 테이블 안에 두 바인딩이 공존합니다.** `work_order_id`는 참조이고 `product_name`·`target_quantity`는 붙임입니다. 이것이 [필드 단위 세분](methodology.md#필드-단위-세분--섹션이-기본값을-정하고-필드가-예외를-선언한다)이 말하는 상태이며, 이론이 뒤늦게 따라간 자리이기도 합니다.

**붙임 값을 언제 굳히는가**는 도출이 답하는 문제이고, **어떻게 굳히는가**는 구현이 답하는 문제입니다.

- 도출이 정하는 것: *문서가 생성되는 순간의 원본 값을 복사한다*
- 구현이 정하는 것: DB 트리거로 할지, 애플리케이션 레이어에서 할지, 이벤트 핸들러로 할지

방법론이 요구하는 것은 하나뿐입니다 — **그 시점 이후 원본이 바뀌어도 이 값은 바뀌지 않을 것.**

> 여기서 스키마의 한계가 드러납니다. `product_name VARCHAR(100)`만 보고는 이것이 **붙임인지, 성능을 위한 비정규화인지, 그냥 실수인지 구별되지 않습니다.** 위 주석을 지우면 의미가 사라집니다. 서식은 답합니다 — **기재란과 붙임란은 다른 칸이니까요.**

### 2.6 실전 예시

**Form 구조**:
```
┌─────────────────────────┐
│ 품질검사의뢰서            │
├─────────────────────────┤
│ [Main Section]          │
│ 제품명: [선택 ▼]        │  ← Selection → FK
│ LOT번호: [입력]         │  ← Text → VARCHAR
│ 수량: [숫자]            │  ← Numeric → INTEGER
├─────────────────────────┤
│ [Child Section]         │
│ 검사항목:               │  ← Multiple → 1:N 테이블
│ ☑ 외관  ☑ 치수  □ 성능 │
├─────────────────────────┤
│ [Reference Section]     │
│ 참조: 생산계획서 PLAN-045│ ← FK (살아있는 링크)
├─────────────────────────┤
│ [Attachment Section]    │
│ 붙임: 도면.pdf          │  ← 파일 (스냅샷)
└─────────────────────────┘
```

**도출되는 Record 구조**:
```sql
-- Main Section → 메인 Entity
CREATE TABLE qc_requests (
  id VARCHAR(50) PRIMARY KEY,
  document_no VARCHAR(50),
  product_id VARCHAR(50),        -- Selection → FK
  lot_number VARCHAR(50),        -- Text → VARCHAR
  quantity INTEGER,              -- Numeric → INTEGER
  production_plan_id VARCHAR(50), -- Reference → FK
  FOREIGN KEY (product_id) REFERENCES products(id),
  FOREIGN KEY (production_plan_id) REFERENCES production_plans(id)
);

-- Child Section → 자식 Entity (1:N)
CREATE TABLE qc_request_items (
  id VARCHAR(50) PRIMARY KEY,
  request_id VARCHAR(50) NOT NULL,
  item_name VARCHAR(100),
  is_checked BOOLEAN,
  FOREIGN KEY (request_id) REFERENCES qc_requests(id)
);

-- Attachment Section → 파일 Entity (스냅샷)
CREATE TABLE qc_request_attachments (
  id VARCHAR(50) PRIMARY KEY,
  request_id VARCHAR(50) NOT NULL,
  file_name VARCHAR(200),
  file_path VARCHAR(500),
  FOREIGN KEY (request_id) REFERENCES qc_requests(id)
);
```

### 2.7 정규화는 따라온다

Section-Field 구조가 명확하면 정규화는 자연스럽게 따라옵니다:

- **1NF**: Child Section 발견 → 별도 Entity로 분리 → 원자성 자동 달성
- **2NF**: Selection Field 발견 → FK로 마스터 참조 → 부분 종속 제거
- **3NF**: Reference Section 발견 → FK로 연결 → 이행 종속 제거

### 2.8 전통 방식과의 차이

| | 전통 데이터 모델링 | Formology |
|---|---|---|
| 과정 | Entity → Attribute → 관계 → 검증 | Form → Section/Field → Entity/Attribute → Record |
| 접근 | Top-down (추상화부터) | Bottom-up (구체적 Form부터) |
| 검증 주체 | 설계자 (정규화 이론) | 현업 (자기 서식) |
| 산출물 | ERD — 현업이 읽지 못함 | 양식 스케치 — 현업이 그림 |

**실측된 것 하나**: 플라스틱 사출 공장 MES 프로젝트에서 **2시간 15분** 만에 22개 서식이 도출되고 4개 주요 양식의 구조가 스케치되었습니다. 참석자는 현업 5명과 개발 2명이었고, 도출된 서식 목록에 **참석자 전원이 그 자리에서 동의**했습니다. 상세는 [사례 연구](casestudy.md)를 보십시오.

> 전통 방식과의 소요 시간 비교는 **의도적으로 싣지 않았습니다.** 대조군을 측정한 적이 없기 때문입니다. 측정하지 않은 숫자로 이기는 것보다, 측정한 숫자 하나를 정확히 제시하는 편이 낫습니다.

### 2.9 검증

**Form 기반 현업 검증**:
```
개발자: "품질검사의뢰서에 제품 정보 Section 있죠?"
현업: "네, 제품명이랑 LOT번호요"
개발자: "제품은 선택 방식이죠?"
현업: "네, 목록에서 선택해요"
→ Section/Field 용어로 자연스러운 검증 완료
```

**ERD 역검증** (Record → Form):
- 테이블 → Section
- FK → Selection Field 또는 Reference Section
- 일반 컬럼 → Field
- 1:N 테이블 → Child Section 또는 Attachment Section

### 2.10 워크숍에서 스키마까지

```
1. 워크숍 → 문서 양식 스케치
2. 패턴 인식 → [선택]=FK, [입력]=컬럼, 체크박스=1:N, 참조=FK(최신), 붙임=파일 테이블
3. 관계 도출 → A→B=부모-자식, "그때"=스냅샷
4. 접미어 패턴 → -서=승인, -지=이력, -표=집계 뷰, -록=시계열 파티션
5. DDL 생성 → CREATE TABLE/INDEX/TRIGGER/VIEW
6. 현업 검증 → "이 테이블이 OO 문서죠?" "맞아요!"

```

> 6단계의 검증 문답이 이 절차의 핵심입니다. 전통 ERD 방식에서 이 문답은 성립하지 않습니다 — 현업에게 "이 테이블이 OO 문서죠?"라고 물을 수 없기 때문입니다. 테이블은 현업의 것이 아닙니다.

### 2.11 저장 경계와 서비스 경계

**모든 것을 한 곳에 넣지 않습니다.**

구조화된 것 — 메타 정보, 필드 값, 관계 — 은 데이터베이스로. 비구조화된 것 — 이미지, 첨부 파일 원본 — 은 파일 저장소로 가고 경로만 남깁니다. 이 경계는 서식이 이미 그어 놓았습니다. **기재란은 값이고 붙임란은 파일입니다.**

**서비스 경계도 마찬가지입니다.** 서식은 그대로 Bounded Context가 됩니다.

```
품질검사 서비스 — 품질검사의뢰서, 검사기록지, 합격확인서
생산 서비스   — 작업지시서, 작업일지
자재 서비스   — 자재요청서, 입고확인서
```

서비스 간 통신은 **문서 ID 참조**와 **문서 발행 이벤트**로 이루어집니다. 별도의 컨텍스트 매핑을 설계할 필요가 없습니다 — 어느 서식이 어느 업무에 속하는지는 현업이 워크숍 [Step 4 그룹핑](workshop.md#step-4-그룹핑-20분)에서 이미 나눠 놓았습니다.

> 여기까지가 원리입니다. **어떤 데이터베이스를, 어떤 스토리지를, 어떤 서비스 런타임을 쓸지는 이 문서가 답하지 않습니다.** 구현체의 몫입니다.

---

---

## 3. API로의 사상

### 3.1 핵심 원칙

> 아래는 **REST를 예로 든 것**입니다. 원리는 *서식 접미어가 동작 패턴을 결정한다*이며, 이는 RPC든 메시지 기반이든 동일하게 성립합니다. 프로토콜 선택은 구현체의 몫입니다.

```
서식(FormType)     = API 리소스 (Resource)
양식(Form) 구조    = API 스키마 (Schema)
문서(Document)     = 리소스 인스턴스 (Resource Instance)
문서 흐름          = API 엔드포인트 조합
서식 접미사        = API 동작 패턴
```

### 3.2 REST 리소스 매핑

#### 서식명 → 리소스 경로

```
"작업지시서"        → /api/v1/work-orders
"품질검사보고서"    → /api/v1/quality-reports
"설비점검일지"      → /api/v1/equipment-inspections
```

#### 접미사별 엔드포인트 패턴

**-서 (書, 요청/의뢰)**:
```yaml
POST   /resource          # 신규 작성
PUT    /resource/:id      # 수정
POST   /resource/:id/submit   # 제출
POST   /resource/:id/approve  # 승인
POST   /resource/:id/reject   # 반려
GET    /resource/:id      # 조회
GET    /resource          # 목록
```

**-지 (紙, 기록)**:
```yaml
POST   /resource          # 신규 기록
PUT    /resource/:id      # 수정
GET    /resource/:id/history  # 이력 조회
GET    /resource/:id      # 조회
GET    /resource          # 목록
```

**-표 (表, 집계)**:
```yaml
GET    /resource          # 집계 결과 조회
GET    /resource/export   # Excel/PDF 다운로드
```

**-록 (錄)** — [1차와 2차가 섞이는 자리](methodology.md#분류의-세-축)입니다.

```yaml
# 1차 (회의록·대장처럼 사람이 쓰는 것)
POST   /resource          # 신규 등록
PUT    /resource/:id      # 수정
GET    /resource/:id      # 조회
GET    /resource          # 목록 (필터링, 페이징)

# 2차 (이력록처럼 축적에서 계산되는 것)
GET    /resource          # 조회만. append-only이므로 수정·삭제 없음
```

---

## 4. 인터페이스로의 사상

### 4.1 핵심 원칙

> **양식의 구조 = 인터페이스의 구조**

종이 양식처럼 보이게 만들면, 사용자가 즉시 이해합니다.

### 4.2 필드 → UI 요소 사상

#### 기본 매핑

| 양식 필드 | UI 요소 | 예시 |
|-----------|---------|------|
| [입력] | Text Input | 이름, 주소 |
| [숫자] | Number Input | 수량, 금액 |
| [선택] | Dropdown | 제품명, 상태 |
| [날짜] | Date Picker | 의뢰일, 완료일 |
| [체크박스] | Checkbox | 긴급, 승인 |
| [텍스트] | Textarea | 특이사항, 비고 |
| [파일] | File Upload | 첨부파일 |

#### 요소 유형별 강조

**본질적 필드 (필수)**: 붉은 * 표시, 명확한 레이블
```
제품명* [드롭다운 ▼]
```

**부가적 필드 (선택)**: 밝은 색, 선택적 표시
```
특이사항 [텍스트]
```

**설명적 요소**: 도움말 아이콘, 클릭 시 형식 안내
```
LOT번호 [입력] (?) → "형식: YY-MM-###"
```

### 4.3 레이아웃 원리

#### 정보 계층

```
문서                    ← 제목 (H1, 크게)
├─ 섹션 A               ← 부제목 (H2, 중간)
│  ├─ 필드 1            ← 입력 (작게)
│  └─ 필드 2
└─ 섹션 B
   ├─ 필드 3
   └─ 필드 4
```

#### 그룹화 (근접성)

관련 필드는 가까이, 다른 그룹은 공백으로 분리:
```
┌──────────────┐
│ 작업자: [  ] │ ← 같은 그룹
│ 날짜: [  ]   │
└──────────────┘
    [공백]
┌──────────────┐
│ 제품: [  ]   │ ← 다른 그룹
└──────────────┘
```

#### 반응형

```
데스크톱: 필드1 [   ]  필드2 [   ]  ← 2열
모바일:   필드1 [   ]               ← 1열
          필드2 [   ]
```

### 4.4 상호작용 패턴

**자동완성**: 입력 시 필터링 → [제품A], [제품AA]...

**의존 선택**: 카테고리 선택 → 제품 목록 동적 업데이트

**지능형 기본값**: 작성자=[현재 사용자], 날짜=[오늘]

**즉시 검증**:
```
LOT번호: [ABC-123] → 올바른 형식 (실시간 확인)
수량: [-5] → 0보다 커야 합니다 (즉시 오류)
```

**상태 표시**:
```
[작성] → [제출] → [승인] → [완료]
 ●────────○────────○────────○
```

### 4.5 관계 시각화

**참조 (Reference)**: 링크 아이콘, 클릭 시 원본 이동, 실시간 정보 표시
```
제품명: [제품A ▼] → 제품 정보: 코드 PRD-001, 규격 100x200mm
```

**붙임 (Attachment)**: 클립 아이콘, "붙임" 배지, 다운로드/미리보기

**스냅샷 (Snapshot)**: 카메라 아이콘, 타임스탬프, 차이 보기 기능

### 4.6 오류 처리

**나쁜 예**: "입력 오류", "잘못된 값"

**좋은 예**: "LOT번호는 'YY-MM-###' 형식이어야 합니다", "수량은 1 이상이어야 합니다. 현재: -5"

**위치**: 필드 바로 아래에 오류 메시지 표시

**성공 피드백**: "저장되었습니다" (2초 후 사라짐), "제출 완료. 의뢰번호: QC-2024-001"

### 4.7 실전 예제: 품질검사의뢰서 레이아웃

```
┌────────────────────────────────────┐
│  품질검사의뢰서 작성               │ H1
├────────────────────────────────────┤
│  기본 정보                         │ H2
│  ┌──────────────────────────────┐ │
│  │ 의뢰번호*: [자동생성]        │ │ 읽기전용
│  │ 제품명*: [선택 ▼]           │ │ 필수
│  │ LOT번호*: [YY-MM-###]        │ │ 필수
│  │ 수량*: [0]                   │ │ 필수
│  └──────────────────────────────┘ │
│                                    │
│  추가 정보                         │ H2
│  ┌──────────────────────────────┐ │
│  │ 특이사항: [         ]        │ │ 선택
│  │ 첨부파일: [파일 선택]        │ │ 선택
│  └──────────────────────────────┘ │
│                                    │
│  [저장] [제출]                     │
└────────────────────────────────────┘
```

**상호작용 흐름**:
1. 페이지 로드 → 의뢰번호 자동생성
2. 제품 선택 → 상세정보 표시
3. 입력 중 → 실시간 검증
4. 제출 → 최종 확인 → 완료 메시지

---

## 5. 서식 템플릿

### 5.1 문서 양식 템플릿

#### 요청서 (-서) 양식

```
# {문서명} (-서)

## 기본 정보
- 문서번호: [자동생성]
- 작성일/작성자: [자동입력]
- 상태: 임시저장 / 제출 / 승인 / 반려

## 요청 내용
- {항목명}: [타입] (필수/선택)

## 첨부 파일
- 붙임: {파일명} (최대 {N}개, {N}MB 이하)

## 승인 정보
- 제출일시, 승인자, 승인일시, 승인의견
```

**Excel 템플릿 예시** (작업지시서):

| 항목 | 타입 | 필수 | 예시 |
|------|------|------|------|
| 지시서번호 | 자동생성 | O | WO20241105001 |
| 제품 | 선택 | O | 제품A |
| 설비 | 선택 | O | 설비1호기 |
| 목표수량 | 직접입력 | O | 1000 |
| 마감일 | 날짜선택 | O | 2024-11-20 |
| 우선순위 | 선택 | O | 높음/보통/낮음 |
| 비고 | 직접입력 | | 긴급 생산 |
| 상태 | 자동관리 | O | 임시저장/제출/승인/반려 |

#### 기록지 (-지) 양식

```
# {문서명} (-지)

## 기본 정보
- 문서번호: [자동생성], 기록일, 기록자

## 기록 내용
- 참조: [선택] (최신 버전 자동 연결)
- {기록 항목}: [측정값], [체크리스트]
- 특이사항: [텍스트]

## 이력 추적 (자동 기록)
| 버전 | 일시 | 작업자 | 변경 내용 |
```

#### 현황표 (-표) 양식

```
# {문서명} (-표)

## 조회 조건
- 기준일, 기간, 구분

## 요약
| 지표 | 값 | 단위 |

## 상세 현황
| {분류} | {지표1} | {지표2} | {비율} |

## 내보내기: Excel / PDF / CSV
```

#### 대장 (-록) 양식

```
# {문서명} (-록)

## 검색 조건
- 검색어, 분류, 상태, 기간

## 목록
| 번호 | {항목} | 등록일 | 상태 | 작업 |

## 페이징: [이전] 1 2 3 [다음]  총 {N}건

## 작업: [신규 등록] [일괄 다운로드] [Excel 업로드]
```

---

> **"사상은 변환이 아닌 발견이다"**
>
> **"기술은 바뀌어도 원리는 불변이다"**

**Formology** — *Workflow First, Ontology Follows*
