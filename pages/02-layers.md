# 02. 4개 레이어, 우리 프로젝트에 대입하면?

이번 프로젝트에서 각 레이어가 실제로 어떤 클래스·파일에 해당하는지 미리 보여드립니다.

| 레이어 | 파일 |
|---|---|
| **Entities** | `domain/entities.py` — Book, Member, Loan |
| **Use Cases** | `application/use_cases.py` — BorrowBook, ReturnBook, ExtendLoan |
| **Interface Adapters** | `adapters/` — SqlBookRepository, LoanResponse 스키마 |
| **Frameworks & Drivers** | `main.py`, `infrastructure/db.py` — FastAPI, SQLAlchemy |

## 의존성 흐름

안쪽 레이어는 바깥쪽 레이어를 절대 import하지 않습니다. import 방향이 곧 의존성 규칙입니다.

```
domain          →  (없음)
application     →  domain
adapters        →  domain, application
infrastructure  →  (외부 라이브러리만)
main.py         →  전 레이어  ← 유일한 연결 지점
```

이 규칙 덕분에 `domain/`과 `application/`은 FastAPI, SQLAlchemy를 전혀 모르는 상태로 유지됩니다. 프레임워크를 교체해도 안쪽 두 레이어는 수정할 필요가 없습니다.
