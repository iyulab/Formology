# 구현 원리

> **"양식의 구조는 구현 구조로 자연스럽게 변환됩니다"**

이 문서는 Formology 방법론을 시스템으로 실현하기 위한 **기술 중립적 원리**를 통합 정리한 문서입니다. 개념적 사상, 데이터베이스 설계, API 설계, 인터페이스 설계, 실전 템플릿, 사례 연구, FAQ, 용어 사전을 포함합니다.

**핵심 원칙**:
1. **기술 중립성** — 특정 기술에 종속되지 않는 보편적 원리
2. **구조 보존** — 양식 구조 ≅ 구현 구조 (동형 사상)
3. **검증 가능성** — 현업이 구현 구조를 확인·검증 가능
4. **점진적 구체화** — 보편적 원리 → 기술별 적용 → 프로젝트별 구현

---

## 목차

1. [개념적 사상](#1-개념적-사상) — 양식에서 시스템으로의 매핑 원리
2. [데이터베이스 설계](#2-데이터베이스-설계) — 5대 사상 규칙, DDL 패턴
3. [API 설계](#3-api-설계) — REST 매핑, 엔드포인트 패턴
4. [인터페이스 설계](#4-인터페이스-설계) — 양식에서 UI로, 접근성
5. [템플릿 모음](#5-템플릿-모음) — 워크숍/DB/API 템플릿
6. [사례 연구](#6-사례-연구) — 제조 MES (플라스틱 사출 공장, 2023)
7. [FAQ](#7-faq)
8. [용어 사전](#8-용어-사전)

---

## 1. 개념적 사상

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

---

## 2. 데이터베이스 설계

### 2.1 설계 철학

Formology에서 데이터베이스 설계는 **양식 구조에서 자연스럽게 도출**됩니다. 사전 설계가 아니라 **발견**입니다.

| 단계 | 전통 ERD 모델링 | Formology 접근 |
|------|----------------|--------------|
| 시작 | Entity 정의 | 서식(FormType) 나열 |
| 분석 | 속성 도출 | 양식(Form) 구조 스케치 (Section/Field) |
| 관계 | Cardinality 분석 | Section 유형 관찰 (Reference/Child) |
| 검증 | 정규화 이론 | 현업 확인 |
| 오류 발견 시점 | 구현 후 (현업이 화면을 볼 때) | 설계 중 (현업이 그 자리에 있으므로) |

### 2.2 5대 사상 규칙

#### 규칙 1: Selection Field in Reference Section = FK

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

#### 규칙 2: Text/Numeric/Date Fields = 컬럼

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

#### 규칙 3: Child Section = 1:N 테이블

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

#### 규칙 4: "참조:" = FK (최신 버전)

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

#### 규칙 5: "붙임:" = 1:N (파일)

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

### 2.3 문서 접미사 → 테이블 특성 패턴

#### -서(書): 공식 문서 = 승인 워크플로우

```sql
-- 승인 워크플로우 필수 필드
status VARCHAR(20) DEFAULT 'DRAFT',   -- DRAFT, SUBMITTED, APPROVED, REJECTED
submitted_at TIMESTAMP,
submitted_by UUID,
approved_at TIMESTAMP,
approved_by UUID,
approval_comments TEXT,
is_official BOOLEAN DEFAULT true,
signature_required BOOLEAN DEFAULT true
```

#### -지(紙): 단순 기록 = 이력 관리

```sql
-- 이력 추적 필수 필드
history JSONB DEFAULT '[]'::jsonb,   -- 변경 이력
version INTEGER DEFAULT 1,
status VARCHAR(20) DEFAULT 'ACTIVE',  -- ACTIVE, ARCHIVED
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
updated_by UUID
```

이력 자동 기록 트리거:
```sql
CREATE OR REPLACE FUNCTION track_record_history()
RETURNS TRIGGER AS $$
BEGIN
  NEW.history = NEW.history || jsonb_build_object(
    'version', NEW.version,
    'updated_at', CURRENT_TIMESTAMP,
    'changes', jsonb_build_object('old', to_jsonb(OLD), 'new', to_jsonb(NEW))
  );
  NEW.version = NEW.version + 1;
  NEW.updated_at = CURRENT_TIMESTAMP;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### -표(表): 집계/통계 = 집계 뷰

```sql
-- 실제 테이블이 아닌 뷰로 구현
CREATE VIEW monthly_production_summary AS
SELECT
  DATE_TRUNC('month', log_date) as month,
  product_id,
  SUM(good_quantity) as total_good,
  SUM(defect_quantity) as total_defect,
  ROUND(100.0 * SUM(good_quantity) / NULLIF(SUM(good_quantity + defect_quantity), 0), 2) as yield_rate
FROM work_logs wl
JOIN work_orders wo ON wl.work_order_id = wo.id
GROUP BY DATE_TRUNC('month', log_date), product_id;

-- 필요시 Materialized View로 성능 향상
CREATE MATERIALIZED VIEW monthly_production_summary_mat AS
SELECT * FROM monthly_production_summary;
```

#### -록(錄): 이력 = 시계열 테이블

```sql
CREATE TABLE defect_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  occurred_at TIMESTAMP NOT NULL,
  recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  product_id UUID NOT NULL,
  defect_type_id UUID NOT NULL,
  quantity INTEGER NOT NULL,
  search_text TSVECTOR
) PARTITION BY RANGE (occurred_at);

-- 시계열 인덱스 + 파티셔닝 + 전문검색
CREATE INDEX idx_defect_history_occurred ON defect_history(occurred_at DESC);
CREATE INDEX idx_defect_history_search ON defect_history USING GIN(search_text);
```

### 2.4 Document 관계 → FK/계층

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

스냅샷 자동 복사 트리거:
```sql
CREATE OR REPLACE FUNCTION snapshot_work_order_data()
RETURNS TRIGGER AS $$
BEGIN
  SELECT p.name, p.code, wo.quantity
  INTO NEW.product_name, NEW.product_code, NEW.target_quantity
  FROM work_orders wo
  JOIN products p ON wo.product_id = p.id
  WHERE wo.id = NEW.work_order_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_snapshot_before_insert
BEFORE INSERT ON work_logs
FOR EACH ROW EXECUTE FUNCTION snapshot_work_order_data();
```

### 2.5 성능 최적화

#### 인덱스 전략

```sql
-- FK 인덱스
CREATE INDEX idx_work_logs_order ON work_logs(work_order_id);
-- 검색 인덱스
CREATE INDEX idx_work_orders_status_date ON work_orders(status, due_date);
-- 부분 인덱스 (활성 데이터만)
CREATE INDEX idx_active_work_orders ON work_orders(due_date) WHERE status IN ('PENDING', 'IN_PROGRESS');
-- 전문 검색 인덱스
CREATE INDEX idx_work_logs_notes_search ON work_logs USING GIN(to_tsvector('korean', notes));
```

#### 파티셔닝 전략

```sql
-- 시계열 데이터는 월별 파티셔닝
CREATE TABLE defect_history (...) PARTITION BY RANGE (occurred_at);

-- 자동 파티션 생성 함수
CREATE OR REPLACE FUNCTION create_monthly_partition(base_table TEXT, partition_date DATE)
RETURNS VOID AS $$
DECLARE
  partition_name TEXT;
  start_date DATE;
  end_date DATE;
BEGIN
  partition_name = base_table || '_' || TO_CHAR(partition_date, 'YYYY_MM');
  start_date = DATE_TRUNC('month', partition_date);
  end_date = start_date + INTERVAL '1 month';
  EXECUTE format('CREATE TABLE IF NOT EXISTS %I PARTITION OF %I
    FOR VALUES FROM (%L) TO (%L)', partition_name, base_table, start_date, end_date);
END;
$$ LANGUAGE plpgsql;
```

### 2.6 마이그레이션 전략

```sql
-- 스키마 버전 테이블
CREATE TABLE schema_versions (
  version VARCHAR(20) PRIMARY KEY,
  description TEXT,
  applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**점진적 적용**:
```
Phase 1: 작업지시서 + 작업일지 → work_orders, work_logs
Phase 2: 품질검사 모듈 추가 → qc_requests, inspection_records
Phase 3: 통계 및 이력 → 뷰, 파티션, 이력 테이블
```

### 2.7 워크숍 → DDL 변환 프로세스 요약

```
1. 워크숍 → 문서 양식 스케치
2. 패턴 인식 → [선택]=FK, [입력]=컬럼, 체크박스=1:N, 참조=FK(최신), 붙임=파일 테이블
3. 관계 도출 → A→B=부모-자식, "그때"=스냅샷
4. 접미어 패턴 → -서=승인, -지=이력, -표=집계 뷰, -록=시계열 파티션
5. DDL 생성 → CREATE TABLE/INDEX/TRIGGER/VIEW
6. 현업 검증 → "이 테이블이 OO 문서죠?" "맞아요!"

```

> 6단계의 검증 문답이 이 절차의 핵심입니다. 전통 ERD 방식에서 이 문답은 성립하지 않습니다 — 현업에게 "이 테이블이 OO 문서죠?"라고 물을 수 없기 때문입니다. 테이블은 현업의 것이 아닙니다.

---

## 3. API 설계

### 3.1 핵심 원칙

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

**-록 (錄, 이력)**:
```yaml
POST   /resource          # 신규 등록
PUT    /resource/:id      # 수정
DELETE /resource/:id      # 삭제
GET    /resource/:id      # 조회
GET    /resource          # 목록 (필터링, 페이징)
```

### 3.3 실전 예제: 작업지시서 API

**양식 구조 → API 스키마**:
```
[Main Section] 지시정보
- 제품명 [Reference Section Field] → productId (FK)
- 수량 [Numeric Field] → targetQuantity
- 납기일 [Date Field] → dueDate
```

**엔드포인트 설계**:

```
POST /api/v1/work-orders
Request: { productId, machineId, targetQuantity, dueDate, priority, notes }
Response: 201 Created { id, orderNumber, status: "DRAFT", ... }

POST /api/v1/work-orders/{id}/submit
Response: 200 OK { status: "SUBMITTED", submittedAt, submittedBy }

POST /api/v1/work-orders/{id}/approve
Request: { comment }
Response: 200 OK { status: "APPROVED", approvedAt, approvedBy }

GET /api/v1/work-orders?status=APPROVED&dueDate=2024-11-20&page=1&limit=20
Response: 200 OK { data: [...], pagination: { page, limit, total, totalPages } }
```

### 3.4 집계 API (-표 패턴)

```
GET /api/v1/production-status/daily?date=2024-11-05
Response: 200 OK {
  date, summary: { totalOrders, completedOrders, totalProduced, defectRate },
  byMachine: [...], byProduct: [...]
}

GET /api/v1/production-status/daily/export?date=2024-11-05&format=xlsx
Response: 200 OK (Binary Excel Data)
```

### 3.5 파일 첨부 처리

```
POST /api/v1/work-orders/{id}/attachments
Content-Type: multipart/form-data
→ 201 Created { id, filename, mimeType, size, url }

GET /api/v1/attachments/{id}/download
→ 200 OK (Binary File Data)

GET /api/v1/work-orders/{id}/attachments
→ 200 OK { attachments: [...] }
```

### 3.6 인증 및 인가

#### JWT 기반 인증

```
POST /api/v1/auth/login
→ { accessToken, refreshToken, expiresIn, user: { id, name, role } }

POST /api/v1/auth/refresh
→ { accessToken, expiresIn }
```

#### 역할 기반 접근 제어 (RBAC)

```yaml
작업지시서 (-서):
  DRAFT: 작성자만 수정/삭제
  SUBMITTED: 승인자만 승인/반려
  APPROVED: 모두 읽기 전용

품질검사보고서 (-지):
  검사자: 작성/수정
  품질관리자: 모든 작업
  일반 사용자: 읽기 전용

생산현황표 (-표):
  모두: 읽기 전용
  관리자: Excel 다운로드
```

### 3.7 GraphQL 지원

#### 타입 정의

```graphql
type WorkOrder {
  id: ID!
  orderNumber: String!
  product: Product!
  targetQuantity: Int!
  status: WorkOrderStatus!
  submittedAt: DateTime
  approvedAt: DateTime
  attachments: [Attachment!]!
}

type QualityReport {
  id: ID!
  reportNumber: String!
  items: [InspectionItem!]!     # 1:N 관계
  overallResult: InspectionResult!
  history: [AuditLog!]!         # -지 패턴: 이력
}

type ProductionStatus {         # -표 패턴: Read-Only
  date: Date!
  summary: ProductionSummary!
  byMachine: [MachineProduction!]!
}
```

#### Query/Mutation

```graphql
type Query {
  workOrder(id: ID!): WorkOrder
  workOrders(status: WorkOrderStatus, page: Int, limit: Int): WorkOrderConnection!
  productionStatus(date: Date!): ProductionStatus!          # -표: Read-Only
}

type Mutation {
  createWorkOrder(input: CreateWorkOrderInput!): WorkOrder!
  submitWorkOrder(id: ID!): WorkOrder!
  approveWorkOrder(id: ID!, comment: String): WorkOrder!
}

type Subscription {
  workOrderStatusChanged(id: ID!): WorkOrder!
  productionStatusUpdated(date: Date!): ProductionStatus!
}
```

### 3.8 고급 패턴

**버전 관리**:
```
GET /api/v1/quality-reports/{id}/versions
GET /api/v1/quality-reports/{id}/versions/{version}
```

**Batch Operations**:
```
POST /api/v1/work-orders/batch
{ operations: [{ action: "approve", id, comment }, ...] }
```

### 3.9 배포 고려사항

```yaml
API 버전 관리: /api/v1/, /api/v2/ (또는 헤더 기반)
캐싱: Redis (집계 데이터), HTTP 캐시 헤더 (ETag, Cache-Control)
보안: JWT + refresh token, RBAC, Schema validation, SQL injection 방지
모니터링: Structured logging, Response time/Error rate 메트릭, OpenTelemetry 추적
GraphQL: DataLoader (N+1 해결), Query complexity 제한, Persisted queries
```

### 3.10 API 설계 체크리스트

- 문서 → 리소스 매핑 완료
- 접미사별 엔드포인트 패턴 적용
- 인증/인가 정책 정의
- 파일 첨부 처리 구현
- 버전 관리 전략 수립
- 에러 핸들링 표준화
- OpenAPI 명세 작성
- API 테스트 작성

---

## 4. 인터페이스 설계

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

### 4.5 접근성 (WCAG 준수)

**시각**: 색 대비 4.5:1 이상, 텍스트 크기 조절 가능, 아이콘 + 텍스트 병행

**키보드**: 모든 기능 키보드 접근, Tab 순서 논리적, Esc로 모달 닫기

**명확성**: 명확한 레이블, 구체적 오류 메시지, 충분한 도움말

**시맨틱 HTML**:
```html
<form>
  <label for="product">제품명*</label>
  <select id="product" required>...</select>
</form>
```

### 4.6 관계 시각화

**참조 (Reference)**: 링크 아이콘, 클릭 시 원본 이동, 실시간 정보 표시
```
제품명: [제품A ▼] → 제품 정보: 코드 PRD-001, 규격 100x200mm
```

**붙임 (Attachment)**: 클립 아이콘, "붙임" 배지, 다운로드/미리보기

**스냅샷 (Snapshot)**: 카메라 아이콘, 타임스탬프, 차이 보기 기능

### 4.7 오류 처리

**나쁜 예**: "입력 오류", "잘못된 값"

**좋은 예**: "LOT번호는 'YY-MM-###' 형식이어야 합니다", "수량은 1 이상이어야 합니다. 현재: -5"

**위치**: 필드 바로 아래에 오류 메시지 표시

**성공 피드백**: "저장되었습니다" (2초 후 사라짐), "제출 완료. 의뢰번호: QC-2024-001"

### 4.8 모바일 최적화

- 최소 터치 타겟: 44x44px, 버튼 간 간격: 8px 이상
- 적절한 키보드: 숫자 필드→숫자 키패드, 이메일→이메일 키보드
- 단계별 진행: 1단계 기본 정보 → 2단계 상세 정보 → 3단계 검토 및 제출

### 4.9 실전 예제: 품질검사의뢰서 레이아웃

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

### 4.10 설계 체크리스트

- **구조**: 양식 레이아웃 반영, 정보 계층 명확, 관련 필드 그룹화
- **상호작용**: 입력 방식 직관적, 피드백 즉시 제공, 오류 메시지 친절
- **접근성**: 키보드 전체 기능, 스크린리더 호환, 색 대비 충분
- **모바일**: 터치 타겟 크기 적절, 키보드 타입 올바름, 단계별 진행 가능

---

## 5. 템플릿 모음

### 5.1 워크숍 준비 템플릿

#### 초대 이메일

```
제목: [Formology 워크숍] {시스템명} 문서 정의 세션 안내

안녕하세요,

{시스템명} 개발을 위한 Formology 워크숍을 아래와 같이 진행합니다.

■ 일시: {YYYY.MM.DD} ({요일}) {HH:MM}~{HH:MM} ({N}시간)
■ 장소: {장소명}
■ 참석자: {팀명} {N}명, 개발팀 {N}명
■ 목적: 업무 문서 목록 도출 및 양식 정의

■ 준비사항:
- 현재 사용 중인 문서 양식 (종이, Excel, PDF 등) 가져오기
- 업무 흐름에 대한 이해
- 특별한 IT 지식은 불필요합니다

■ 진행 방식:
1. 실제 사용하는 "문서"를 나열합니다
2. 문서의 양식을 간단히 스케치합니다
3. 문서 간 흐름을 정리합니다

※ Entity, Domain Model 등 어려운 용어는 사용하지 않습니다
※ 여러분의 업무 용어 그대로 사용합니다
```

#### 워크숍 체크리스트

**1주 전**:
- 참석자 선정 (현업 3~5명, 개발 1~2명)
- 회의실 예약 (6~8인용, 3시간 30분)
- 준비물 구매 (포스트잇 5색 각 100장, 마커펜 5색, A4 30장)
- 초대 메일 발송 및 참석 확인

**1일 전**: 참석자 리마인더, 준비물 점검, 진행 시나리오 리허설

**당일 30분 전**: 회의실 환기, 화이트보드 지우기, 포스트잇/마커 배치, 간식/음료 준비, 카메라 준비

**종료 직후**: 화이트보드 사진 촬영 (최소 5장), 포스트잇 정리 (버리지 말 것), 감사 메일, 디지털화 시작

#### 워크숍 타임테이블

| 시간 | 단계 | 활동 | 산출물 |
|------|------|------|--------|
| 15분 | 오프닝 | 소개, 규칙 설명 | 참여자 이해 |
| 15분 | 업무 선정 | 대상 업무 범위 | 업무 범위 |
| 30분 | 브레인스토밍 | 문서 목록 도출 | 15~30개 문서 |
| 20분 | 그룹핑 | 비슷한 문서 묶기 | 4~6개 그룹 |
| 15분 | 우선순위 | 중요도x빈도 매트릭스 | MVP 범위 |
| 15분 | 휴식 | - | - |
| 20분 | 흐름 그리기 | 문서 간 화살표 연결 | 워크플로우 |
| 40분 | 양식 스케치 | 주요 문서 3~5개 상세화 | 양식 초안 |
| 15분 | 데이터 관계 | 입력 방식 확인 | ERD 초안 |
| 10분 | 확인/정리 | 체크리스트 확인 | 최종 산출물 |
| 5분 | 마무리 | 다음 단계 안내 | 액션 아이템 |

### 5.2 문서 양식 템플릿

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

### 5.3 데이터베이스 스키마 템플릿

#### 요청서 (-서) 테이블

```sql
CREATE TABLE {table_name} (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  {document_number} VARCHAR(20) UNIQUE NOT NULL,

  -- 업무 필드
  {field1_name} {data_type} NOT NULL,
  {field3_id} UUID NOT NULL,                       -- [선택] = FK

  -- 승인 워크플로우 (-서 패턴 필수)
  status VARCHAR(20) DEFAULT 'DRAFT'
    CHECK (status IN ('DRAFT', 'SUBMITTED', 'APPROVED', 'REJECTED')),
  submitted_at TIMESTAMP, submitted_by UUID,
  approved_at TIMESTAMP, approved_by UUID, approval_comment TEXT,
  rejected_at TIMESTAMP, rejected_by UUID, rejection_reason TEXT,

  -- 메타데이터
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by UUID NOT NULL,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY ({field3_id}) REFERENCES {related_table}(id)
);

CREATE INDEX idx_{table_name}_status ON {table_name}(status);
```

#### 기록지 (-지) 테이블

```sql
CREATE TABLE {table_name} (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  {document_number} VARCHAR(20) UNIQUE NOT NULL,
  {referenced_document_id} UUID NOT NULL,          -- 참조 문서

  -- 업무 필드
  {field1_name} {data_type} NOT NULL,

  -- 이력 추적 (-지 패턴 필수)
  history JSONB DEFAULT '[]'::JSONB,

  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by UUID NOT NULL,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY ({referenced_document_id}) REFERENCES {referenced_table}(id)
);
```

#### 1:N 관계 테이블

```sql
CREATE TABLE {parent_table}_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  {parent_table}_id UUID NOT NULL,
  {field1_name} {data_type} NOT NULL,
  display_order INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY ({parent_table}_id) REFERENCES {parent_table}(id) ON DELETE CASCADE
);
```

### 5.4 API 코드 템플릿

#### Express 라우터 템플릿

```typescript
const router = Router();

// CRUD
router.get('/', authenticate, controller.list);
router.get('/:id', authenticate, controller.get);
router.post('/', authenticate, validate(createSchema), controller.create);
router.put('/:id', authenticate, validate(updateSchema), controller.update);
router.delete('/:id', authenticate, authorize('ADMIN'), controller.remove);

// -서 패턴: 승인 워크플로우
router.post('/:id/submit', authenticate, controller.submit);
router.post('/:id/approve', authenticate, authorize('MANAGER', 'ADMIN'), controller.approve);
router.post('/:id/reject', authenticate, authorize('MANAGER', 'ADMIN'), controller.reject);

// -지 패턴: 이력 조회
router.get('/:id/history', authenticate, controller.getHistory);

// 파일 첨부
router.post('/:id/attachments', authenticate, controller.uploadAttachment);
router.get('/:id/attachments', authenticate, controller.listAttachments);
```

#### 서비스 레이어 템플릿

```typescript
class ResourceService {
  async create(data)  { /* status = DRAFT, 번호 자동생성 */ }
  async findById(id)  { /* 조회 */ }
  async list(filters) { /* 필터링 + 페이징 */ }
  async update(id, data) { /* 수정 */ }
  async submit(id, userId) {
    // DRAFT → SUBMITTED, 승인자 알림
  }
  async approve(id, userId, comment?) {
    // SUBMITTED → APPROVED, 작성자 알림
  }
}
```

### 5.5 프로젝트 구조 템플릿

```
{project-name}/
├── src/
│   ├── config/        (database, jwt, app)
│   ├── models/        (user, resource1, resource2)
│   ├── controllers/   (auth, resource1, resource2)
│   ├── services/      (resource1-service, resource2-service)
│   ├── routes/        (auth, resource1, resource2)
│   ├── middleware/     (auth, validation, error-handler)
│   ├── schemas/       (resource1, resource2)
│   ├── database/
│   │   ├── migrations/
│   │   └── seeds/
│   ├── utils/         (logger, errors)
│   └── server.ts
├── tests/
│   ├── unit/
│   └── integration/
├── docs/
├── .env.example
├── package.json
└── tsconfig.json
```

### 5.6 템플릿 사용 가이드

1. **워크숍 준비**: 초대 이메일 → 프로젝트 정보 입력 → 발송. 체크리스트 출력.
2. **문서 양식**: 도출된 문서명으로 접미사(-서/-지/-표/-록) 템플릿 선택 → 항목 대체 → Excel로 현업 검증
3. **데이터베이스**: 접미사별 테이블 템플릿 선택 → `{table_name}`, `{field_name}` 실제 값으로 대체
4. **API 코드**: `{resource}` 리소스명으로 대체 → 접미사별 패턴 확인 후 적용
5. **프로젝트 구조**: 디렉터리 구조 복사 → 실제 이름 변경 → 설정 조정

---

## 6. 사례 연구

### 6.1 프로젝트 개요: 플라스틱 사출 공장 MES

| 항목 | 내용 |
|------|------|
| **산업** | 플라스틱 사출 제조 |
| **규모** | 직원 30명, 사출기 8대 |
| **기간** | 4개월 (2023.03 ~ 2023.06) |
| **예산** | 35,000,000원 |
| **팀** | 개발 2명, 현업 5명 |

**도입 전 문제점**: 종이 작업지시서 (분실, 오기재), Excel 품질관리 (실시간 불가), 구두 지시 (누락, 오해). 불량률 3.2%, 정보 검색 60분/일.

**목표**: 실시간 생산 현황 파악, 품질 데이터 자동 집계, 불량률 1% 이하 달성.

### 6.2 Phase 1: 워크숍 (1일, 2시간)

#### 서식(FormType) 도출

**Step 1: 브레인스토밍 (30분)**

```
도출된 22개 서식:
작업지시서, 자재요청서, 입고확인서, 일일생산일지,
품질검사의뢰서, 검사기록지, 합격확인서, 불량기록지,
금형정비의뢰서, 정비일지, 시정조치서, 예방조치서,
출하지시서, 출하확인서, 월별생산현황표, 불량이력록,
재고현황표, 작업표준서, 검사기준서, 설비가동일지,
가동률현황표, 재작업의뢰서
```

**Step 2: 그룹핑 (20분)**

```
[요청/의뢰] 6개: 작업지시서, 자재요청서, 품질검사의뢰서...
[기록]      8개: 일일생산일지, 검사기록지, 불량기록지...
[보고/명세] 2개: 시정조치서, 예방조치서
[표/현황]   3개: 월별생산현황표, 재고현황표...
[록/이력]   1개: 불량이력록
[기준/표준] 2개: 작업표준서, 검사기준서
```

**Step 3: 우선순위 (15분)**

```
MVP 7개: 작업지시서, 일일생산일지, 품질검사의뢰서,
         검사기록지, 불량기록지, 합격확인서, 월별생산현황표
```

**Step 4: 주요 양식 스케치 (40분)**

작업지시서 양식 구조:
```
┌──────────────────────────────┐
│ [Main Section] 지시정보      │
├──────────────────────────────┤
│ 지시번호: [자동]             │
│ 제품명: [Selection]          │ ← Reference Section (products FK)
│ 수량: [Numeric]              │
│ 사출기: [Selection]          │ ← Reference Section (machines FK)
│ 금형: [Selection]            │ ← Reference Section (molds FK)
├──────────────────────────────┤
│ [Child Section] 자재목록:    │ ← 1:N
│ - 원료 [Selection] [수량]    │
├──────────────────────────────┤
│ [Reference Section] 작업기준 │
│ - 작업표준서: [참조]         │ ← 최신 버전 (FK)
│ - 검사기준서: [참조]         │
└──────────────────────────────┘
```

**Section → Entity 사상 자동 도출**:
```
작업지시서 → Main Section → work_orders
  ├─ Child Section → work_order_materials (1:N)
  ├─ Reference Section → products, machines (FK)
  └─ Reference Section → work_standards (FK, latest)

일일생산일지 → Main Section → production_logs
  ├─ Reference Section → work_orders (FK)
  ├─ Child Section → defect_occurrences (M:N)
  └─ FileUpload → production_attachments (1:N)
```

**워크숍 성과**: 22개 서식 도출, 7개 MVP 선정, 4개 주요 양식 구조 스케치, ERD 초안 자동 도출. **소요 시간: 2시간 15분**. 참석자 7명(현업 5, 개발 2) 전원이 도출된 서식 목록에 그 자리에서 동의했습니다.

> **왜 표준 절차(3시간 20분)보다 짧았는가**: [워크숍 가이드](workshop.md#타임테이블-총-3시간-20분-휴식-15분-포함)의 Step 6(흐름 그리기)과 Step 8(데이터 관계 발견)을 이 세션에서는 수행하지 않고 후속 작업으로 넘겼습니다. 흐름도는 개발팀이 초안을 만들어 1주 후 검증 미팅에서 확인했습니다. **표준 절차를 다 밟으면 3시간 20분이 걸립니다.**

### 6.3 Phase 2: 개발 (3개월)

#### Sprint 1-2 (4주): 기본 기능

- 7개 문서 CRUD 완성
- 문서 간 관계 구현
- 기본 워크플로우 자동화

#### Sprint 3 (2주): 패턴 발견

개발자가 자연스럽게 발견한 패턴:

```
패턴 1: "의뢰서" = 승인 필요 → RequestDocument 추상 클래스
패턴 2: "기록지" = 이력 추적 → RecordDocument 추상 클래스
패턴 3: "일지" = 시간순 정렬 → DailyDocument 추상 클래스
패턴 4: "표" = 집계/통계 → SummaryDocument 추상 클래스
```

**효과**: Sprint 4 이후 추가된 서식은 해당 추상 클래스를 상속하는 것으로 승인 흐름·이력 추적·정렬 규칙이 이미 갖춰진 상태에서 시작했습니다. 접미어가 같으면 UX도 같아졌고, 이는 설계한 것이 아니라 따라온 것입니다.

**중요**: 현업은 이런 패턴을 전혀 모릅니다. 그냥 "의뢰서", "기록지"일 뿐입니다.

#### Sprint 4-6 (6주): 2차 기능

- 작업표준서, 검사기준서 (버전 관리)
- 통계 화면 및 대시보드
- 모바일 최적화
- 바코드 스캔 기능

### 6.4 Phase 3: 운영

```
Week 1: 관리자 교육 (2시간)
Week 2: 현장 직원 교육 (3시간 x 2회)
Week 3: 병행 운영 (종이 + 시스템)
Week 4: 완전 전환
```

### 6.5 성과 측정

#### 정량적 성과

| 지표 | 도입 전 | 도입 후 | 개선 |
|------|---------|---------|------|
| **불량률** | 3.2% | 0.8% | **75% 감소** |
| **정보 검색** | 60분/일 | 5분/일 | **92% 감소** |
| **현황 파악** | 4시간 | 실시간 | **100% 향상** |
| **문서 작성** | 30분/건 | 5분/건 | **83% 감소** |
| **데이터 정확도** | 70% | 95%+ | **36% 향상** |

> **이 숫자가 무엇의 효과인지에 대하여.** 위 개선은 **종이·구두 업무를 디지털 시스템으로 옮긴 것 전체의 효과**이며, 그중 Formology 고유의 기여분은 분리 측정되지 않았습니다. 대조군 — 같은 공장을 전통 방식으로 구축한 시스템 — 이 존재하지 않기 때문입니다.
>
> 방법론이 실제로 책임지는 구간은 따로 있습니다: **워크숍에서 도출한 22개 서식 목록으로 요구사항 수집이 끝났고, 개발 4개월 동안 추가 요구사항 수집 세션이 열리지 않았습니다.** 불량률은 디지털화의 성과이고, 요구사항이 뒤집히지 않은 것은 방법론의 성과입니다. 둘을 섞어 읽지 마십시오.

**ROI**:
```
투자: 35,000,000원

연간 효과:
- 불량 감소: 45,000,000원
- 시간 절약: 18,000,000원
- 재고 최적화: 8,000,000원
총: 71,000,000원/년

ROI: 203% (첫 해)
회수 기간: 6개월
```

> 연간 효과 항목은 공장 측이 산정한 값이며 독립 검증을 거치지 않았습니다. **이 수치를 다른 프로젝트의 기대값으로 옮기지 마십시오.** 불량 감소액이 전체의 63%를 차지하는데, 그 크기는 도입 전 불량률(3.2%)에 종속됩니다. 불량률이 낮은 공장에서는 같은 방법론이 같은 ROI를 내지 않습니다.

#### 정성적 성과

- 생산 관리자: "우리가 쓰던 용어 그대로라 적응이 빨랐어요."
- 품질 담당자: "품질검사 결과를 실시간으로 보니 즉시 대응 가능해요."
- 현장 반장: "최신 작업표준서를 자동으로 보여주니 구버전 실수가 없어졌어요."
- 공장장: "실시간 전체 현황이 이렇게 강력할 줄 몰랐습니다."
- 개발자: "Formology 덕분에 현업과 소통이 쉬웠어요. Entity 설명 안 해도 되니까."

### 6.6 성공 요인

1. **현업 주도 설계**: 현업이 문서를 나열하고, 개발자가 그대로 구현. 2시간 만에 합의.
2. **점진적 복잡도**: Phase 1 단순 CRUD → Phase 2 자동화 → Phase 3 패턴 적용 (현업은 모름)
3. **즉시 검증**: "이 문서 목록 맞아요?" → "네!" — 이의 제기 없이 승인. 1개월 후 "화면이 양식이랑 똑같네요."
4. **자연스러운 진화**: 초기 22개 → 1개월 +2개 → 3개월 +3개. 강요 없이 필요 시 추가.

### 6.7 실패한 것 — 그리고 그것이 절차를 어떻게 바꿨는가

성공 사례만 싣는 방법론 문서는, 역설적으로 **적용 범위를 모르는 문서**로 읽힙니다. 독자가 "우리 회사에 되나?"를 판단할 근거를 얻지 못하기 때문입니다. 그래서 이 절은 잘된 것보다 **어긋난 것**을 먼저 적습니다.

#### 워크숍이 놓친 것 셋

**1. 사용 기기를 묻지 않았다.**

양식 스케치는 정확했지만 전부 PC 화면을 전제한 구조였습니다. 현장은 태블릿을 원했고, 한 화면에 다 들어가던 섹션이 태블릿에서는 스크롤 세 번이 되었습니다. 2주를 들여 반응형으로 다시 짰습니다.

> **왜 놓쳤는가**: 워크숍은 "무엇을 쓰는가"를 물었지 "어디서 쓰는가"를 묻지 않았습니다. 서식은 매체 중립적이라고 암묵적으로 가정한 것입니다. 종이 서식은 실제로 매체 중립적이지만 **화면은 아닙니다.**

**2. 입력 방식을 묻지 않았다.**

LOT번호는 워크숍에서 "텍스트 입력"으로 정리되었습니다. 현장에서는 그 번호가 이미 바코드로 인쇄되어 붙어 있었습니다. 작업자는 여덟 자리를 손으로 옮겨 적고 있었고, 오타가 났습니다. 바코드 스캔은 2차 기능으로 밀려 있었습니다.

> **왜 놓쳤는가**: 서식의 **필드 타입**은 물었지만 **입력 경로**는 묻지 않았습니다. 종이에서는 둘이 같습니다 — 손으로 씁니다. 시스템에서는 갈라집니다.

**3. 독자를 묻지 않았다.**

우선순위 매트릭스는 "빈번함 × 중요함"으로 그렸고, 월별생산현황표는 빈도가 낮아 2차로 내려갔습니다. 그런데 그 표는 **경영진이 보는 유일한 화면**이었습니다. 빈도는 낮고 중요도는 최고였는데, 매트릭스를 채운 사람들이 현장 실무자였으므로 그 중요도가 반영되지 않았습니다.

> **왜 놓쳤는가**: 매트릭스의 "중요함"이 **누구에게 중요한가**를 묻지 않았습니다. 참석자 구성이 곧 중요도 판정의 표본이라는 사실을 절차가 인지하지 못했습니다.

#### 여기서 절차가 바뀌었다

세 실패는 방법론의 한계가 아니라 **적용 조건의 발견**이었습니다. Step 7(양식 스케치)에 질문 셋이 추가되었고, 이후 세션에서는 같은 문제가 재현되지 않았습니다.

| 발견 | Step 7에 추가된 질문 |
|---|---|
| 매체를 물어야 한다 | "이 서식, 어떤 기기로 쓰세요?" (PC / 태블릿 / 모바일) |
| 입력 경로를 물어야 한다 | "이 칸, 손으로 치세요? 아니면 스캔하거나 자동으로 들어와요?" |
| 독자를 물어야 한다 | "이거 주로 누가 봐요?" (현장 / 관리자 / 경영진) |

> 세 질문 모두 **현장 언어**이고 **서식 한 장을 앞에 놓고** 물을 수 있습니다. P0 원칙을 건드리지 않으면서 절차가 개선되었습니다. 방법론이 자기 실패로부터 배우는 경로가 이렇게 생깁니다.

#### 잘된 것

- 워크숍 산출물을 **재해석 없이** 개발 입력으로 사용 — 중간 변환 문서가 없었습니다
- MVP 우선순위 엄수 (7개만) — 22개를 다 만들자는 압력을 막았습니다
- 패턴 내부화 — 추상 클래스 4종은 현업에게 한 번도 노출되지 않았습니다
- 즉시 피드백 반영 — 모바일 문제가 2주 안에 해결되었습니다

### 6.8 프로젝트 타임라인

```
2023.03.10  워크숍 (2시간)
2023.03.13  개발 시작
2023.04.10  Sprint 1-2 완료
2023.05.08  Sprint 3-4 완료
2023.06.12  Sprint 5-6 완료
2023.06.19  교육 및 전환
2023.07.01  정식 운영
2023.09.30  성과 측정 — 모든 목표 달성
```

---

## 7. FAQ

### 일반 질문

**Q1: Formology와 기존 방법론의 가장 큰 차이는?**

시작점이 다릅니다. 기존은 추상적 개념(Entity, Domain Model)부터, Formology는 구체적 문서(의뢰서, 보고서)부터 시작합니다. 결과적으로 현업이 직접 참여하고 즉시 검증할 수 있습니다.

**Q2: 문서가 너무 많이 나오면?**

괜찮습니다. 실제 업무가 그만큼 복잡한 것입니다. 그룹핑 → MVP 핵심 5~7개만 → 단계별 개발 → 패턴 재사용으로 대응합니다. 50개 도출 시: 10개 그룹 → MVP 7개 → 2차 15개 → 3차 나머지.

**Q3: 개발자가 Entity 모델링 하고 싶어해요**

코드 내부에서는 자유, 하지만 현업에게는 문서 용어만 사용합니다. 현업 소통: "품질검사의뢰서", 코드 내부: "QcRequest". 화면/메뉴는 반드시 문서 용어로.

**Q4: 전자결재 시스템이랑 뭐가 다른가요?**

범위가 다릅니다. 전자결재는 결재 문서만, Formology는 모든 업무 문서(작업일지, 체크리스트 포함)를 다룹니다. 전자결재는 Formology의 일부입니다.

### 기술 질문

**Q5: 모든 문서를 DB에 저장하나요?**

하이브리드 권장. 구조화된 데이터(메타정보, 필드 값, 관계) → DB. 비구조화된 데이터(이미지, 첨부파일) → 파일 (경로만 DB).

**Q6: 성능 문제는 없나요?**

일반적으로 문제없습니다. 오히려 유리합니다: 문서 단위 조회로 인덱스 설계 용이, 자연스러운 정규화, 예측 가능한 쿼리 패턴. 주의: 대용량 첨부파일은 별도 스토리지, 이력 테이블은 파티셔닝, 검색은 전문검색 엔진 활용.

**Q7: 마이크로서비스에도 맞나요?**

완벽히 맞습니다. 문서 = Bounded Context. 품질검사 서비스(검사의뢰서, 검사기록지), 생산 서비스(작업지시서, 작업일지), 자재 서비스(자재요청서, 입고확인서). 서비스 간 통신: 문서 ID로 참조, 이벤트 기반.

**Q8: 문서 버전 관리는 어떻게?**

패턴별로 다르게 접근:
- 패턴 1: 이력 테이블 (version, snapshot JSON)
- 패턴 2: Supersedes 관계 (supersedes_id FK, effective_date)
- 패턴 3: 이벤트 소싱 (event_type, event_data JSON)

### 적용 질문

**Q9: 어떤 프로젝트에 적합한가요?**

- 매우 적합: 제조 MES/QMS/CMMS, 병원 EMR/OCS, 물류 WMS/TMS, 사무 전자결재/문서관리, 연구 LIMS
- 부분 적합: SNS, 이커머스, 교육
- 부적합: 순수 알고리즘, 실시간 제어 시스템, 게임, 스트리밍

**Q10: 기존 시스템에 적용 가능한가요?**

가능하지만 단계적 접근이 필요합니다. 1단계: 기존 화면/데이터 분석 → 문서 역변환 가능 확인. 2단계: 기존→문서 매핑. 3단계: 신규 기능은 Formology 방식, 기존 기능은 점진적 전환.

**Q11: 비개발자도 시스템 설계에 참여할 수 있나요?**

바로 그것이 Formology의 핵심입니다. 현업이 문서 목록 도출, 양식 스케치, 업무 흐름 정의를 주도합니다. IT 지식은 불필요하고 업무 이해만 필수입니다.

### 비교 질문

**Q12: DDD(Domain-Driven Design)와의 차이는?**

| 측면 | DDD | Formology |
|------|-----|---------|
| 시작 | Entity, Aggregate | 문서, 양식 |
| 추상화 | 즉시 | 점진적 |
| 현업 참여 | 간접 (인터뷰) | 직접 (워크숍) |
| 학습곡선 | 가파름 | 완만함 |

결합 가능: Formology로 시작(구체적) → 패턴 발견 후 DDD 적용(추상화).

**Q13: 온톨로지와의 관계는?**

Formology는 경량 온톨로지를 자연스럽게 도출합니다. 전통 온톨로지는 최상위 개념부터, Formology는 구체적 문서부터 시작합니다. 온톨로지 용어는 시스템 내부에, 문서 용어는 시스템 표면에 위치합니다.

**Q14: 추천 기술 스택은?**

Formology는 기술 중립적이지만 추천은 있습니다. Backend: Node.js/Python/Java/C#. Database: PostgreSQL (추천). Frontend: React/Vue/Angular + Form 라이브러리. 문서 기반 REST API 구현과 양식 기반 UI 개발에 용이한 스택이면 됩니다.

### 프로세스 질문

**Q15: 워크숍 참석자가 의견 충돌하면?**

둘 다 기록하고 차이를 명확히 합니다. "A부서는 이렇게, B부서는 이렇게." 시스템에서 둘 다 지원 검토 → 불가능하면 우선순위 협의. 진행자가 판단하거나 한쪽 의견을 무시하거나 완벽한 합의를 강요하면 안 됩니다.

**Q16: 워크숍 후 문서가 계속 바뀌면?**

자연스러운 현상입니다. 변경 이력 관리(언제, 누가, 왜), 영향도 분석(다른 문서 영향?), 버전 관리(v1.0→v1.1), 정기 리뷰(분기별)로 대응합니다.

**Q17: 실제 효과는?**

측정한 것은 [사례 연구](#6-사례-연구) 하나뿐이며, 거기서 확인된 사실은 셋입니다.

- 워크숍 **2시간 15분**에 22개 서식이 도출되고 참석자 7명 전원이 그 자리에서 동의했다.
- 개발 4개월 동안 **추가 요구사항 수집 세션이 열리지 않았다.** 서식 수는 22개에서 27개로 늘었으나 모두 기존 목록에 얹히는 증분이었고, 설계를 뒤집는 변경은 없었다.
- 1개월 후 현업의 반응은 **"화면이 양식이랑 똑같네요"**였다.

전통 방식과의 대조 수치는 싣지 않습니다. 같은 조직에서 두 방식을 나란히 돌려 본 적이 없기 때문입니다. 그런 비교표를 제시하는 방법론 문서는 대개 대조군을 측정하지 않았습니다.

---

## 8. 용어 사전

### 핵심 개념 체계

Formology는 **2차원 개념 체계**를 사용합니다:

```
[차원 1: 인스턴스 계층 - 수직]
FormType → Form → Document → Record

[차원 2: 구조 분해 - 수평]
Form = Section[] + Field[]
```

### 인스턴스 계층

#### L0: FormType (서식 개념)

업무에서 사용하는 **문서의 추상적 개념**. 모양 없는 순수 개념. 현업이 "우리는 이런 서식 씁니다"라고 부르는 이름. 1개 FormType → N개 Form (1:N).

예시: "작업지시서", "품질검사의뢰서", "설비점검일지", "월별생산현황표"

#### L1: Form (양식 구조)

FormType의 **구체적 Section/Field 구조**. 모양이 있는 구체적 레이아웃. Section과 Field로 구성. 동일 FormType의 버전별/부서별 양식이 존재할 수 있음.

```
Form 예시:
┌─────────────────────────────┐
│ [Main Section] 헤더         │
│  문서번호: [자동채번] Field  │
├─────────────────────────────┤
│ [Main Section] 본문         │
│  제품명: [Selection] Field  │ ← Reference Section
│  수량: [Numeric] Field      │
├─────────────────────────────┤
│ [Child Section] 상세 (1:N)  │
│  항목: [Text] Field         │
└─────────────────────────────┘
```

#### L2: Document (문서 인스턴스)

유저가 **실제로 작성한 Form의 인스턴스**. Field에 데이터가 채워진 상태. 문서 ID와 고유 상태(초안/제출/승인 등)를 가짐.

예시: 작업지시서 #WO-2024-001 (초안), #WO-2024-002 (승인됨)

Document 생명주기: 생성 → 초안 → 제출 → 승인 → 완료 → 보관

#### L3: Record (레코드)

Document가 **데이터베이스에 저장된 형태**. Section → Entity(테이블), Field → Attribute(컬럼).

```
[Main Section]        → main_table
  Field: 문서번호    → column: document_id
  Field: 제품명 (FK) → column: product_id (FK)
[Child Section] (1:N) → child_table
  Field: 항목        → column: item_name
```

### 구조 분해

#### Section (섹션)

Form을 구성하는 **의미 단위 그룹**. Section → Entity(테이블) 사상의 기본 단위.

**4가지 Section 유형**:

| 유형 | 개념 | 원본 수정 | DB 구현 | 사용 예시 |
|------|------|-----------|---------|-----------|
| **Main Section** | 핵심 정보 섹션 | - | 주 테이블 | 문서 본문 |
| **Child Section** | 1:N 종속 섹션 | - | 자식 테이블 (parent FK) | 자재 내역, 검사 항목 |
| **Reference Section** | 살아있는 링크 | 자동 반영 | FK | 승인/처리 문서 |
| **Attachment Section** | 고정된 사본 | 영향 없음 | 복제 테이블/JSON | 증빙/계약 문서 |

**선택 기준**: 실시간 동기화 필요 → Reference Section. 과거 시점 고정 필요 → Attachment Section.

#### Field (필드)

Section 내 **개별 입력 단위**. Field → Attribute(컬럼) 사상.

**기본 입력 Field**:

| Field 유형 | 설명 | DB 타입 |
|-----------|------|---------|
| Text | 단행 텍스트 | VARCHAR |
| TextArea | 다행 텍스트 | TEXT |
| Numeric | 숫자 | INTEGER, DECIMAL |
| Date | 날짜 | DATE |
| Time | 시간 | TIME, TIMESTAMP |
| Checkbox | 단일 체크 | BOOLEAN |

**선택 입력 Field**:

| Field 유형 | 설명 | DB 타입 |
|-----------|------|---------|
| Selection | 단일 선택 (드롭다운) | VARCHAR (FK) |
| Radio | 단일 선택 (라디오) | ENUM |
| Checklist | 다중 선택 | JSON, M:N 테이블 |

**특수 Field**: FileUpload (파일 경로), 자동채번 (시스템 생성), 자동 (로그인 유저 등)

**핵심 규칙**: Reference Section 내의 Selection Field → FK (Foreign Key) 사상. Main Section의 Selection → ENUM 또는 일반 컬럼.

### 사상 규칙 요약

| 규칙 | From | To |
|------|------|-----|
| 규칙 1 | Section | Entity (테이블) |
| 규칙 2 | Field | Attribute (컬럼) |
| 규칙 3 | Child Section | 1:N 관계 (parent FK) |
| 규칙 4 | Reference Section | FK (실시간 동기화) |
| 규칙 5 | Attachment Section | 복제 테이블/JSON (과거 고정) |

### 문서 분류 체계 (서/지/표/록)

#### -서 (書): 공식 문서

공식 요청/의뢰/승인. 워크플로우: 생성→초안→제출→승인→완료. 법적/계약적 효력.
예시: 작업지시서, 품질검사의뢰서, 자재요청서, 승인요청서.

#### -지 (紙): 일상 기록

일상 업무 기록. 시간순 정렬(날짜별). 승인 없이 즉시 작성.
예시: 작업일지, 설비점검일지, 생산일지, 근무일지.

#### -표 (表): 집계/현황

데이터 집계/통계. 시스템 자동 생성 가능. 읽기 전용(조회용). PDF/Excel 출력.
예시: 월별생산현황표, 재고현황표, 불량현황표, 가동률현황표.

#### -록 (錄): 이력/로그

변경 이력 추적. 시간순 누적 기록. Append-only (삭제/수정 불가).
예시: 불량이력록, 변경이력록, 승인이력록, 접근이력록.

### 명명 규칙

#### FormType 명명

현업이 부르는 이름 그대로. 패턴: `[업무영역][대상][접미사]`.

좋은 예: 작업지시서, 품질검사의뢰서, 설비점검일지.
나쁜 예: 작업지시(접미사 누락), WorkOrder(영문 사용).

#### 구현 레이어별 명명

| 레이어 | 규칙 | 예시 |
|--------|------|------|
| 현업/화면 | 한글 원문 | 작업지시서 |
| API Endpoint | kebab-case | `/api/v1/work-orders` |
| Database | snake_case | `work_orders` |
| Code (변수) | camelCase | `workOrder` |
| Code (클래스) | PascalCase | `WorkOrder` |

### 용어 사용 가이드

#### 현업과 대화 시

사용: FormType, 양식, 문서, 섹션, 항목.
회피: Entity, Record, Table, FK, Schema.

```
"작업지시서라는 서식(FormType)이 있고요,
그 양식(Form)은 지시 정보 섹션(Main Section)과
자재 내역 섹션(Child Section)으로 나뉩니다."
```

#### 개발자와 대화 시

사용: FormType, Form, Section, Field, Record, Entity, FK 모두 가능.

```
"작업지시서 FormType의 Form 구조에서
Main Section은 work_orders 테이블로 사상되고,
Child Section은 work_order_materials로 1:N 관계를 형성합니다.
Reference Section의 Selection Field는 FK로 구현됩니다."
```

### 개념 변환 흐름도

```
[현업 언어] "우리는 작업지시서를 씁니다"
  ↓
[FormType] "작업지시서"
  ↓
[Form 구조] Main Section + Child Section + Reference Section
  ↓
[Record 사상] work_orders + work_order_materials (1:N) + FK
  ↓
[구현] Database + API + Frontend
```

### 체크리스트

**FormType 정의**: 현업 실제 이름? 접미사 명확? 업무 역할 명확? 중복 없음?

**Form 설계**: Section 유형 명확? Main Section 1개 이상? Child 1:N 명확? Reference=실시간? Attachment=과거 고정? Field 유형 적절? Selection=FK?

**Record 사상**: Section→Entity 명확? Field→Attribute 명확? FK 정의? 1:N parent_id? snake_case? 필수/선택 제약?

---

> **"사상은 변환이 아닌 발견이다"**
>
> **"기술은 바뀌어도 원리는 불변이다"**

**Formology** — *Workflow First, Ontology Follows*
