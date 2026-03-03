# 방법론

Formology의 핵심 개념: 문서 분류, 구조, 관계, 네이밍, 데이터 모델링.

---

## 1. 문서의 종류

### 모든 것이 문서다

업무의 모든 커뮤니케이션은 **포멀리티(formality)의 차이**일 뿐, 본질적으로 모두 문서입니다.

```
비공식 ←────────────────────────────────→ 공식
(informal)                           (formal)

구두지시   메모   카톡   이메일   의뢰서   계약서
```

**세 가지 차원**으로 구분됩니다:

| 차원 | 비공식 | 중간 | 공식 |
|------|--------|------|------|
| 작성 노력 | 간단히 | 표준 양식 | 엄격한 절차 |
| 보관 기간 | 임시 | 중기 | 장기/영구 |
| 공식성 | 구두 확인 | 업무 기록 | 법적 효력 |

### 서(書)/지(紙)/표(表)/록(錄) 분류 체계

**서(書)** — 공식 문서 (포멀리티 레벨 2~5)

| 접미어 | 의미 | 예시 |
|--------|------|------|
| 의뢰서 | "해주세요" | 품질검사의뢰서 |
| 신청서 | "허가해주세요" | 휴가신청서 |
| 지시서 | "하세요" | 작업지시서 |
| 보고서 | "했습니다" | 품질검사보고서 |
| 확인서 | "맞습니다" | 출하확인서 |
| 명세서 | "이렇습니다" | 거래명세서 |
| 계약서 | "약속합니다" | 품질보증계약서 |

**지(紙)** — 기록 용지 (포멀리티 레벨 0~2)

| 접미어 | 의미 | 예시 |
|--------|------|------|
| 메모 | 간단한 기록 | 업무메모 |
| 일지 | 날마다 기록 | 작업일지 |
| 기록지 | 사실 기록 | 불량기록지 |
| 체크지 | 점검 항목 | 설비점검체크지 |

**표(表)** — 목록/집계 (여러 문서의 집계 View)

| 접미어 | 의미 | 예시 |
|--------|------|------|
| 현황표 | 지금 상태 | 재고현황표 |
| 집계표 | 합산 결과 | 일일생산집계표 |
| 목록표 | 항목 나열 | 자재목록표 |

**록(錄)** — 이력 모음 (여러 문서의 이력 View)

| 접미어 | 의미 | 예시 |
|--------|------|------|
| 회의록 | 회의 내용 | 품질회의록 |
| 이력록 | 변경 이력 | 설비이력록 |
| 대장 | 전체 관리 목록 | 거래처대장 |

> **핵심**: 표/록/철/편은 Document를 "보는 방식"일 뿐입니다. 같은 Record 데이터, 다른 View/Query.

### 포멀리티 레벨

| 레벨 | 포멀리티 | 예시 | 시스템 요구사항 |
|------|---------|------|----------------|
| 0 | 암묵적 | 전화, 구두지시 | 선택적 기록 |
| 1 | 낮음 | 메모, 카톡 | 간단한 입력 |
| 2 | 중간 | 의뢰서, 일지 | 양식 준수 |
| 3 | 높음 | 지시서, 보고서 | 승인 필요 |
| 4 | 매우높음 | 검사보고서 | 다단계 승인 |
| 5 | 법적 | 계약서, 증명서 | 법적 효력, 영구 보관 |

### 실전 적용 패턴

**패턴 1: 요청-응답**
```
의뢰서 → 기록지 → 보고서
예: 품질검사의뢰서 → 품질검사기록지 → 품질검사보고서
```

**패턴 2: 지시-실행-보고**
```
지시서 → 일지 → 보고서
예: 작업지시서 → 작업일지 → 작업완료보고서
```

**패턴 3: 신청-검토-승인**
```
신청서 → 검토의견서 → 승인서
예: 휴가신청서 → 부서장검토의견서 → 휴가승인서
```

---

## 2. 문서의 구조

### FormType → Form → Document → Record

Formology는 두 개의 독립적 차원으로 문서를 이해합니다.

**차원 1: 인스턴스 계층** (추상에서 구체로)

