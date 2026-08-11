# 1-04. STEP 1 · Entities — 핵심 규칙을 메서드로 표현하기

각 규칙을 해당 Entity의 메서드로 캡슐화합니다.

> 사실 이 **엔티티 설계**가 클린 아키텍쳐에서 가장 고민해야할 부분이고, 또 중요한 부분이라고 생각합니다. "규칙을 어느 객체에 담고 무엇을 필드로 둘지"를 정하는 감각은 단번에 얻어지지 않는다고 생각하고 있습니다.. 저 역시 [객체지향의 사실과 오해](https://product.kyobobook.co.kr/detail/S000001628109)(조영호, 위키북스) 같은 책을 참고하며 조금씩 이해를 높여가려고 노력 중입니다.

## 개요

코드를 보기 전에, 우리가 만들 Entity 세 개가 각각 **어떤 데이터를 가지고(속성)**, **무엇을 할 수 있는지(행동)** 살펴봅니다.

- **Book(도서)** 은 `id·제목·저자·총 권수·대출 가능 권수`를 가지고, 빌려주고(borrow_copy) 반납받는(return_copy) 일을 합니다. 이를 통해 해당 책의 대출 가능 권수가 갱신됩니다.
- **Member(회원)** 은 `id·이름·현재 대출 권수·연체 여부`를 가지고, 해당 회원에 대해 대출이 가능한지 스스로 검사(assert_can_borrow)하고 대출/반납 시 해당 회원의 대출권수를 갱신합니다.
- **Loan(대출 기록)** 은 `누가·어떤 책을·언제 빌려서·언제까지인지`를 가지고, 연장(extend)·연체 판정(is_overdue)·반납 처리(mark_returned)를 합니다.

> ### 🧭 설계 사고법 — "이 필드가 왜 필요한가"를 판단하는 법
>
> `active_loans_count`·`available_copies` 같은 필드는 "저장할 데이터"에서 나오지 않습니다. **규칙(행동)에서 거꾸로 도출**해야 합니다.
>
> #### 4단계로 도출하기
>
> 1. **규칙을 먼저 적는다** (데이터 말고)
>    회원은 최대 5권 · 연체 중이면 불가 · 재고 없으면 불가
> 2. **규칙이 내리는 판단을 문장으로**
>    `대출 권수 < 5?` · `연체 없나?` · `남은 재고 > 0?`
> 3. **판단에 필요한 값을 찾는다** — 이게 곧 필드다
>    `active_loans_count` · `has_overdue_loan` · `available_copies`
> 4. **그 값을 규칙의 주인 엔티티에 붙인다**
>    회원 규칙 → `Member` · 재고 규칙 → `Book`
>
> > **원칙** — 판단(규칙)과 그에 필요한 데이터는 같은 엔티티에 둡니다. 규칙 검사가 엔티티 바깥(Use Case)에서 일어난다면, 엔티티에 필드/메서드가 빠졌다는 신호입니다.
>
> #### 헷갈리면 두 질문을 분리하세요
>
> - **엔티티가 알아야 하나?** — 규칙을 지키려면 필요한가? → `active_loans_count`는 **예**
> - **DB에 저장해야 하나?** — 저장 vs 계산(성능·일관성)은 **별개 문제** → 여기선 **아니오** (loans에서 계산)
>
> 그래서 `Member`엔 필드가 **있어도** `members` 테이블엔 컬럼이 **없습니다**. (엔티티 필드 ≠ DB 컬럼)
>
> #### 필드로 둘까, 메서드로 둘까?
>
> - 원본을 엔티티가 **자기 안에 가짐** → 메서드로 계산 (`Loan.is_overdue()`는 자기 `due_at`으로 판단)
> - 원본이 **다른 곳(loans)에 있음** → 필드로 받아 채움 (`active_loans_count`는 Repository가 조회 시 세서 넣음)
>
> | 규칙 | 필요한 값 | 엔티티 | 원본/파생 |
> |---|---|---|---|
> | 최대 5권 | `active_loans_count` | Member | 파생(채워넣음) |
> | 연체 시 불가 | `has_overdue_loan` | Member | 파생(채워넣음) |
> | 재고 없으면 불가 | `available_copies` | Book | 파생(채워넣음) |
> | 연장 2회까지 | `extension_count` | Loan | 원본(저장·자체계산) |

각 규칙이 왜 그 Entity의 메서드로 들어가는지 염두에 두고, 이제 실제 코드를 봅니다.

### Book — 재고 규칙을 지키는 도서

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
```

"재고가 없으면 빌려줄 수 없다"는 규칙을 Book이 직접 지킵니다.

- `borrow_copy()` — 재고 확인과 차감을 **한 메서드 안에서** 함께 합니다. 재고가 없으면 예외를 던지고, 있으면 `available_copies`를 −1 합니다.
- `return_copy()` — 차감했던 재고를 되돌립니다. 총 권수를 넘겨 늘리려 하면 막습니다.
- 외부에서 `available_copies -= 1`을 직접 만지지 못하게 **메서드로 감싼** 것이 핵심입니다. 재고를 바꾸는 유일한 통로가 이 두 메서드입니다.

### Member — 대출 자격을 스스로 판단하는 회원

```python
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
```

"연체 중이거나 5권을 채웠으면 못 빌린다"는 자격 규칙을 Member가 판단합니다.

- `assert_can_borrow()` — 연체 여부와 대출 한도 **두 규칙을 검사**하고, 위반 시 예외를 던집니다. 판단만 하고 상태는 바꾸지 않습니다.
- `register_borrow()` / `register_return()` — 대출·반납 시 `active_loans_count`를 +1 / −1 합니다. 대출 쪽은 권수를 올리기 전에 `assert_can_borrow()`로 한 번 더 확인합니다.
- 규칙의 기준값(5권)을 `MAX_ACTIVE_LOANS` **상수**로 빼두어, 규칙이 코드에 숫자로 흩어지지 않게 했습니다.

### Loan — 대출 한 건의 생애를 표현

```python
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

대출 한 건이 생겨서(발급) 연장되고 반납되기까지를 Loan이 담습니다.

- `new()` — 대출 기간(14일)을 반영해 `due_at`을 계산하며 새 대출을 만듭니다. 기간 규칙이 생성 시점에 들어갑니다.
- `is_overdue()` — 자기 `due_at`·`returned_at`만 보고 **스스로 연체를 판정**합니다. 원본을 자기 안에 가지고 있어 메서드로 계산하는 예입니다.
- `mark_returned()` — 이미 반납된 건을 또 반납하려 하면 막습니다.
- `extend()` — "2회까지" "연체 중엔 불가" 두 조건을 확인한 뒤 기한을 7일 미룹니다. 연장 규칙이 전부 이 안에 있습니다.

## 핵심 설계 원칙

- **순수 Python만 사용**: `from dataclasses import dataclass`만 import합니다. FastAPI, SQLAlchemy가 전혀 없습니다.
- **규칙은 메서드 안에**: `borrow_copy()`가 재고 검사와 차감을 함께 수행합니다. 외부에서 직접 `available_copies -= 1`을 쓰지 않습니다.
- **예외로 실패를 표현**: `if` 분기가 아닌 `raise`로 실패를 명확히 전달합니다.
