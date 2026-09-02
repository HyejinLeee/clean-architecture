# 1-09. STEP 6 · Testing — DB 없이 비즈니스 규칙 검증하기

`InMemoryRepository`로 Use Case를 테스트하면 서버 실행 없이 0.05초 만에 전체 규칙을 검증할 수 있습니다.

## 개요

코드를 보기 전에, 테스트를 어떻게 구성하는지 살펴봅니다.

- **Entity 단위 테스트** — `Loan.extend()` 같은 개별 규칙을 Entity만 만들어 직접 검증합니다. (준비물이 필요 없는 가장 단순한 테스트)
- **가짜 저장소(`InMemoryRepository`)** — 실제 DB 대신 딕셔너리로 동작하는 Repository를 만들어 Use Case에 주입합니다.
- **Use Case 테스트** — 대출·연장 같은 시나리오를 가짜 저장소로 실행해 규칙(재고 부족·연장 한도 등)을 검증합니다.
- **실행** — `pytest -v`로 DB·서버 없이 전체를 빠르게 돌립니다.

가장 안쪽 레이어인 **Entity 테스트부터** 보고, 그다음 가짜 저장소가 필요한 **Use Case 테스트**로 넘어갑니다.

## Entity 단위 테스트 예시

**Entity**는 가장 안쪽 레이어로, `Book`·`Member`·`Loan` 같은 **순수 Python 객체**입니다. DB·프레임워크·저장소를 전혀 모르고 비즈니스 규칙만 담습니다(대출 기간 14일, 연장 최대 2회 등). 의존하는 것이 없으니 **가짜 저장소도 필요 없이**, 객체 하나만 만들어 메서드를 직접 호출해 규칙을 검증할 수 있습니다. 그래서 가장 단순한 이 테스트부터 봅니다.

아래는 `Loan.extend()`(연장) 규칙을 검증하는 예입니다.

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

Use Case는 여러 Entity와 저장소를 함께 조율하므로, 이를 테스트하려면 먼저 **가짜 저장소**가 필요합니다.

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

## 왜 `Mock`이 아니라 '가짜 저장소(Fake)'를 선택했나

저장소 자리에 끼울 수 있는 테스트 더블은 여러 가지입니다. 흔히 쓰는 `unittest.mock`의
`Mock` 대신, 이 프로젝트는 위처럼 **직접 구현한 `InMemoryRepository`(Fake)** 를 씁니다.

Fake를 택한 이유는 다음과 같습니다.

- **인터페이스를 진짜로 구현한다** — Fake는 STEP 3의 추상 `BookRepository`를 상속해 구현하므로, 인터페이스가 바뀌면 테스트에서 바로 드러납니다. 
- **상태를 가진다 — 시나리오가 그대로 이어진다** — `save()`로 저장한 대출을 반납 테스트에서 다시 `get()`으로 꺼내 쓰는 흐름이 자연스럽게 재현됩니다. 
- **결과를 검증한다 — 리팩터링에 강하다** — Fake 테스트는 `available_copies == 0`처럼 **결과 상태**를 확인하므로 내부 구현을 바꿔도 결과만 같으면 통과합니다. 
- **클린 아키텍처와 결이 맞는다** — STEP 3에서 Repository를 추상화(DIP)한 덕분에, 운영엔 `SqlBookRepository`·테스트엔 `InMemoryBookRepository`를 같은 자리에 끼우기만 하면 됩니다. Fake는 이 구조를 그대로 활용하는 방식입니다.

> **그럼 `Mock`은 언제?** 직접 구현할 수 없는 외부 경계(외부 HTTP API·이메일 발송·현재 시각·난수)를 잘라낼 때 유용합니다. 이 예제는 시간 의존성을 Mock 대신 `execute(today=...)`로 **값을 주입**해 풀었으므로, Mock이 필요한 지점이 없었습니다.

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

## 테스트 실행

```bash
pytest -v
# 20개 테스트, 약 0.05초
```

DB를 띄우거나 서버를 실행하지 않아도 됩니다. 이것이 클린 아키텍처가 주는 가장 큰 실익입니다.
