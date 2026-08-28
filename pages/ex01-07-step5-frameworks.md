# 1-08. STEP 5 · Frameworks & Drivers — FastAPI 엔드포인트로 조립하기

`main.py`는 모든 레이어를 조립(wiring)하는 유일한 곳입니다. 도메인 예외는 여기서 HTTP 응답으로 변환됩니다.

## 개요

코드를 보기 전에, `main.py`가 하는 일과 엔드포인트 구성을 살펴봅니다.

- **엔드포인트 3개** — `POST /loans`(대출)·`POST /loans/{id}/return`(반납)·`POST /loans/{id}/extend`(연장)이 각각 Use Case 하나에 연결됩니다.
- **조립(wiring)** — 유스케이스마다 의존성 프로바이더(`get_borrow_uc` 등)를 두어, 각 라우트가 **자기가 쓸 유스케이스 하나만** 주입받습니다.
- **예외 변환** — `DomainError`를 단일 핸들러로 잡아 HTTP 400으로 바꿉니다.
- **DB 세션 & 트랜잭션** — `infrastructure/db.py`의 `get_db()`가 요청별 세션을 열고, 성공 시 커밋·실패 시 롤백한 뒤 닫습니다 (요청 하나 = 트랜잭션 하나).

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


# 유스케이스별 의존성 프로바이더 — 라우트는 자기가 쓸 유스케이스 하나만 주입받는다.
def get_borrow_uc(session: Session = Depends(get_db)) -> BorrowBookUseCase:
    return BorrowBookUseCase(
        SqlBookRepository(session), SqlMemberRepository(session), SqlLoanRepository(session)
    )


def get_return_uc(session: Session = Depends(get_db)) -> ReturnBookUseCase:
    return ReturnBookUseCase(
        SqlBookRepository(session), SqlMemberRepository(session), SqlLoanRepository(session)
    )


def get_extend_uc(session: Session = Depends(get_db)) -> ExtendLoanUseCase:
    return ExtendLoanUseCase(SqlLoanRepository(session))


@app.post("/loans", response_model=LoanResponse)
def borrow_book(req: BorrowRequest, uc: BorrowBookUseCase = Depends(get_borrow_uc)):
    loan = uc.execute(book_id=req.book_id, member_id=req.member_id)
    return LoanResponse.from_entity(loan)


@app.post("/loans/{loan_id}/return", response_model=LoanResponse)
def return_book(loan_id: int, uc: ReturnBookUseCase = Depends(get_return_uc)):
    loan = uc.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)


@app.post("/loans/{loan_id}/extend", response_model=LoanResponse)
def extend_loan(loan_id: int, uc: ExtendLoanUseCase = Depends(get_extend_uc)):
    loan = uc.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)
```

> **유스케이스별 프로바이더를 쓰는 이유.** `get_borrow_uc` 같은 함수를 `Depends(...)`로 주입하면,
> 라우트는 **필요한 유스케이스 하나만** 받습니다. 안 쓰는 유스케이스를 만들지 않고, 조립 로직이
> FastAPI 의존성 시스템에 자연스럽게 얹힙니다. 저장소(Repository) 생성은 이 프로바이더 안에서만
> 일어나므로, 조립 지점은 여전히 `main.py` 한 곳입니다. SQLite → PostgreSQL 교체 시에도
> 프로바이더의 저장소 생성 부분만 바꾸면 됩니다.

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
        db.commit()          # 라우트가 정상 종료되면 여기서 한 번만 커밋
    except Exception:
        db.rollback()        # 도중에 예외가 나면 전체 롤백 (부분 저장 방지)
        raise
    finally:
        db.close()
```

> **여기가 트랜잭션 경계입니다.** 저장소의 `save()`는 `commit`이 아니라 `flush`만 하므로(STEP 4),
> 실제 확정은 이 `get_db()`가 담당합니다. 라우트가 정상적으로 끝나면 `yield db` 다음 줄에서
> **한 번만 커밋**하고, 도중에 `DomainError` 같은 예외가 나면 **롤백**한 뒤 다시 던집니다.
> 그 예외는 이어서 위의 `handle_domain_error` 핸들러가 HTTP 400으로 변환합니다.
> 덕분에 "대출 기록은 저장됐는데 책 재고는 안 줄어든" 부분 저장 상태가 생기지 않습니다.
> (요청 하나의 모든 DB 작업이 all-or-nothing으로 묶임)

## `main.py`가 유일한 연결 지점인 이유

- 다른 어떤 파일도 전 레이어를 import하지 않습니다
- 레이어 교체(예: SQLite → PostgreSQL)는 `main.py`의 Repository 생성 부분만 바꾸면 됩니다
- `DomainError` 단일 핸들러로 모든 비즈니스 예외를 HTTP 400으로 일괄 변환합니다
