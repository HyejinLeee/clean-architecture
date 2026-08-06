# 08. STEP 4 · Adapters — SQLAlchemy로 실제 구현 만들기

ORM 모델과 domain Entity는 서로 다른 클래스입니다. Adapters 레이어가 둘 사이를 변환합니다.

## 먼저 큰 그림부터

코드를 보기 전에, Adapters 레이어가 만들어 낼 조각들이 각각 **어떤 역할을 하는지** 개요를 살펴봅니다.

한 문장으로 요약하면 이렇습니다.

- **ORM 모델(`BookModel`·`LoanModel` 등)** 은 DB 테이블과 1:1로 대응하는 SQLAlchemy 클래스입니다.
- **매핑 함수(`_loan_to_entity`)** 는 ORM 모델을 순수 domain Entity로 변환해, SQLAlchemy가 안쪽 레이어로 새어들지 않게 막습니다.
- **Repository 구현체(`SqlLoanRepository`)** 는 STEP 3의 추상 인터페이스를 실제 DB 조작으로 채웁니다.
- **Pydantic 스키마(`LoanResponse`)** 는 Entity를 API 응답(JSON) 형태로 바꿉니다.

Entity ↔ ORM ↔ API 스키마가 서로 다른 클래스로 분리돼 있다는 점을 염두에 두고, 이제 실제 코드를 봅니다.

## ORM 모델 정의

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
