# 1-08. STEP 5 · Frameworks & Drivers — FastAPI 엔드포인트로 조립하기

`main.py`는 모든 레이어를 조립(wiring)하는 유일한 곳입니다. 도메인 예외는 여기서 HTTP 응답으로 변환됩니다.

## 개요

코드를 보기 전에, `main.py`가 하는 일과 엔드포인트 구성을 살펴봅니다.

- **엔드포인트 3개** — `POST /loans`(대출)·`POST /loans/{id}/return`(반납)·`POST /loans/{id}/extend`(연장)이 각각 Use Case 하나에 연결됩니다.
- **조립(wiring)** — 요청마다 Repository 구현체를 만들어 Use Case에 주입하고 `execute()`를 호출합니다.
- **예외 변환** — `DomainError`를 단일 핸들러로 잡아 HTTP 400으로 바꿉니다.
- **DB 세션** — `infrastructure/db.py`의 `get_db()`가 요청별 세션을 열고 닫습니다.

`main.py`가 전 레이어를 아는 유일한 파일이라는 점을 염두에 두고, 이제 실제 코드를 봅니다.

## 조립(Wiring)

```python
# main.py
from fastapi import Depends, FastAPI
from fastapi.responses import JSONResponse
from sqlalchemy.orm import Session

from adapters.repositories import SqlBookRepository, SqlLoanRepository, SqlMemberRepository
from adapters.schemas import BorrowRequest, LoanResponse
from application.use_cases import BorrowBookUseCase, ExtendLoanUseCase, ReturnBookUseCase
from domain.exceptions import DomainError
from infrastructure.db import Base, engine, get_db

Base.metadata.create_all(bind=engine)       # 최초 실행 시 테이블 생성

app = FastAPI(title="도서 대출 관리 시스템")


@app.exception_handler(DomainError)
def handle_domain_error(request, exc: DomainError):
    return JSONResponse(status_code=400, content={"detail": str(exc)})


def _build_use_cases(session: Session):
    book_repo = SqlBookRepository(session)
    member_repo = SqlMemberRepository(session)
    loan_repo = SqlLoanRepository(session)
    return (
        BorrowBookUseCase(book_repo, member_repo, loan_repo),
        ReturnBookUseCase(book_repo, member_repo, loan_repo),
        ExtendLoanUseCase(loan_repo),
    )


@app.post("/loans", response_model=LoanResponse)
def borrow_book(req: BorrowRequest, session: Session = Depends(get_db)):
    borrow_uc, _, _ = _build_use_cases(session)
    loan = borrow_uc.execute(book_id=req.book_id, member_id=req.member_id)
    return LoanResponse.from_entity(loan)


@app.post("/loans/{loan_id}/return", response_model=LoanResponse)
def return_book(loan_id: int, session: Session = Depends(get_db)):
    _, return_uc, _ = _build_use_cases(session)
    loan = return_uc.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)


@app.post("/loans/{loan_id}/extend", response_model=LoanResponse)
def extend_loan(loan_id: int, session: Session = Depends(get_db)):
    _, _, extend_uc = _build_use_cases(session)
    loan = extend_uc.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)
```

## DB 세션 팩토리

```python
# infrastructure/db.py
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

DATABASE_URL = "sqlite:///./library.db"

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


class Base(DeclarativeBase):     # 모든 ORM 모델의 부모 (STEP 4에서 import)
    pass


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

## `main.py`가 유일한 연결 지점인 이유

- 다른 어떤 파일도 전 레이어를 import하지 않습니다
- 레이어 교체(예: SQLite → PostgreSQL)는 `main.py`의 Repository 생성 부분만 바꾸면 됩니다
- `DomainError` 단일 핸들러로 모든 비즈니스 예외를 HTTP 400으로 일괄 변환합니다
