# 1-09. STEP 6 · Testing — DB 없이 비즈니스 규칙 검증하기

`InMemoryRepository`로 Use Case를 테스트하면 서버 실행 없이 0.05초 만에 전체 규칙을 검증할 수 있습니다.

## 개요

코드를 보기 전에, 테스트를 어떻게 구성하는지 살펴봅니다.

- **가짜 저장소(`InMemoryRepository`)** — 실제 DB 대신 딕셔너리로 동작하는 Repository를 만들어 Use Case에 주입합니다.
- **Use Case 테스트** — 대출·연장 같은 시나리오를 가짜 저장소로 실행해 규칙(재고 부족·연장 한도 등)을 검증합니다.
- **Entity 단위 테스트** — `Loan.extend()` 같은 개별 규칙을 Entity만 만들어 직접 검증합니다.
- **실행** — `pytest -v`로 DB·서버 없이 전체를 빠르게 돌립니다.

추상 Repository 덕분에 DB 없이 테스트가 가능하다는 점을 염두에 두고, 이제 실제 코드를 봅니다.

## 가짜 저장소 구현

```python
# tests/fakes.py
from typing import Optional
from application.repositories import BookRepository, MemberRepository, LoanRepository
from domain.entities import Book, Member, Loan
from domain.exceptions import BookNotFoundError, LoanNotFoundError, MemberNotFoundError


class InMemoryBookRepository(BookRepository):
    def __init__(self, books: Optional[list[Book]] = None):
        self._books = {b.id: b for b in (books or [])}
        self._next_id = max(self._books.keys(), default=0) + 1

    def get(self, book_id: int) -> Book:
        if book_id not in self._books:
            raise BookNotFoundError(f"책 id={book_id}를 찾을 수 없습니다")
        return self._books[book_id]

    def save(self, book: Book) -> Book:
        if book.id is None:
            book.id = self._next_id
            self._next_id += 1
        self._books[book.id] = book
        return book


class InMemoryMemberRepository(MemberRepository):
    def __init__(self, members: Optional[list[Member]] = None):
        self._members = {m.id: m for m in (members or [])}
        self._next_id = max(self._members.keys(), default=0) + 1

    def get(self, member_id: int, today=None) -> Member:
        if member_id not in self._members:
            raise MemberNotFoundError(f"회원 id={member_id}를 찾을 수 없습니다")
        return self._members[member_id]

    def save(self, member: Member) -> Member:
        if member.id is None:
            member.id = self._next_id
            self._next_id += 1
        self._members[member.id] = member
        return member


class InMemoryLoanRepository(LoanRepository):
    def __init__(self):
        self._loans: dict[int, Loan] = {}
        self._next_id = 1

    def get(self, loan_id: int) -> Loan:
        if loan_id not in self._loans:
            raise LoanNotFoundError(f"대출 id={loan_id}를 찾을 수 없습니다")
        return self._loans[loan_id]

    def save(self, loan: Loan) -> Loan:
        if loan.id is None:
            loan.id = self._next_id
            self._next_id += 1
        self._loans[loan.id] = loan
        return loan

    def find_active_loan_by_book(self, book_id: int) -> Optional[Loan]:
        for loan in self._loans.values():
            if loan.book_id == book_id and loan.returned_at is None:
                return loan
        return None

    def list_by_member(self, member_id: int) -> list[Loan]:
        return [l for l in self._loans.values() if l.member_id == member_id]
```

## Use Case 테스트 예시

```python
# tests/test_use_cases.py
from datetime import date

import pytest

from application.use_cases import BorrowBookUseCase, ExtendLoanUseCase
from domain.entities import Book, Member
from domain.exceptions import BookNotAvailableError
from tests.fakes import InMemoryBookRepository, InMemoryLoanRepository, InMemoryMemberRepository


@pytest.fixture
def repos():
    book_repo = InMemoryBookRepository([
        Book(id=1, title="클린 아키텍처", author="마틴", total_copies=1, available_copies=1),
    ])
    member_repo = InMemoryMemberRepository([
        Member(id=1, name="김철수"),
    ])
    loan_repo = InMemoryLoanRepository()
    return book_repo, member_repo, loan_repo


class TestBorrowBookUseCase:
    def test_successful_borrow(self, repos):
        book_repo, member_repo, loan_repo = repos
        use_case = BorrowBookUseCase(book_repo, member_repo, loan_repo)

        loan = use_case.execute(book_id=1, member_id=1, today=date(2026, 1, 1))

        assert loan.id is not None
        assert book_repo.get(1).available_copies == 0
        assert member_repo.get(1).active_loans_count == 1

    def test_borrow_fails_when_no_copies_available(self, repos):
        book_repo, member_repo, loan_repo = repos
        book_repo.get(1).available_copies = 0
        use_case = BorrowBookUseCase(book_repo, member_repo, loan_repo)

        with pytest.raises(BookNotAvailableError):
            use_case.execute(book_id=1, member_id=1)


class TestExtendLoanUseCase:
    def test_extend_moves_due_date(self, repos):
        book_repo, member_repo, loan_repo = repos
        borrow_uc = BorrowBookUseCase(book_repo, member_repo, loan_repo)
        extend_uc = ExtendLoanUseCase(loan_repo)

        loan = borrow_uc.execute(book_id=1, member_id=1, today=date(2026, 1, 1))
        original_due = loan.due_at

        extended_loan = extend_uc.execute(loan_id=loan.id, today=date(2026, 1, 5))

        assert extended_loan.due_at > original_due
        assert extended_loan.extension_count == 1
```

## Entity 단위 테스트 예시

```python
# tests/test_entities.py
from datetime import date, timedelta

import pytest

from domain.entities import Loan
from domain.exceptions import LoanExtensionError


class TestLoan:
    def test_extend_adds_7_days(self):
        loan = Loan.new(book_id=1, member_id=1, today=date(2026, 1, 1))
        original_due = loan.due_at
        loan.extend(as_of=date(2026, 1, 5))
        assert loan.due_at == original_due + timedelta(days=7)
        assert loan.extension_count == 1

    def test_extend_third_time_raises(self):
        loan = Loan.new(book_id=1, member_id=1, today=date(2026, 1, 1))
        loan.extend(as_of=date(2026, 1, 5))
        loan.extend(as_of=date(2026, 1, 6))
        with pytest.raises(LoanExtensionError):
            loan.extend(as_of=date(2026, 1, 7))
```

## 테스트 실행

```bash
pytest -v
# 19개 테스트, 약 0.05초
```

DB를 띄우거나 서버를 실행하지 않아도 됩니다. 이것이 클린 아키텍처가 주는 가장 큰 실익입니다.
