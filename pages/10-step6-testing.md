# 10. STEP 6 · Testing — DB 없이 비즈니스 규칙 검증하기

`InMemoryRepository`로 Use Case를 테스트하면 서버 실행 없이 0.05초 만에 전체 규칙을 검증할 수 있습니다.

## 가짜 저장소 구현

```python
# tests/fakes.py
from application.repositories import BookRepository, MemberRepository, LoanRepository
from domain.entities import Book, Member, Loan


class InMemoryBookRepository(BookRepository):
    def __init__(self, books: list[Book] = None):
        self._books = {b.id: b for b in (books or [])}

    def get(self, book_id: int) -> Book:
        return self._books[book_id]

    def save(self, book: Book) -> Book:
        self._books[book.id] = book
        return book


class InMemoryLoanRepository(LoanRepository):
    def __init__(self):
        self._loans: dict[int, Loan] = {}
        self._next_id = 1

    def get(self, loan_id: int) -> Loan:
        return self._loans[loan_id]

    def save(self, loan: Loan) -> Loan:
        if loan.id is None:
            loan.id = self._next_id
            self._next_id += 1
        self._loans[loan.id] = loan
        return loan

    def find_active_loan_by_book(self, book_id: int):
        return next(
            (l for l in self._loans.values()
             if l.book_id == book_id and l.returned_at is None),
            None
        )

    def list_by_member(self, member_id: int) -> list[Loan]:
        return [l for l in self._loans.values() if l.member_id == member_id]
```

## Use Case 테스트 예시

```python
# tests/test_use_cases.py
import pytest
from datetime import date
from application.use_cases import BorrowBookUseCase, ExtendLoanUseCase
from domain.entities import Book, Member
from domain.exceptions import BookNotAvailableError, MemberCannotBorrowError
from tests.fakes import InMemoryBookRepository, InMemoryMemberRepository, InMemoryLoanRepository


@pytest.fixture
def repos():
    book_repo = InMemoryBookRepository([
        Book(id=1, title="클린 아키텍처", author="마틴", total_copies=1, available_copies=1),
    ])
    member_repo = InMemoryMemberRepository([Member(id=1, name="김철수")])
    loan_repo = InMemoryLoanRepository()
    return book_repo, member_repo, loan_repo


def test_borrow_fails_when_no_copies(repos):
    book_repo, member_repo, loan_repo = repos
    book_repo.get(1).available_copies = 0
    use_case = BorrowBookUseCase(book_repo, member_repo, loan_repo)

    with pytest.raises(BookNotAvailableError):
        use_case.execute(book_id=1, member_id=1)


def test_extend_twice_succeeds(repos):
    book_repo, member_repo, loan_repo = repos
    borrow_uc = BorrowBookUseCase(book_repo, member_repo, loan_repo)
    extend_uc = ExtendLoanUseCase(loan_repo)

    loan = borrow_uc.execute(book_id=1, member_id=1, today=date(2026, 1, 1))
    extend_uc.execute(loan_id=loan.id, today=date(2026, 1, 5))
    extended = extend_uc.execute(loan_id=loan.id, today=date(2026, 1, 6))

    assert extended.extension_count == 2
```

## Entity 단위 테스트 예시

```python
# tests/test_entities.py
from datetime import date, timedelta
import pytest
from domain.entities import Loan
from domain.exceptions import LoanExtensionError


def test_extend_adds_7_days():
    loan = Loan.new(book_id=1, member_id=1, today=date(2026, 1, 1))
    original_due = loan.due_at
    loan.extend(as_of=date(2026, 1, 5))
    assert loan.due_at == original_due + timedelta(days=7)
    assert loan.extension_count == 1


def test_extend_third_time_raises():
    loan = Loan.new(book_id=1, member_id=1, today=date(2026, 1, 1))
    loan.extend(as_of=date(2026, 1, 5))
    loan.extend(as_of=date(2026, 1, 6))
    with pytest.raises(LoanExtensionError):
        loan.extend(as_of=date(2026, 1, 7))
```

## 테스트 실행

```bash
pytest -v
# 20개 테스트, 약 0.05초
```

DB를 띄우거나 서버를 실행하지 않아도 됩니다. 이것이 클린 아키텍처가 주는 가장 큰 실익입니다.