```
FormType (양식 유형 / 서식)
  │ = 추상 개념, 모양 없음
  │ = 필수 본질 정의: "이것이 없으면 이 서식이 아니다"
  │
  ├─ realization (구체화, 1:N)
  ↓
Form (양식)
  │ = 구체적 레이아웃, 모양 있음
  │ = 필수 요소(상속) + 부가 요소(확장) + Decoration(UX 최적화)
  │
  ├─ fill (작성)
  ↓
Document (문서)
  │ = 양식지를 유저가 작성한 결과
  │ = 사람이 보고, 읽고, 공유하고, 승인하는 단위
  │
  ├─ persist (저장)
  ↓
Record (레코드)
  = 데이터베이스에 저장된 데이터
  = 시스템이 처리하는 단위
```

**예시**:
- **FormType**: "품질검사의뢰서" — 필수 Section/Field: 제품명, LOT번호, 수량, 검사항목
- **Form**: Web 입력 양식, 모바일 양식, PDF 출력 양식 (같은 FormType의 1:N 구현)
- **Document**: QC-REQ-001 — 2024-11-07 김철수가 작성한 의뢰서
- **Record**: qc_requests 테이블의 한 행

**차원 2: 구조 분해** (전체에서 부분으로)

```
Form / Document
  ├─ Section (섹션) = 의미적 그룹, 엔티티 경계 결정
  │   ├─ Main Section → 메인 테이블
  │   ├─ Child Section → 자식 테이블 (1:N)
  │   ├─ Reference Section → FK (살아있는 링크)
  │   └─ Attachment Section → 파일 테이블 (스냅샷)
  │
  └─ Field (필드) = 원자적 데이터 단위, Column으로 매핑
      ├─ Text → VARCHAR/TEXT
      ├─ Numeric → INTEGER/DECIMAL
      ├─ Date/Time → DATE/TIMESTAMP
      ├─ Selection → FK
      ├─ Boolean → BOOLEAN
      └─ File → VARCHAR (경로)
```

### 포멀리티와 계층의 관계

| 레벨 | FormType | Form | Document | Record |
|------|----------|------|----------|--------|
| 0~1 | 불필요~간단 | 간소 | 선택적 | 최소 |
| 2~3 | 표준~엄격 | 표준 | 필수, 승인 | 이력 관리 |
| 4~5 | 법적 | 법정 | 다단계 승인 | 감사 추적 |

---

## 3. 문서의 관계

문서는 독립적으로 존재하지 않습니다. Form의 Section 구조가 문서 간 관계를 정의합니다.

### 참조 (Reference Section) — 살아있는 링크

**정의**: 다른 Document나 마스터 데이터를 현재 시점에서 가리키는 Section.

**구현**: Foreign Key (FK)

**특징**:
- 원본 수정 시 자동 반영
- 최신 상태 유지
- 저장 공간 효율적

**사용 시나리오**: 마스터 데이터 참조, 진행 중인 프로젝트, 실시간 정보 필요

```
Form: 품질검사의뢰서
└─ [Reference Section]
    └─ Field: 제품명 [선택 ▼] → FK to 제품 마스터
```

### 붙임 (Attachment Section) — 스냅샷

**정의**: 과거 시점의 복사본을 저장하는 Section.

**구현**: 복제 테이블 또는 파일 저장

**특징**:
- 원본 변경과 무관
- 그 시점의 증거 보존
- 법적 효력

**사용 시나리오**: 승인 시점 자료, 계약 증빙, 감사 대응

```
Form: 품질검사보고서
└─ [Attachment Section]
    ├─ 검사기준서 v2.1 스냅샷 (복제 테이블)
    └─ 불량사진.jpg (파일)
```

### 스냅샷 — Attachment의 특수 형태

Document의 특정 버전 상태를 복사하여 이력 관리에 활용합니다.

**활용 패턴**:
- **승인 시 스냅샷**: 문서 승인 시 현재 상태 저장
- **주기적 스냅샷**: 매월 말일 진행 중 문서 상태 저장
- **이벤트 기반 스냅샷**: 계약 체결, 감사 시작, 분쟁 발생 시

### 선택 기준

| 기준 | 참조 (Reference) | 붙임 (Attachment) |
|------|-------------------|-------------------|
| 시점 | 현재 (living) | 과거 (frozen) |
| 동기화 | O | X |
| 증거 능력 | 약함 | 강함 |
| 포멀리티 | 레벨 0~3 | 레벨 4~5 |
| DB 구현 | FK | 복제 테이블 or 파일 |

