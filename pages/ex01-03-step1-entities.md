# 05. STEP 1 · Entities — 핵심 규칙을 메서드로 표현하기

각 규칙을 해당 Entity의 메서드로 캡슐화합니다.

## 먼저 큰 그림부터

코드를 보기 전에, 우리가 만들 Entity 세 개가 각각 **어떤 데이터를 가지고(속성)**, **무엇을 할 수 있는지(행동)** 개요를 살펴봅니다.

한 문장으로 요약하면 이렇습니다.

- **Book(도서)** 은 `id·제목·저자·총 권수·대출 가능 권수`를 가지고, **빌려주고(borrow_copy) 반납받는(return_copy)** 일을 합니다.
- **Member(회원)** 은 `id·이름·현재 대출 권수·연체 여부`를 가지고, **대출이 가능한지 스스로 검사(assert_can_borrow)** 하고 대출/반납 시 권수를 갱신합니다.
- **Loan(대출 기록)** 은 `누가·어떤 책을·언제 빌려서·언제까지인지`를 가지고, **연장(extend)·연체 판정(is_overdue)·반납 처리(mark_returned)** 를 합니다.

각 규칙이 왜 그 Entity의 메서드로 들어가는지 염두에 두고, 이제 실제 코드를 봅니다.

```python
# domain/entities.py
@dataclass
class Book:
    id: int
    title: str
    author: str
    total_copies: int
    available_copies: int

    def has_available_copy(self) -> bool:
        return self.available_copies > 0

    def borrow_copy(self) -> None:
        """사본 1권을 대출 처리한다. 재고가 없으면 예외를 던진다."""
        if not self.has_available_copy():
            raise BookNotAvailableError(f"'{self.title}'의 대출 가능한 사본이 없습니다")
        self.available_copies -= 1

    def return_copy(self) -> None:
        """사본 1권을 반납 처리한다."""
        if self.available_copies >= self.total_copies:
            raise ValueError("모든 사본이 이미 반납된 상태입니다")
        self.available_copies += 1


@dataclass
class Member:
    id: int
    name: str
    active_loans_count: int = 0
    has_overdue_loan: bool = False

    MAX_ACTIVE_LOANS = 5

    def assert_can_borrow(self) -> None:
        if self.has_overdue_loan:
            raise MemberCannotBorrowError(f"{self.name} 회원은 연체 중이라 대출할 수 없습니다")
        if self.active_loans_count >= self.MAX_ACTIVE_LOANS:
            raise MemberCannotBorrowError(
                f"{self.name} 회원은 최대 대출 권수({self.MAX_ACTIVE_LOANS}권)를 초과했습니다")

    def register_borrow(self) -> None:
        self.assert_can_borrow()
        self.active_loans_count += 1

    def register_return(self) -> None:
        self.active_loans_count = max(0, self.active_loans_count - 1)



@dataclass
class Loan:
    id: int
    book_id: int
    member_id: int
    borrowed_at: date
    due_at: date
    returned_at: Optional[date] = None
    extension_count: int = 0

    LOAN_PERIOD_DAYS = 14
    EXTENSION_DAYS = 7
    MAX_EXTENSIONS = 2

    @staticmethod
    def new(book_id: int, member_id: int, today: date) -> "Loan":
        return Loan(
            id=None,
            book_id=book_id,
            member_id=member_id,
            borrowed_at=today,
            due_at=today + timedelta(days=Loan.LOAN_PERIOD_DAYS),
        )

    def is_overdue(self, as_of: date) -> bool:
        return self.returned_at is None and as_of > self.due_at

    def mark_returned(self, as_of: date) -> None:
        if self.returned_at is not None:
            raise LoanAlreadyReturnedError("이미 반납된 대출입니다")
        self.returned_at = as_of

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
