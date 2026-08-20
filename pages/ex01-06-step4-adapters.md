# 1-07. STEP 4 · Adapters — SQLAlchemy로 실제 구현 만들기

ORM 모델과 domain Entity는 서로 다른 클래스입니다. Adapters 레이어가 둘 사이를 변환합니다.

## 어댑터(Adapter)란?

어댑터는 **STEP 3에서 정한 추상 인터페이스를 상속받아, 비어 있던 메서드를 실제 기술로 채운 구현체**입니다.


```python
# application/repositories.py  (STEP 3 — 약속만, 본문은 비어 있음)
class LoanRepository(ABC):
    @abstractmethod
    def get(self, loan_id: int) -> Loan: ...      # 어떻게는 아직 비어 있음
    @abstractmethod
    def save(self, loan: Loan) -> Loan: ...
```

STEP 4의 어댑터는 이 인터페이스를 **상속받아**(`class SqlLoanRepository(LoanRepository)`), 그 빈칸을 SQLAlchemy 로 구현합니다.

```python
# adapters/repositories.py  (STEP 4 — 상속받아 실제로 구현)
class SqlLoanRepository(LoanRepository):              # ← STEP 3 인터페이스 상속
    def get(self, loan_id: int) -> Loan:
        model = self.session.get(LoanModel, loan_id)  # 빈칸을 실제 DB 조회로 채움
        return _loan_to_entity(model)
    ...
```

즉 **약속(인터페이스)은 STEP 3, 실제 기술 구현(어댑터)은 STEP 4**입니다. 이렇게 나누면:

- **안쪽(유스케이스)** 은 `LoanRepository`(약속)에만 의존합니다. 구현이 SQLite인지 인메모리인지 모릅니다.
- 같은 약속을 다르게 구현하면 **갈아끼울 수 있습니다** — 운영엔 `SqlLoanRepository`, 테스트엔 `InMemoryLoanRepository`.

```
[STEP 3]  LoanRepository (ABC, 약속)
                 ▲ 상속받아 구현
[STEP 4]  SqlLoanRepository   ·   InMemoryLoanRepository   ·   PostgresLoanRepository ...
```

그래서 SQLite → PostgreSQL로 바꿔도 **어댑터만 새로 구현**하면 되고, 유스케이스 코드는 그대로입니다.

## 상세 구현 예시

크게 네 부분으로 나누어 구현을 설명합니다. 프로젝트 상황에 따라 여러 방법이 있을 수 있을 수 있으니 참고용으로 보시면 좋겠습니다.

- **ORM 모델** — DB 테이블과 1:1 대응 (`BookModel` 등)
- **매핑 함수** — ORM ↔ Entity 변환 (`_loan_to_entity` 등)
- **Repository 구현체** — 추상 인터페이스의 실제 DB 구현 (`SqlLoanRepository` 등)
- **Pydantic 스키마** — Entity → API 응답 JSON (`LoanResponse`)

이제 하나씩 실제 코드로 봅니다.

## ORM 모델

> **ORM(Object-Relational Mapping)이란?** DB의 **표(테이블·행)** 와 코드의 **객체(클래스·인스턴스)** 를 자동으로 이어주는 기술입니다. `SELECT`·`INSERT` 같은 SQL을 직접 쓰는 대신, **파이썬 객체를 다루면 라이브러리가 알아서 SQL로 번역**해 줍니다. 여기서는 SQLAlchemy를 씁니다.
>
> - **클래스 ↔ 테이블**: `BookModel` 클래스 = `books` 테이블
> - **속성 ↔ 컬럼**: `title` 속성 = `title` 컬럼
> - **인스턴스 ↔ 행(row)**: `BookModel(...)` 객체 하나 = 테이블의 한 줄
>
> 즉 아래 클래스들은 "이 파이썬 객체를 이런 테이블에 저장해줘"라고 SQLAlchemy에 알려주는 **설계도**입니다.

```python
# adapters/orm_models.py
from datetime import date
from sqlalchemy import Date, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from infrastructure.db import Base          # Base는 infrastructure에서 선언 (STEP 5)


class BookModel(Base):
    __tablename__ = "books"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(200), nullable=False)
    author: Mapped[str] = mapped_column(String(200), nullable=False)
    total_copies: Mapped[int] = mapped_column(Integer, nullable=False)
    # available_copies는 저장하지 않음 — loans 테이블에서 실시간 계산


class MemberModel(Base):
    __tablename__ = "members"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(200), nullable=False)
    # active_loans_count, has_overdue_loan도 저장하지 않고 loans에서 계산


class LoanModel(Base):
    __tablename__ = "loans"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    book_id: Mapped[int] = mapped_column(ForeignKey("books.id"), nullable=False)
    member_id: Mapped[int] = mapped_column(ForeignKey("members.id"), nullable=False)
    borrowed_at: Mapped[date] = mapped_column(Date, nullable=False)
    due_at: Mapped[date] = mapped_column(Date, nullable=False)
    returned_at: Mapped[date | None] = mapped_column(Date, nullable=True)
    extension_count: Mapped[int] = mapped_column(Integer, default=0, nullable=False)
```

파이썬 클래스를 DB 테이블에 매핑한 것으로, 각 모델이 곧 하나의 테이블(`books`·`members`·`loans`)입니다.

