# 09. STEP 5 · Frameworks & Drivers — FastAPI 엔드포인트로 조립하기

`main.py`는 모든 레이어를 조립(wiring)하는 유일한 곳입니다. 도메인 예외는 여기서 HTTP 응답으로 변환됩니다.

## 조립(Wiring)

```python
# main.py
from fastapi import FastAPI, Depends
from fastapi.responses import JSONResponse
from sqlalchemy.orm import Session

from domain.exceptions import DomainError
from application.use_cases import BorrowBookUseCase, ReturnBookUseCase, ExtendLoanUseCase
from adapters.repositories import SqlBookRepository, SqlMemberRepository, SqlLoanRepository
from adapters.schemas import BorrowRequest, LoanResponse
from infrastructure.db import get_db

app = FastAPI()


@app.exception_handler(DomainError)
def handle_domain_error(request, exc: DomainError):
    return JSONResponse(status_code=400, content={'detail': str(exc)})


@app.post('/loans', response_model=LoanResponse)
def borrow_book(req: BorrowRequest, session: Session = Depends(get_db)):
    use_case = BorrowBookUseCase(
        book_repo=SqlBookRepository(session),
        member_repo=SqlMemberRepository(session),
        loan_repo=SqlLoanRepository(session),
    )
    loan = use_case.execute(book_id=req.book_id, member_id=req.member_id)
    return LoanResponse.from_entity(loan)


@app.post('/loans/{loan_id}/return', response_model=LoanResponse)
def return_book(loan_id: int, session: Session = Depends(get_db)):
    use_case = ReturnBookUseCase(
        book_repo=SqlBookRepository(session),
        member_repo=SqlMemberRepository(session),
        loan_repo=SqlLoanRepository(session),
    )
    loan = use_case.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)


@app.post('/loans/{loan_id}/extend', response_model=LoanResponse)
def extend_loan(loan_id: int, session: Session = Depends(get_db)):
    use_case = ExtendLoanUseCase(loan_repo=SqlLoanRepository(session))
    loan = use_case.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)
```

## DB 세션 팩토리

```python
# infrastructure/db.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("sqlite:///./books.db", connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine)

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
