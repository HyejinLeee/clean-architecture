# 1-11. 완성 — 최종 프로젝트 구조 & 확장 아이디어

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

## 직접 실행하기

```bash
pip install -r requirements.txt
python seed_data.py
uvicorn main:app --reload   # http://127.0.0.1:8000/docs

pytest -v                   # 20개 테스트, ~0.05초
```

## 이 예제의 범위 밖 (학습용 한계)

이 프로젝트는 **클린 아키텍처의 구조를 익히는 데 집중한 학습용 예제**입니다. 실제 서비스에 필요한 몇 가지는 의도적으로 다루지 않았습니다.

- **동시성은 고려하지 않습니다.** 예를 들어 마지막 1권을 두 사람이 **동시에** 대출 요청하면, 둘 다 "재고 있음"을 확인하고 둘 다 빌리는 상황이 생길 수 있습니다. 이를 막으려면 잠금(lock)이나 트랜잭션 격리, DB 제약 같은 장치가 필요하지만, 구조 학습에 집중하기 위해 여기서는 다루지 않습니다.
- 인증·권한, 페이지네이션, 로깅·모니터링 등 운영에 필요한 요소도 범위 밖입니다.

이런 항목들은 클린 아키텍처의 장점을 활용해 **바깥 레이어에 자연스럽게 덧붙일 수 있는 확장 지점**입니다.

