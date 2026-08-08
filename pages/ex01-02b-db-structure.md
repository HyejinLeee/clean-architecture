# 1-03. DB 구조와 샘플 데이터 준비

구현(STEP 1~)에 들어가기 전에, 데이터가 **어디에 어떤 모양으로** 저장되고, **어떻게 넣는지**를 먼저 봅니다. 이걸 알아두면 이후 코드가 "어떤 테이블을 건드리는 건지" 감이 잡힙니다.

## 한눈에 보기

이 프로젝트는 **SQLite 파일 하나(`library.db`)** 에 **테이블 3개**를 둡니다.

```
  books                 loans                    members
──────────────    ─────────────────────    ──────────────
id (PK)       ◀── book_id   (FK)            id (PK)
title             member_id (FK) ──▶
author            borrowed_at              name
total_copies      due_at
                  returned_at
                  extension_count
```

`loans`가 **중심 테이블**입니다. 누가(`member_id`) 어떤 책을(`book_id`) 언제 빌리고 반납했는지를 한 행에 기록하며, `books`·`members`의 `id`만 참조합니다.

## 테이블별 컬럼

**`books` (도서)**

| 컬럼 | 설명 |
|---|---|
| `id` | PK, 자동 증가 |
| `title` / `author` | 제목 / 저자 |
| `total_copies` | 보유 총 권수 |

**`members` (회원)**

| 컬럼 | 설명 |
|---|---|
| `id` | PK, 자동 증가 |
| `name` | 회원 이름 |

**`loans` (대출 기록)**

| 컬럼 | 설명 |
|---|---|
| `id` | PK, 자동 증가 |
| `book_id` | `books.id` 참조 (FK) |
| `member_id` | `members.id` 참조 (FK) |
| `borrowed_at` | 대출일 |
| `due_at` | 반납 기한 |
| `returned_at` | 반납일 (`NULL`이면 **아직 대출 중**) |
| `extension_count` | 연장 횟수 |

## ⚠️ 저장하지 않는 값 — 파생값

도메인 모델(1-02)에서 본 `available_copies`(재고), `active_loans_count`(대출 권수), `has_overdue_loan`(연체 여부)은 **테이블에 컬럼으로 없습니다.** 이 값들은 저장하지 않고, 조회할 때마다 `loans`에서 계산합니다.

- `available_copies` = `total_copies − (그 책의 반납 안 된 loans 개수)`
- `active_loans_count` = `그 회원의 반납 안 된 loans 개수`
- `has_overdue_loan` = `그 회원의 반납 안 됐고 기한 지난 loans가 하나라도 있는가`

> 이렇게 하면 "재고 컬럼과 실제 대출 기록이 어긋나는" 문제가 구조적으로 사라집니다. 자세한 계산 방식과 이유는 **STEP 4(Adapters)** 에서 다룹니다.

## DB 파일과 테이블은 언제 생기나

DB 설정은 `infrastructure/db.py` 한 곳에 있습니다.

```python
# infrastructure/db.py
DATABASE_URL = "sqlite:///./library.db"          # 프로젝트 폴더에 library.db 파일 생성
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

테이블 자체는 아래 한 줄이 실행될 때 (없으면) 만들어집니다. `seed_data.py`와 `main.py`가 시작할 때 호출합니다.

```python
Base.metadata.create_all(bind=engine)            # 정의된 모델대로 테이블 생성
```

## 샘플 데이터 넣기

테이블만 있으면 비어 있으니, 데모용 데이터를 넣는 스크립트를 실행합니다.

```bash
python seed_data.py
```

이 스크립트는 **테이블을 만들고 → 책 3권·회원 3명·대출 3건**을 넣습니다. 이미 데이터가 있으면 건너뜁니다(여러 번 실행해도 안전).

넣는 회원 3명은 **서로 다른 상태**라서, 이후 대출·연체 규칙을 바로 실습해볼 수 있습니다.

| 회원 | 상태 | 이유 |
|---|---|---|
| 김철수 | 정상 (대출 1건 진행 중) | 반납 기한이 아직 남음 |
| 이영희 | **대출 불가** | 연체 대출(기한 16일 초과)이 있어 `has_overdue_loan = True` |
| 박지민 | 정상 (대출 없음) | 자유롭게 대출 가능 |

```python
# seed_data.py (발췌)
books = [
    BookModel(title="클린 아키텍처", author="로버트 마틴", total_copies=2),
    BookModel(title="클린 코드", author="로버트 마틴", total_copies=1),
    BookModel(title="객체지향의 사실과 오해", author="조영호", total_copies=3),
]
members = [
    MemberModel(name="김철수"),  # 대출 1건 진행 중
    MemberModel(name="이영희"),  # 연체 대출 → 대출 불가
    MemberModel(name="박지민"),  # 대출 없음 → 정상
]
session.add_all(books)
session.add_all(members)
session.flush()                  # PK(id) 자동 할당 → 아래 loans에서 참조
# ... 각 회원 상태에 맞는 loans 3건을 add_all 후 commit
```

준비가 끝났습니다. 이제 **STEP 1**부터 이 구조를 코드로 하나씩 만들어 갑니다. (여기서 본 ORM 모델과 저장 코드의 실제 구현은 STEP 4에서 자세히 다룹니다.)
