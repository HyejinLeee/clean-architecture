# 06. STEP 2 · Use Cases — 대출 시나리오를 하나의 흐름으로 조립하기

각 사용자 시나리오를 하나의 Use Case로 조립합니다.

## 먼저 큰 그림부터

우리가 만들 Use Case 네 개가 각각 **어떤 시나리오를**, **어떤 순서로 처리하는지** 개요를 살펴봅니다.

네가지의 use case : 대출, 반납, 연장, 대출 목록 조회

- **BorrowBookUseCase(대출)** 은 `회원 검증(assert_can_borrow) → 재고 차감(borrow_copy) → 대출 기록 생성(Loan.new) → loan·member·book 각각 저장` 순서로 대출을 완성합니다. 재고와 회원 권수가 바뀌므로 **반드시 각각 저장**해야 반영됩니다.
- **ReturnBookUseCase(반납)** 은 `반납 처리(mark_returned) → 재고 복구(return_copy) → 회원 권수 감소(register_return) → 각각 저장` 순서로 반납을 처리합니다.
- **ExtendLoanUseCase(연장)** 은 대출을 찾아 `연장(extend)`을 호출합니다. 이미 연장했거나 연체 중이면 예외가 발생합니다.
- **ListMemberLoansUseCase(대출 목록)** 은 상태를 바꾸지 않고 회원의 대출 목록을 **조회만** 합니다.

각 Use Case가 규칙을 직접 구현하지 않고 Entity 메서드를 올바른 순서로 호출한다는 점을 염두에 두고, 이제 실제 코드를 봅니다.

> 아래 코드에서 각 Use Case가 의존하는 `BookRepository`, `MemberRepository`, `LoanRepository`는 데이터를 저장·조회하는 **방법만 약속한 추상 Repository 인터페이스**입니다. Use Case는 이 인터페이스에만 의존할 뿐, 실제 저장 방식이 SQLite인지 인메모리인지는 알지 못합니다(DIP). 지금은 "저장소를 다루는 창구" 정도로만 이해하고 넘어가면 되고, 세 인터페이스의 실제 정의는 바로 다음 **STEP 3(Interface)**에서 자세히 다룹니다.

### 1) 대출 — `BorrowBookUseCase`

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
```

네 개 중 가장 복잡한 Use Case입니다. **조회 → 검증 → 상태 변경 → 저장**의 네 단계로 흐릅니다.

- **조회**: 책과 회원을 저장소에서 꺼냅니다. 회원은 `today`를 넘겨 그 날짜 기준으로 연체 여부(`has_overdue_loan`)를 계산합니다.
- **검증**: `member.assert_can_borrow()`가 연체·대출 한도를 검사하고, `book.borrow_copy()`가 재고가 남았는지 확인하며 한 권 차감합니다. 규칙 위반 시 여기서 예외가 발생하고, 그 아래 코드는 **실행되지 않습니다**(뒤 단계로 넘어가기 전에 막힘).
- **상태 변경**: `Loan.new(...)`로 새 대출 기록을 만들어 저장하고(발급된 `id`를 돌려받음), `member.register_borrow()`로 회원의 대출 권수를 +1 합니다.
- **저장**: 이 시나리오에서 값이 바뀐 것은 `member`(권수)와 `book`(재고) 두 객체이므로 **각각 `save()`** 합니다. 하나라도 빠뜨리면 재고나 권수 변경이 반영되지 않습니다.

### 2) 반납 — `ReturnBookUseCase`

```python
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
```

대출의 **반대 과정**입니다. 대출 기록(`loan`)을 기준으로 관련된 책과 회원을 되짚어 원상 복구합니다.

- `loan.mark_returned(today)` — 대출을 반납 상태로 표시합니다(반납일 기록).
- `book.return_copy()` — 차감했던 재고를 한 권 되돌립니다.
- `member.register_return()` — 회원의 대출 권수를 −1 합니다.
- 대출에서는 새로 만든 `loan`을 저장했다면, 반납에서는 **`loan`·`book`·`member` 세 객체가 모두 바뀌므로 셋 다 저장**합니다.

여기서 `book_id`, `member_id`를 직접 받지 않고 **`loan`에서 꺼내 쓴다**는 점에 주목하세요. 반납은 "어떤 대출을 되돌리는가"만 알면 나머지 대상은 대출 기록이 알려줍니다.

### 3) 연장 — `ExtendLoanUseCase`

```python
@dataclass
class ExtendLoanUseCase:
    loan_repo: LoanRepository

    def execute(self, loan_id: int, today: Optional[date] = None) -> Loan:
        today = today or date.today()
        loan = self.loan_repo.get(loan_id)
        loan.extend(today)                # 이미 연장했거나 연체 중이면 예외
        return self.loan_repo.save(loan)
```

바뀌는 대상이 **대출 하나뿐**이라 가장 단순합니다. 그래서 `loan_repo` **하나에만** 의존합니다(책·회원 저장소가 필요 없음).

- `loan.extend(today)` — 연장 가능 여부(이미 연장했는지, 연체 중인지)를 **Entity가 스스로 판단**하고, 가능하면 반납 기한을 미룹니다. 규칙 위반 시 예외가 발생합니다.
- 규칙 판단이 전부 `Loan` 안에 있으므로, Use Case는 "연장해라"라고 **호출만** 하고 바뀐 대출을 저장할 뿐입니다.

### 4) 대출 목록 조회 — `ListMemberLoansUseCase`

```python
@dataclass
class ListMemberLoansUseCase:
    loan_repo: LoanRepository

    def execute(self, member_id: int) -> list[Loan]:
        # 조회만 하는 Use Case — 검증·상태 변경 없이 Repository에 위임
        return self.loan_repo.list_by_member(member_id)
```

유일한 **조회 전용(read-only)** Use Case입니다. 상태를 바꾸지 않으므로 검증도, 저장도 없습니다.

- 하는 일은 저장소의 `list_by_member(member_id)` 호출을 **그대로 위임**하는 것뿐입니다.
- 이렇게 단순해도 별도의 Use Case로 두는 이유는, 화면·API가 저장소를 직접 만지지 않고 **항상 Use Case를 통해** 도메인에 접근하도록 창구를 일관되게 유지하기 위해서입니다.

## 핵심 설계 원칙

- **Use Case는 조율자**: 직접 규칙을 구현하지 않고, Entity 메서드를 올바른 순서로 호출합니다. 규칙(연체·재고·연장 한도)은 모두 Entity 안에 있고, Use Case는 "언제 무엇을" 호출할지 순서만 정합니다.
- **바꿨으면 저장한다**: 대출은 `loan`·`member`·`book` 세 객체가 모두 바뀌므로 각각 `repo.save()`로 저장합니다. 저장을 빠뜨리면 재고·대출 권수 변경이 사라집니다.
- **조회형 Use Case는 단순**: `ListMemberLoansUseCase`처럼 상태를 바꾸지 않는 경우 검증·저장 없이 Repository 호출만 위임합니다.
- **Repository 인터페이스에만 의존**: `book_repo`가 실제로 SQLite인지, 인메모리인지 알지 못합니다. (DIP 원칙)
- **`today` 주입**: 시간 의존적 로직을 테스트하기 위해 날짜를 외부에서 주입받습니다. `date.today()`를 내부에서 호출하면 테스트가 불가능해집니다.
