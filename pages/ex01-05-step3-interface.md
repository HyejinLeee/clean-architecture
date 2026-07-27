# 07. STEP 3 · Interface — Repository 인터페이스로 DB를 추상화하기

Use Case는 이 추상 클래스만 알고, 실제 구현(SQLAlchemy/인메모리)은 전혀 알지 못합니다.

```python
# application/repositories.py
from abc import ABC, abstractmethod
from typing import Optional
from domain.entities import Book, Member, Loan


class BookRepository(ABC):
    @abstractmethod
    def get(self, book_id: int) -> Book: ...

    @abstractmethod
    def save(self, book: Book) -> Book: ...


class MemberRepository(ABC):
    @abstractmethod
    def get(self, member_id: int, as_of: date = None) -> Member: ...

    @abstractmethod
    def save(self, member: Member) -> Member: ...


class LoanRepository(ABC):
    @abstractmethod
    def get(self, loan_id: int) -> Loan: ...

    @abstractmethod
    def save(self, loan: Loan) -> Loan: ...

    @abstractmethod
    def find_active_loan_by_book(self, book_id: int) -> Optional[Loan]: ...

    @abstractmethod
    def list_by_member(self, member_id: int) -> list[Loan]: ...
```

## DIP — 의존성 역전 원칙

> "고수준 모듈(Use Case)은 저수준 모듈(SQLAlchemy)에 의존해서는 안 된다. 둘 다 추상화에 의존해야 한다."

```
Use Case  →  Repository (추상)  ←  SqlRepository (구현체)
Use Case  →  Repository (추상)  ←  InMemoryRepository (테스트용)
```

추상 인터페이스 덕분에 Use Case 코드를 단 한 줄도 수정하지 않고 SQLite를 PostgreSQL로, 또는 테스트용 인메모리 저장소로 교체할 수 있습니다.
