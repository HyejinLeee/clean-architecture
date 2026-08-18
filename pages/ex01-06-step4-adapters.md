# 1-07. STEP 4 · Adapters — SQLAlchemy로 실제 구현 만들기

ORM 모델과 domain Entity는 서로 다른 클래스입니다. Adapters 레이어가 둘 사이를 변환합니다.

## 어댑터(Adapter)란?

어댑터는 **서로 다른 두 세계를 이어주는 변환기**입니다. 해외여행용 **전원 어댑터**가 한국 플러그를 유럽 콘센트에 맞춰주듯, 또는 **통역사**가 서로 다른 언어를 옮겨주듯이요.

클린 아키텍처에는 말이 안 통하는 두 세계가 있습니다.

- **안쪽(도메인·유스케이스)** — 순수한 `Book`·`Loan` 객체와 추상 인터페이스로만 이야기합니다. DB도 웹도 모릅니다.
- **바깥쪽(기술)** — SQLAlchemy 모델, SQLite 행, HTTP·JSON 같은 구체적인 기술로 이야기합니다.

이 둘은 **직접 대화할 수 없습니다** (안쪽이 바깥을 알면 의존성 규칙 위반). 그래서 **Adapters가 중간에서 양방향으로 통역**합니다.

```
     안쪽 (순수 도메인)            Adapters            바깥 (기술)
   ────────────────────       ──────────────      ────────────────────
    Book · Loan Entity     ◀──   변환 / 통역   ──▶   SQLAlchemy · SQLite
    Repository 인터페이스    ◀──   (양방향)      ──▶   FastAPI · JSON
```

구체적으로 이 레이어가 하는 통역은 세 가지입니다.

| 방향 | 무엇을 → 무엇으로 | 담당 |
|---|---|---|
| **밖 → 안** | DB 행(`BookModel`) → 순수 `Book` Entity | `_book_to_entity` |
| **안 → 밖 (DB)** | `Book` Entity → 저장용 `BookModel` (추상 Repository "약속"을 SQLAlchemy로 구현) | `SqlBookRepository` |
| **안 → 밖 (API)** | `Loan` Entity → JSON 응답 `LoanResponse` | `LoanResponse` |

덕분에 **SQLAlchemy·FastAPI 같은 기술이 안쪽으로 새어들지 않습니다** — 도메인·유스케이스 코드엔 `import sqlalchemy`조차 없습니다. 그래서 기술을 바꿔도(예: SQLite→PostgreSQL) **어댑터만 갈아끼우면** 되고, 도메인은 그대로입니다.

## 개요

앞서 본 통역을 실제로 해내려면, Adapters 레이어에서 만들 조각은 **네 가지**입니다.

- **ORM 모델** — DB 테이블과 1:1 대응 (`BookModel` 등)
- **매핑 함수** — ORM ↔ Entity 변환 (`_loan_to_entity` 등)
- **Repository 구현체** — 추상 인터페이스의 실제 DB 구현 (`SqlLoanRepository` 등)
- **Pydantic 스키마** — Entity → API 응답 JSON (`LoanResponse`)

이제 하나씩 실제 코드로 봅니다.

## ORM 모델 정의

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

## Repository 구현체

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

- **`_loan_to_entity`** — 조회한 `LoanModel`(ORM)을 순수 `Loan` Entity로 바꾸는 매핑 함수. 모든 메서드가 이걸 거쳐 **항상 Entity만 반환**합니다(ORM 객체를 바깥으로 내보내지 않음).
- **`get`** — id로 찾고, 없으면 `LoanNotFoundError`, 있으면 Entity로 변환해 반환합니다.
- **`save`** — id가 없으면 새 행을 INSERT(`add`), 있으면 변경분만 UPDATE한 뒤 `commit`합니다.
- **`find_active_loan_by_book`·`list_by_member`** — `query(...).filter(...)`로 조건 조회합니다. SQLAlchemy 문법은 이 클래스 안에서만 쓰이고, 결과는 다시 Entity로 변환됩니다.

즉 이 클래스가 **SQL·세션 같은 DB 세부를 전담**하므로, Use Case는 STEP 3의 인터페이스만 알면 되고 SQLAlchemy를 전혀 모릅니다.

> ### 💡 헷갈리기 쉬운 점 — `save()`는 구현마다 다르게 동작합니다
>
> Use Case는 대출 시 `book_repo.save(book)`·`member_repo.save(member)`도 호출하지만, **SQL 구현에서는 이 두 호출이 바뀐 값을 실제로 저장하지 않습니다.**
>
> | 호출 | SQL 구현에서 실제 동작 | 바뀐 값이 DB에 반영? |
> |---|---|---|
> | `loan_repo.save(loan)` | `loans`에 행 INSERT/UPDATE | ✅ 진짜 저장 |
> | `book_repo.save(book)` | `total_copies`만 반영 | ❌ `available_copies`는 컬럼이 없어 무시 |
> | `member_repo.save(member)` | 기존 회원이면 아무것도 안 함 | ❌ `active_loans_count`는 저장 안 함 |
>
> 재고·대출 권수는 저장하는 값이 아니라 `loans`에서 매번 계산하는 **파생값**이기 때문입니다. 그래서 대출로 인한 실제 DB 변화는 **`loans`에 행 하나 추가**뿐이고, 재고·권수는 다음 조회 때 자동으로 다시 계산됩니다.
>
> **그런데도 Use Case가 두 `save()`를 부르는 이유**는, Use Case가 "어떤 구현인지"를 몰라야 하기 때문입니다. 테스트용 `InMemoryRepository`처럼 객체를 통째로 저장하는 구현에서는 이 `save()`가 **반드시 필요**합니다(안 부르면 재고 −1이 사라짐). Use Case는 원칙대로 "바꿨으면 저장한다"만 지키고, 그 요청을 어떻게 이행할지(전부 저장 / 일부는 파생이라 무시)는 각 구현이 알아서 처리합니다.

## 엔티티 ↔ ORM 분리가 중요한 이유

`_loan_to_entity()` 같은 명시적 매핑 함수를 두면:
- SQLAlchemy 모델이 domain 레이어에 침투하지 않습니다
- ORM 스키마를 변경해도 Entity 코드는 그대로 유지됩니다
- 두 클래스의 필드명이 달라도 됩니다 (자유롭게 매핑 가능)

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
