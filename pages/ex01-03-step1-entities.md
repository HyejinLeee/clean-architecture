# 05. STEP 1 · Entities — 핵심 규칙을 메서드로 표현하기

`Book.borrow_copy()`, `Member.assert_can_borrow()`, `Loan.extend()` — 각 규칙을 해당 Entity의 메서드로 캡슐화합니다.

```python
# domain/entities.py
@dataclass
class Book:
    id: int
    title: str
    author: str
    total_copies: int
    available_copies: int

    def borrow_copy(self) -> None:
        if self.available_copies <= 0:
            raise BookNotAvailableError(
                f"'{self.title}'의 대출 가능한 사본이 없습니다"
            )
        self.available_copies -= 1

    def return_copy(self) -> None:
        if self.available_copies >= self.total_copies:
            raise ValueError("모든 사본이 이미 반납된 상태입니다")
        self.available_copies += 1


@dataclass
class Member:
    id: Optional[int]
    name: str
    active_loans_count: int = 0
    has_overdue_loan: bool = False
    MAX_ACTIVE_LOANS: int = field(default=5, repr=False, compare=False)

    def assert_can_borrow(self) -> None:
        if self.has_overdue_loan:
            raise MemberCannotBorrowError(f"{self.name} 회원은 연체 중이라 대출할 수 없습니다")
        if self.active_loans_count >= self.MAX_ACTIVE_LOANS:
            raise MemberCannotBorrowError(
                f"{self.name} 회원은 최대 대출 권수({self.MAX_ACTIVE_LOANS}권)를 초과했습니다"
            )


@dataclass
class Loan:
    id: Optional[int]
    book_id: int
    member_id: int
    borrowed_at: date
    due_at: date
    returned_at: Optional[date] = None
    extension_count: int = 0

    LOAN_PERIOD_DAYS: int = field(default=14, repr=False, compare=False)
    EXTENSION_DAYS: int = field(default=7, repr=False, compare=False)
    MAX_EXTENSIONS: int = field(default=2, repr=False, compare=False)

    @staticmethod
    def new(book_id: int, member_id: int, today: date) -> "Loan":
        return Loan(
            id=None,
            book_id=book_id,
            member_id=member_id,
            borrowed_at=today,
            due_at=today + timedelta(days=Loan.LOAN_PERIOD_DAYS),
        )

    def extend(self, as_of: date) -> None:
        if self.extension_count >= self.MAX_EXTENSIONS:
            raise LoanExtensionError(f"최대 {self.MAX_EXTENSIONS}회까지 연장 가능합니다")
        if self.is_overdue(as_of):
            raise LoanExtensionError("연체 중인 대출은 연장할 수 없습니다")
        self.due_at = self.due_at + timedelta(days=self.EXTENSION_DAYS)
        self.extension_count += 1
```

## 핵심 설계 원칙

- **순수 Python만 사용**: `from dataclasses import dataclass`만 import합니다. FastAPI, SQLAlchemy가 전혀 없습니다.
- **규칙은 메서드 안에**: `borrow_copy()`가 재고 검사와 차감을 함께 수행합니다. 외부에서 직접 `available_copies -= 1`을 쓰지 않습니다.
- **예외로 실패를 표현**: `if` 분기가 아닌 `raise`로 실패를 명확히 전달합니다.
- **상수는 필드로**: `MAX_EXTENSIONS = 2`를 하드코딩 대신 `field()`로 정의해 서브클래스에서 오버라이드 가능합니다.
