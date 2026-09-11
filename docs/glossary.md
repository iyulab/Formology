# 용어 사전

**이 문서가 용어 정본입니다.** 다른 문서에서 다른 표기가 보이면 오류입니다.

---

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

Document가 **데이터베이스에 저장된 형태**. 정의 섹션 → Entity(테이블), Field → Attribute(컬럼), 투영 섹션 → 기존 Entity로의 FK 또는 값 복사.

```
[Main Section]        → main_table
  Field: 문서번호    → column: document_id
  Field: 제품명 (FK) → column: product_id (FK)
[Child Section] (1:N) → child_table
  Field: 항목        → column: item_name
```

### 기록관리 표준과의 대응 (ISO 15489 · ISO 23081)

이 용어 체계는 기록관리(records management) 표준과 **축이 다릅니다.** 표준은 정보가 *확정되어 증거로 유지되는가*로 document와 record를 가르고, Formology는 *진술인가 저장 형태인가*로 Document와 Record를 가릅니다. 그래서 이름은 겹치되 뜻은 엇갈립니다 — 기록관리 독자는 아래 표로 읽으십시오.

| Formology | ISO 15489 / ISO 23081 | 비고 |
|---|---|---|
| **Document** — 확정된 진술, 고쳐 쓰지 않음, 감사 대상 | **record** | 표준의 *document*는 아직 확정되지 않은 정보입니다. Formology의 초안 상태 Document가 그쪽에 가깝습니다 |
| **Record** — Document가 DB에 저장된 형태 | 대응어 없음 | 표준은 저장 형태를 별도 개념으로 두지 않습니다 |
| 기재 주체 ([방법론](methodology.md#기재-주체--누가-이-칸을-채우는가)) | **agent** | |
| 흐름 ([방법론](methodology.md#2-문서의-흐름)) | **business** (업무 활동) | |
| **존재 근거(mandate)** — [아래](#존재-근거-mandate--시범) | **mandate** | ISO 23081-2의 네 엔티티 중 Formology에 없던 하나 |

이름을 표준에 맞춰 바꾸지 않습니다. Formology의 `Document`/`Record`는 진술과 저장이라는 자기 축의 이름이고, 표준의 이름을 빌리면 `Record`가 갈 곳이 없습니다.

### 구조 분해

#### Section (섹션)

Form을 구성하는 **의미 단위 그룹**. 엔티티 경계를 결정하는 기본 단위.

**4가지 Section 유형**:

| 유형 | 무리 | 개념 | 원본 수정 | DB 구현 | 사용 예시 |
|------|------|------|-----------|---------|-----------|
| **Main Section** | 정의 | 핵심 정보 섹션 | - | 주 테이블 (**새 엔티티**) | 문서 본문 |
| **Child Section** | 정의 | 1:N 종속 섹션 | - | 자식 테이블 (**새 엔티티**, parent FK) | 자재 내역, 검사 항목 |
| **Reference Section** | 투영 | 살아있는 링크 | 자동 반영 | 기존 엔티티로의 FK | 승인/처리 문서 |
| **Attachment Section** | 투영 | 고정된 사본 | 영향 없음 | 복제 테이블/JSON | 증빙/계약 문서 |

**선택 기준**: 실시간 동기화 필요 → Reference Section. 과거 시점 고정 필요 → Attachment Section.

**정의와 투영의 구분**: 정의 섹션은 그 엔티티의 **출처**입니다 — 테이블이 이 서식 때문에 존재합니다. 투영 섹션은 이미 있는 엔티티에서 **그 맥락에 필요한 속성만 골라 옵니다.** 무엇을 골랐는가가 도메인 지식이며, 이것은 ERD에 적을 자리가 없는 정보입니다 ([방법론](methodology.md#정의하는-섹션과-투영하는-섹션)).

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

### 존재 근거 (mandate) — 시범

모든 FormType은 **존재 근거** 하나를 적습니다: 이 서식이 있어야 하는 이유. 값은 넷 중 하나입니다.

| 존재 근거 | 뜻 | 예시 |
|---|---|---|
| 법령 | 법·규정이 작성을 요구한다 | 산업안전 점검일지 |
| 계약 | 거래 상대와의 약속이 요구한다 | 거래명세서 |
| 내규 | 조직이 스스로 정한 규칙이 요구한다 | 휴가신청서 |
| 관행 | 아무도 정하지 않았는데 쓰고 있다 | — |

**관행이 답이면 그 서식은 걷어낼 후보입니다.** [철학](philosophy.md)이 말하는 "아무도 걷어내지 않은 서식"이 정확히 이 칸이 비는 서식이고, 워크숍의 "안 쓰면 무슨 일이 생깁니까"가 이 칸을 채우는 질문입니다. 기록관리 표준(ISO 23081-2)이 record의 메타데이터에 mandate를 두는 이유도 같습니다. 이 슬롯은 시범입니다 — [분류의 세 축](methodology.md#분류의-세-축)을 늘리는 것이 아니라 FormType에 붙는 속성 하나입니다.

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

**FormType 정의**: 현업 실제 이름? 접미사 명확? 업무 역할 명확? 중복 없음? 존재 근거(mandate) 적었는가 — 관행이면 걷어낼 후보?

**Form 설계**: Section 유형 명확? Main Section 1개 이상? Child 1:N 명확? Reference=실시간? Attachment=과거 고정? Field 유형 적절? Selection=FK?

**Record 사상**: Section→Entity 명확? Field→Attribute 명확? FK 정의? 1:N parent_id? snake_case? 필수/선택 제약?

---

---

**Formology** — *Workflow First, Ontology Follows*
