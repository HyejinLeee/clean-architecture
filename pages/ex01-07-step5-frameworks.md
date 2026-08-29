# 1-08. STEP 5 · Frameworks & Drivers — FastAPI 엔드포인트로 조립하기

`main.py`는 모든 레이어를 조립(wiring)하는 유일한 곳입니다. 도메인 예외는 여기서 HTTP 응답으로 변환됩니다.

## 개요

코드를 보기 전에, `main.py`가 하는 일과 엔드포인트 구성을 살펴봅니다.

- **엔드포인트 3개** — `POST /loans`(대출)·`POST /loans/{id}/return`(반납)·`POST /loans/{id}/extend`(연장)이 각각 Use Case 하나에 연결됩니다.
- **조립(wiring)** — 각 라우트가 DB 세션을 주입받아, 그 자리에서 저장소를 Use Case에 연결(조립)해 실행합니다.
- **예외 변환** — `DomainError`를 단일 핸들러로 잡아 HTTP 400으로 바꿉니다.
- **DB 세션 & 트랜잭션** — `infrastructure/db.py`의 `get_db()`가 요청별 세션을 열고, 성공 시 커밋·실패 시 롤백한 뒤 닫습니다 (요청 하나 = 트랜잭션 하나).

`main.py`가 전 레이어를 아는 유일한 파일이라는 점을 염두에 두고, 이제 실제 코드를 봅니다.

## 조립(Wiring)

`main.py`는 길어 보이지만 **두 덩어리**로 나눠 읽으면 쉽습니다.

```
① 앱 초기화 — 임포트 · 테이블 생성 · 예외 핸들러
② 라우트    — 세션을 받아 저장소를 유스케이스에 조립하고 실행한다  ← 조립이 일어나는 곳
```

순서대로 봅니다.

### ① 앱 초기화 — 임포트 · 테이블 · 예외 핸들러

가장 먼저 앱을 세팅합니다. **이 파일만이 전 레이어를 import하는 유일한 곳**이라는 점에 주목하세요.

```python
# main.py (초기화)
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
```

- **임포트** — `adapters`·`application`·`domain`·`infrastructure` 네 레이어를 여기서 한꺼번에 불러옵니다. 다른 파일은 이렇게 전 레이어를 알지 못합니다.
- **`Base.metadata.create_all(bind=engine)`** — 앱이 처음 뜰 때 ORM 모델에 맞는 테이블(`books`·`members`·`loans`)을 없으면 만들어 줍니다.
- **`handle_domain_error`** — 모든 규칙 위반(`DomainError` 계열)을 HTTP 400으로 바꾸는 단일 핸들러입니다. (자세한 흐름은 "예외 처리" 장에서)

### ② 라우트 — 세션을 받아 조립하고 실행

엔드포인트는 각각 **DB 세션 하나를 주입받아**, 그 자리에서 저장소를 Use Case에 연결(조립)하고 실행합니다.

```python
# main.py (이어서)
@app.post("/loans", response_model=LoanResponse)
def borrow_book(req: BorrowRequest, session: Session = Depends(get_db)):
    uc = BorrowBookUseCase(
        SqlBookRepository(session), SqlMemberRepository(session), SqlLoanRepository(session)
    )
    loan = uc.execute(book_id=req.book_id, member_id=req.member_id)
    return LoanResponse.from_entity(loan)


@app.post("/loans/{loan_id}/return", response_model=LoanResponse)
def return_book(loan_id: int, session: Session = Depends(get_db)):
    uc = ReturnBookUseCase(
        SqlBookRepository(session), SqlMemberRepository(session), SqlLoanRepository(session)
    )
    loan = uc.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)


@app.post("/loans/{loan_id}/extend", response_model=LoanResponse)
def extend_loan(loan_id: int, session: Session = Depends(get_db)):
    uc = ExtendLoanUseCase(SqlLoanRepository(session))
    loan = uc.execute(loan_id=loan_id)
    return LoanResponse.from_entity(loan)
```

`borrow_book`을 한 줄씩 뜯어봅니다.

- **`session: Session = Depends(get_db)`** — FastAPI가 요청마다 `get_db()`(다음 절)를 호출해 만든 DB 세션을 이 인자에 넣어 줍니다.
- **`uc = BorrowBookUseCase(SqlBookRepository(session), ...)`** — 그 세션 위에 구체 저장소 3개를 만들고, Use Case 생성자에 주입합니다. **바로 이 줄이 "조립"입니다.** 저장소 셋이 **같은 세션**을 공유하므로 한 요청이 하나의 트랜잭션으로 묶입니다.
- **`uc.execute(...)`** — 조립된 Use Case를 실행합니다. 규칙 위반이 있으면 여기서 `DomainError`가 나고, 위의 핸들러가 400으로 바꿉니다.
- **`LoanResponse.from_entity(loan)`** — 순수 Entity를 API 응답(JSON)으로 변환해 돌려줍니다.
- **필요한 것만 조립** — 대출·반납은 책·회원·대출 저장소 셋 다, 연장은 대출 저장소 하나만 사용합니다.

### 요청 한 번의 전체 흐름

```
POST /loans 요청
   ↓
get_db()   → 요청용 DB 세션 생성 (Depends)
   ↓
borrow_book() 안에서:
   SqlBookRepository(session) … 저장소 3개 생성
   → BorrowBookUseCase(...)     Use Case 조립
   → uc.execute()              대출 실행
   → LoanResponse 반환
   ↓
get_db()가 커밋(성공) / 롤백(예외) 후 세션 종료
```

> **왜 라우트 안에서 조립할까?** 저장소를 Use Case에 연결하는 조립을 라우트에 그대로 두면, "이 엔드포인트가 어떤 저장소로 어떤 Use Case를 실행하는지"가 한눈에 읽힙니다. 별도의 프로바이더나 DI 프레임워크 없이도 조립 지점은 `main.py` 한 곳으로 유지됩니다. 저장소 생성이 세 라우트에 반복되는 게 부담이 될 만큼 커지면, 그때 의존성 프로바이더나 DI 컨테이너로 분리하면 됩니다.

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