**원칙**:
- 마스터 데이터 → Reference Section (FK)
- 법적 증빙 → Attachment Section (스냅샷)
- 동일 문서에 혼용 가능 (진행 중 = FK, 완료 시 = 스냅샷)

---

## 4. 문서 이름 짓기

### 기본 공식

```
[대상] + [행위] + [접미어]

예시:
품질검사 + 의뢰 + 서 = "품질검사의뢰서"
작업 + 지시 + 서 = "작업지시서"
불량 + 기록 + 지 = "불량기록지"
```

### 네이밍 원칙

1. **현장 언어 우선**: ❌ `InspectionRequest` ✅ 품질검사의뢰서
2. **구체적으로**: ❌ 관리서 ✅ 재고관리대장
3. **일관성 유지**: 같은 도메인은 같은 용어 (품질검사의뢰서, 품질검사기록지, 품질검사보고서)
4. **간결성**: 7자 이내 권장 (단, 명확성 > 간결성)

### 메뉴/화면 이름

```
❌ 개발자 중심: "환자 추가", "작업 등록", "검사 입력"
✅ 현장 중심: "환자기록지 작성", "작업지시서 발행", "검사기록지 작성"

패턴: [FormType 이름] + [업무 동사]
```

### 다국어 매핑

구현 레이어별로 일관된 네이밍을 유지합니다:

| 레이어 | 이름 | 언어 |
|--------|------|------|
| FormType | 품질검사의뢰서 | 한글 |
| Frontend (화면) | 품질검사의뢰서 | 한글 |
| API Endpoint | /qc-requests | 영문 |
| Database | qc_requests | 영문 snake_case |
| Code (변수) | qcRequest | 영문 camelCase |
| Code (클래스) | QcRequest | 영문 PascalCase |

**영문 약어 가이드**:

| 한글 | 영문 | 약어 |
|------|------|------|
| 품질검사 | Quality Check | qc |
| 작업 | Work | work |
| 자재 | Material | mat |
| 설비 | Equipment | eq |
| 불량 | Defect | def |
| 의뢰 | Request | req |
| 보고 | Report | rpt |

---

## 5. 데이터 모델 도출

### 핵심 원리

> **Form의 Section-Field 구조를 보면 데이터 모델이 보인다.**

추상화 없이도, Form의 구조만 있으면 ERD가 자연스럽게 도출됩니다.

### 사상 규칙 (Form → Record)

| Form 구조 | 데이터 모델 | 설명 |
|-----------|-------------|------|
| Main Section | 메인 테이블 | `qc_requests` |
| Child Section | 자식 테이블 (1:N) | `qc_request_items` |
| Reference Section | FK (참조키) | `production_plan_id FK` |
| Attachment Section | 파일 테이블 (1:N) | `qc_request_attachments` |

| Field 타입 | DB 컬럼 타입 | 예시 |
|-----------|-------------|------|
| Text | VARCHAR/TEXT | `document_no VARCHAR(50)` |
| Numeric | INTEGER/DECIMAL | `quantity INTEGER` |
| Date/Time | DATE/TIMESTAMP | `created_at TIMESTAMP` |
| Selection (Single) | FK | `product_id FK` |
| Selection (Multiple) | 1:N 테이블 | `qc_items` |
| Boolean | BOOLEAN | `is_urgent BOOLEAN` |

### 실전 예시

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

### 정규화 자동 적용

Section-Field 구조가 명확하면 정규화는 자연스럽게 따라옵니다:

- **1NF**: Child Section 발견 → 별도 Entity로 분리 → 원자성 자동 달성
- **2NF**: Selection Field 발견 → FK로 마스터 참조 → 부분 종속 제거
- **3NF**: Reference Section 발견 → FK로 연결 → 이행 종속 제거

### 전통 방식 vs Formology

| | 전통 데이터 모델링 | Formology |
|---|---|---|
| 소요 시간 | 5주 | 2시간 30분 |
| 현업 이해 | 40% | 95% |
| 과정 | Entity → Attribute → 관계 → 검증 | Form → Section/Field → Entity/Attribute → Record |
| 접근 | Top-down (추상화부터) | Bottom-up (구체적 Form부터) |

### 검증 방법

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

---

**Formology** — *Workflow First, Ontology Follows*
