# 06. STEP 2 · Use Cases — 대출 시나리오를 하나의 흐름으로 조립하기

각 사용자 시나리오를 하나의 Use Case로 조립합니다.

## 먼저 큰 그림부터

코드를 보기 전에, 우리가 만들 Use Case 네 개가 각각 **어떤 시나리오를**, **어떤 순서로 처리하는지** 개요를 살펴봅니다.

한 문장으로 요약하면 이렇습니다.

- **BorrowBookUseCase(대출)** 은 `회원 검증(assert_can_borrow) → 재고 차감(borrow_copy) → 대출 기록 생성(Loan.new) → loan·member·book 각각 저장` 순서로 대출을 완성합니다. 재고와 회원 권수가 바뀌므로 **반드시 각각 저장**해야 반영됩니다.
- **ReturnBookUseCase(반납)** 은 `반납 처리(mark_returned) → 재고 복구(return_copy) → 회원 권수 감소(register_return) → 각각 저장` 순서로 반납을 처리합니다.
- **ExtendLoanUseCase(연장)** 은 대출을 찾아 `연장(extend)`을 호출합니다. 이미 연장했거나 연체 중이면 예외가 발생합니다.
- **ListMemberLoansUseCase(대출 목록)** 은 상태를 바꾸지 않고 회원의 대출 목록을 **조회만** 합니다.

각 Use Case가 규칙을 직접 구현하지 않고 Entity 메서드를 올바른 순서로 호출한다는 점을 염두에 두고, 이제 실제 코드를 봅니다.

```python
# application/use_cases.py
@dataclass
class BorrowBookUseCase:
    book_repo: BookRepository
    member_repo: MemberRepository
    loan_repo: LoanRepository

    def execute(self, book_id: int, member_id: int, today: Optional[date] = None) -> Loan:
        today = today or date.today()

        # 1. 조회
        book = self.book_repo.get(book_id)
        member = self.member_repo.get(member_id, today)   # today 기준으로 연체 여부 계산

        # 2. 검증 (규칙 위반 시 예외 발생)
        member.assert_can_borrow()        # 연체·대출 한도
        book.borrow_copy()                # 재고

        # 3. 상태 변경
        loan = Loan.new(book_id=book.id, member_id=member.id, today=today)
        loan = self.loan_repo.save(loan)
        member.register_borrow()          # 회원 대출 권수 +1

        # 4. 저장 (바뀐 것을 각각 저장)
        self.member_repo.save(member)
        self.book_repo.save(book)
        return loan


@dataclass
class ReturnBookUseCase:
    book_repo: BookRepository
    member_repo: MemberRepository
    loan_repo: LoanRepository

    def execute(self, loan_id: int, today: Optional[date] = None) -> Loan:
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

    def execute(self, loan_id: int, today: Optional[date] = None) -> Loan:
        today = today or date.today()
        loan = self.loan_repo.get(loan_id)
        loan.extend(today)                # 이미 연장했거나 연체 중이면 예외
        return self.loan_repo.save(loan)


@dataclass
class ListMemberLoansUseCase:
    loan_repo: LoanRepository

    def execute(self, member_id: int) -> list[Loan]:
        # 조회만 하는 Use Case — 검증·상태 변경 없이 Repository에 위임
        return self.loan_repo.list_by_member(member_id)
```

## 핵심 설계 원칙

- **Use Case는 조율자**: 직접 규칙을 구현하지 않고, Entity 메서드를 올바른 순서로 호출합니다. 규칙(연체·재고·연장 한도)은 모두 Entity 안에 있고, Use Case는 "언제 무엇을" 호출할지 순서만 정합니다.
- **바꿨으면 저장한다**: 대출은 `loan`·`member`·`book` 세 객체가 모두 바뀌므로 각각 `repo.save()`로 저장합니다. 저장을 빠뜨리면 재고·대출 권수 변경이 사라집니다.
- **조회형 Use Case는 단순**: `ListMemberLoansUseCase`처럼 상태를 바꾸지 않는 경우 검증·저장 없이 Repository 호출만 위임합니다.
- **Repository 인터페이스에만 의존**: `book_repo`가 실제로 SQLite인지, 인메모리인지 알지 못합니다. (DIP 원칙)
- **`today` 주입**: 시간 의존적 로직을 테스트하기 위해 날짜를 외부에서 주입받습니다. `date.today()`를 내부에서 호출하면 테스트가 불가능해집니다.
