# 11. 완성 — 최종 프로젝트 구조

## 파일 구조

```
library_app/
├── domain/
│   ├── entities.py       # Book, Member, Loan
│   └── exceptions.py     # DomainError 계층
├── application/
│   ├── repositories.py   # 추상 인터페이스 (ABC)
│   └── use_cases.py      # BorrowBook / ReturnBook / ExtendLoan
├── adapters/
│   ├── orm_models.py     # SQLAlchemy 모델
│   ├── repositories.py   # SQLAlchemy 구현체
│   └── schemas.py        # Pydantic 요청/응답 스키마
├── infrastructure/
│   └── db.py             # SQLite 엔진, get_db 팩토리
├── main.py               # 조립(wiring) — 유일한 연결 지점
├── seed_data.py          # 샘플 데이터 삽입
└── tests/
    ├── fakes.py           # 인메모리 Repository
    ├── test_entities.py   # Entity 단위 테스트
    └── test_use_cases.py  # Use Case 통합 테스트
```

## 핵심 지표

| 지표 | 값 |
|---|---|
| 단위 테스트 통과 | **20개** |
| Entity (Book · Member · Loan) | **3개** |
| 레이어 완전 분리 | **4개** |
| 테스트에 필요한 DB | **0개** |

## 레이어별 의존성 요약

| 레이어 | import 가능한 대상 |
|---|---|
| `domain/` | 표준 라이브러리만 |
| `application/` | `domain/`만 |
| `adapters/` | `domain/`, `application/`, 외부 라이브러리(SQLAlchemy, Pydantic) |
| `infrastructure/` | 외부 라이브러리만 |
| `main.py` | 전 레이어 |

이 규칙을 지키면, 어떤 레이어를 교체해도 안쪽 레이어는 영향받지 않습니다.
