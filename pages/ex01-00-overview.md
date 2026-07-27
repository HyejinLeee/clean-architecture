# 예제 1. 도서 대출 관리 시스템

클린 아키텍처를 처음 적용하기에 좋은 예제입니다. 규칙이 명확하고, Entity 간 관계가 단순하며, 실생활에서 익숙한 도메인이라 개념을 직관적으로 이해할 수 있습니다.

## 기술 스택

| 역할 | 선택 |
|---|---|
| 언어 | Python 3.11+ |
| API 프레임워크 | FastAPI |
| ORM | SQLAlchemy 2.0 |
| DB | SQLite |
| 테스트 | pytest |

## 시스템 개요

도서관 사서가 사용하는 간단한 대출 관리 API입니다.

- 회원이 책을 **대출**하고 **반납**할 수 있습니다
- 대출 기간은 14일이며, 연체 전에 한해 **최대 2회(각 7일) 연장** 가능합니다
- 연체 중이거나 5권 이상 대출 중인 회원은 추가 대출 불가합니다

## 이 예제에서 배우는 것

- `@dataclass`로 Entity를 순수 Python으로 표현하는 방법
- Use Case가 여러 Entity를 올바른 순서로 조율하는 방식
- `ABC`로 Repository 인터페이스를 정의하고 DIP를 달성하는 방법
- SQLAlchemy ORM 모델과 domain Entity를 완전히 분리하는 이유
- `InMemoryRepository`로 DB 없이 비즈니스 규칙을 빠르게 테스트하는 방법

## 전체 흐름

```
비즈니스 규칙 정의
      ↓
도메인 모델 설계 (Entity 3개: Book, Member, Loan)
      ↓
STEP 1. Entities    — 규칙을 메서드로 캡슐화
STEP 2. Use Cases   — 시나리오를 하나의 흐름으로 조립
STEP 3. Interface   — Repository 추상화 (DIP)
STEP 4. Adapters    — SQLAlchemy 구현체 + Pydantic 스키마
STEP 5. Frameworks  — FastAPI 엔드포인트 조립
STEP 6. Testing     — DB 없이 20개 테스트 통과
```
