# 03. 어떤 프로젝트에든 적용하는 방법

클린 아키텍처를 새 프로젝트에 도입할 때 따르는 공통 절차입니다. 도메인이 달라도 순서는 동일합니다.

## 적용 순서

```
1. 비즈니스 규칙 나열        → 규칙이 Entity의 메서드가 됩니다
2. Entity 설계               → 어떤 상태와 행동이 필요한가?
3. Use Case 정의             → 누가, 언제, 어떤 순서로 Entity를 조율하는가?
4. Repository 인터페이스 정의 → Use Case가 필요로 하는 데이터 접근 메서드는?
5. Repository 구현체 작성    → SQLAlchemy, 파일, 외부 API 등
6. Framework 조립            → FastAPI, CLI, 스케줄러 등
7. 테스트 작성               → InMemory 구현체로 DB 없이 검증
```

## Entity를 찾는 질문

> "이 비즈니스에서 **상태를 갖고, 스스로 규칙을 지키는** 개념은 무엇인가?"

| 도메인 | Entity 예시 |
|---|---|
| 도서 대출 | Book, Member, Loan |
| 쇼핑몰 | Product, Cart, Order |
| 예약 시스템 | Room, Guest, Reservation |
| HR 시스템 | Employee, Department, LeaveRequest |

상태(데이터)만 있고 규칙이 없다면 Entity가 아닌 단순 데이터 클래스일 수 있습니다.

## Use Case를 찾는 질문

> "사용자(또는 시스템)가 **하나의 목적으로** 수행하는 행동은 무엇인가?"

Use Case 하나 = 하나의 `execute()` 메서드. 여러 Entity를 조율하되, 규칙 자체는 구현하지 않습니다.

## Repository 인터페이스를 찾는 질문

> "Use Case가 데이터를 **어떻게 가져오고 저장**하는가?"

- `get(id)` — 단건 조회
- `save(entity)` — 저장 (신규/수정 통합)
- `find_by_xxx(...)` — 조건 조회
- `list_by_xxx(...)` — 목록 조회

구현 방식(SQL, NoSQL, 파일)은 이 단계에서 결정하지 않습니다.

## 파일 구조 템플릿

어떤 프로젝트든 동일한 구조에서 시작합니다.

```
project_name/
├── domain/
│   ├── entities.py       # 순수 Python, 외부 라이브러리 없음
│   └── exceptions.py     # DomainError 상속 계층
├── application/
│   ├── repositories.py   # ABC 추상 인터페이스
│   └── use_cases.py
├── adapters/
│   ├── orm_models.py
│   ├── repositories.py   # SQLAlchemy 구현체
│   └── schemas.py        # Pydantic
├── infrastructure/
│   └── db.py
├── main.py               # 유일한 조립 지점
└── tests/
    ├── fakes.py           # InMemory 구현체
    ├── test_entities.py
    └── test_use_cases.py
```