- **`Mapped[타입]` + `mapped_column(...)`** — 컬럼 하나를 선언합니다. `primary_key`(PK)·`ForeignKey`(FK)·`nullable`(NULL 허용) 같은 제약을 함께 지정합니다.
- **파생값은 컬럼이 없습니다** — 주석처럼 `available_copies`·`active_loans_count`·`has_overdue_loan`은 저장하지 않고 `loans`에서 계산합니다. 그래서 `books`엔 제목·저자·총권수만, `members`엔 이름만 둡니다.
- **순수 Entity와는 별개 클래스** — 이 `BookModel`(SQLAlchemy)은 STEP 1의 `Book`(dataclass)과 다릅니다. DB 기술 표현이며, 뒤의 매핑 함수가 순수 Entity로 바꿔줍니다.

## 매핑 함수

```python
# adapters/repositories.py
def _loan_to_entity(m: LoanModel) -> Loan:
    return Loan(
        id=m.id,
        book_id=m.book_id,
        member_id=m.member_id,
        borrowed_at=m.borrowed_at,
        due_at=m.due_at,
        returned_at=m.returned_at,
        extension_count=m.extension_count,
    )
```

ORM 모델(`LoanModel`)을 순수 `Loan` Entity로 바꿔주는 함수입니다.

### 엔티티 ↔ ORM 분리가 중요한 이유

`_loan_to_entity()` 같은 명시적 매핑 함수를 두면:

- SQLAlchemy 모델이 domain 레이어에 침투하지 않습니다
- ORM 스키마를 변경해도 Entity 코드는 그대로 유지됩니다
- 두 클래스의 필드명이 달라도 됩니다 (자유롭게 매핑 가능)

## Repository 구현체

```python
# adapters/repositories.py
class SqlLoanRepository(LoanRepository):
    def __init__(self, session: Session):
        self.session = session

    def get(self, loan_id: int) -> Loan:
        model = self.session.get(LoanModel, loan_id)
        if model is None:
            raise LoanNotFoundError(f"대출 id={loan_id}를 찾을 수 없습니다")
        return _loan_to_entity(model)

    def save(self, loan: Loan) -> Loan:
        model = self.session.get(LoanModel, loan.id) if loan.id else None
        if model is None:                       # 새 대출 → INSERT
            model = LoanModel(
                book_id=loan.book_id, member_id=loan.member_id,
                borrowed_at=loan.borrowed_at, due_at=loan.due_at,
                returned_at=loan.returned_at, extension_count=loan.extension_count,
            )
            self.session.add(model)
        else:                                    # 기존 대출 → 변경분만 UPDATE
            model.due_at = loan.due_at
            model.returned_at = loan.returned_at
            model.extension_count = loan.extension_count
        self.session.commit()
        self.session.refresh(model)
        return _loan_to_entity(model)

    def find_active_loan_by_book(self, book_id: int) -> Optional[Loan]:
        model = (
            self.session.query(LoanModel)
            .filter(LoanModel.book_id == book_id, LoanModel.returned_at.is_(None))
            .first()
        )
        return _loan_to_entity(model) if model else None

    def list_by_member(self, member_id: int) -> list[Loan]:
        models = self.session.query(LoanModel).filter(LoanModel.member_id == member_id).all()
        return [_loan_to_entity(m) for m in models]
```

STEP 3의 추상 `LoanRepository`를 SQLAlchemy로 **실제 구현**한 것입니다. `self.session`으로 DB를 조작합니다.

- **`get`** — id로 찾고, 없으면 `LoanNotFoundError`, 있으면 `_loan_to_entity`로 변환해 반환합니다.
- **`save`** — id가 없으면 새 행을 INSERT(`add`), 있으면 변경분만 UPDATE한 뒤 `commit`합니다.
- **`find_active_loan_by_book`·`list_by_member`** — `query(...).filter(...)`로 조건 조회합니다. SQLAlchemy 문법은 이 클래스 안에서만 쓰이고, 결과는 **항상 Entity로 변환**해 돌려줍니다.

즉 이 클래스가 **SQL·세션 같은 DB 세부를 전담**하므로, Use Case는 STEP 3의 인터페이스만 알면 되고 SQLAlchemy를 전혀 모릅니다.

## Pydantic 스키마

```python
# adapters/schemas.py
class BorrowRequest(BaseModel):
    book_id: int
    member_id: int


class LoanResponse(BaseModel):
    id: int
    book_id: int
    member_id: int
    borrowed_at: date
    due_at: date
    returned_at: Optional[date] = None
    extension_count: int

    @staticmethod
    def from_entity(loan) -> "LoanResponse":
        return LoanResponse(
            id=loan.id, book_id=loan.book_id, member_id=loan.member_id,
            borrowed_at=loan.borrowed_at, due_at=loan.due_at,
            returned_at=loan.returned_at, extension_count=loan.extension_count,
        )
```

API 경계에서 쓰는 입출력 형식입니다.

- **`LoanResponse`** — 순수 `Loan` Entity를 **API 응답(JSON)** 으로 바꿉니다. `from_entity()`가 Entity를 받아 JSON 필드로 옮깁니다.
- **`BorrowRequest`** — 반대로 **들어오는 요청 JSON을 검증**해 받습니다.
- 도메인이 JSON·HTTP를 모르도록, 이 변환을 어댑터에서 담당합니다.
