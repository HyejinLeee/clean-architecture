# 08. STEP 4 · Adapters — SQLAlchemy로 실제 구현 만들기

ORM 모델과 domain Entity는 서로 다른 클래스입니다. Adapters 레이어가 둘 사이를 변환합니다.

## ORM 모델 정의

```python
# adapters/orm_models.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import Integer, String, Date, ForeignKey

class Base(DeclarativeBase):
    pass

class BookModel(Base):
    __tablename__ = 'books'
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String)
    author: Mapped[str] = mapped_column(String)
    total_copies: Mapped[int] = mapped_column(Integer)
    available_copies: Mapped[int] = mapped_column(Integer)

class LoanModel(Base):
    __tablename__ = 'loans'
    id: Mapped[int] = mapped_column(primary_key=True)
    book_id: Mapped[int] = mapped_column(ForeignKey('books.id'))
    member_id: Mapped[int] = mapped_column(ForeignKey('members.id'))
    borrowed_at: Mapped[date] = mapped_column(Date)
    due_at: Mapped[date] = mapped_column(Date)
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
            raise LoanNotFoundError(f"Loan {loan_id} not found")
        return _loan_to_entity(model)

    def save(self, loan: Loan) -> Loan:
        if loan.id is None:
            model = LoanModel(
                book_id=loan.book_id,
                member_id=loan.member_id,
                borrowed_at=loan.borrowed_at,
                due_at=loan.due_at,
                returned_at=loan.returned_at,
                extension_count=loan.extension_count,
            )
            self.session.add(model)
            self.session.flush()
            loan.id = model.id
        else:
            model = self.session.get(LoanModel, loan.id)
            model.due_at = loan.due_at
            model.returned_at = loan.returned_at
            model.extension_count = loan.extension_count
        self.session.commit()
        return loan
```

## 엔티티 ↔ ORM 분리가 중요한 이유

`_loan_to_entity()` 같은 명시적 매핑 함수를 두면:
- SQLAlchemy 모델이 domain 레이어에 침투하지 않습니다
- ORM 스키마를 변경해도 Entity 코드는 그대로 유지됩니다
- 두 클래스의 필드명이 달라도 됩니다 (자유롭게 매핑 가능)

## Pydantic 스키마

```python
# adapters/schemas.py
class LoanResponse(BaseModel):
    id: int
    book_id: int
    member_id: int
    borrowed_at: date
    due_at: date
    returned_at: Optional[date] = None
    extension_count: int

    @staticmethod
    def from_entity(loan: Loan) -> "LoanResponse":
        return LoanResponse(
            id=loan.id,
            book_id=loan.book_id,
            member_id=loan.member_id,
            borrowed_at=loan.borrowed_at,
            due_at=loan.due_at,
            returned_at=loan.returned_at,
            extension_count=loan.extension_count,
        )
```
