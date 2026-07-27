# 06. STEP 2 · Use Cases — 대출 시나리오를 하나의 흐름으로 조립하기

`BorrowBookUseCase`는 Member 검증 → Book 재고 차감 → Loan 생성 순서로 Entity들을 조율합니다.

## 처리 흐름

```
1. 회원 검증         2. 재고 차감          3. 대출 생성         4. 저장
assert_can_borrow() → book.borrow_copy() → Loan.new(...) → repo.save(...)
```

```python
# application/use_cases.py
@dataclass
class BorrowBookUseCase:
    book_repo: BookRepository
    member_repo: MemberRepository
    loan_repo: LoanRepository

    def execute(self, book_id: int, member_id: int, today: date = None) -> Loan:
        today = today or date.today()
        book = self.book_repo.get(book_id)
        member = self.member_repo.get(member_id, today)
        member.assert_can_borrow()        # 회원 검증
        book.borrow_copy()                # 재고 차감
        loan = Loan.new(book_id, member_id, today)
        return self.loan_repo.save(loan)


@dataclass
class ReturnBookUseCase:
    book_repo: BookRepository
    member_repo: MemberRepository
    loan_repo: LoanRepository

    def execute(self, loan_id: int, today: date = None) -> Loan:
        today = today or date.today()
        loan = self.loan_repo.get(loan_id)
        loan.mark_returned(today)
        book = self.book_repo.get(loan.book_id)
        book.return_copy()
        member = self.member_repo.get(loan.member_id, today)
        member.register_return()
        self.loan_repo.save(loan)
        self.book_repo.save(book)
        self.member_repo.save(member)
        return loan


@dataclass
class ExtendLoanUseCase:
    loan_repo: LoanRepository

    def execute(self, loan_id: int, today: date = None) -> Loan:
        today = today or date.today()
        loan = self.loan_repo.get(loan_id)
        loan.extend(as_of=today)
        return self.loan_repo.save(loan)
```

## 핵심 설계 원칙

- **Use Case는 조율자**: 직접 규칙을 구현하지 않고, Entity 메서드를 올바른 순서로 호출합니다.
- **Repository 인터페이스에만 의존**: `book_repo`가 실제로 SQLite인지, 인메모리인지 알지 못합니다. (DIP 원칙)
- **`today` 주입**: 시간 의존적 로직을 테스트하기 위해 날짜를 외부에서 주입받습니다. `date.today()`를 내부에서 호출하면 테스트가 불가능해집니다.
