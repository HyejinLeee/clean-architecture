# 12. 직접 실행하고, 확장해보세요

## 실행 방법

```bash
# 의존성 설치
pip install -r requirements.txt

# 샘플 데이터 삽입 (books.db 생성)
python seed_data.py

# API 서버 실행
uvicorn main:app --reload
```

서버가 실행되면 브라우저에서 `http://127.0.0.1:8000/docs`를 열면 Swagger UI로 API를 바로 테스트할 수 있습니다.

```bash
# 전체 테스트 실행 (DB, 서버 없이 즉시)
pytest -v   # 20개 테스트, ~0.05초
```

## API 엔드포인트

| 메서드 | 경로 | 설명 |
|---|---|---|
| `POST` | `/loans` | 도서 대출 |
| `POST` | `/loans/{id}/return` | 도서 반납 |
| `POST` | `/loans/{id}/extend` | 대출 연장 (최대 2회) |
| `GET` | `/loans/{id}` | 대출 조회 |

## 다음 확장 아이디어

### 난이도 ★
- **연체료 계산 Use Case 추가** — 연체 일수에 비례한 요금 산정
- **관리자용 '연체 회원 목록' 조회 API 추가**

### 난이도 ★★
- **예약(Reservation) 기능** — 대출 중인 책 예약 대기열
- **회원 등록/조회 API 추가**

### 난이도 ★★★
- **PostgreSQL로 Repository만 교체해보기** — 도메인 코드 변경 없이 DB 교체 검증
- **이벤트 기반 연체 알림** — 반납 기한 D-1 이메일 발송 Use Case

각 확장 아이디어는 모두 동일한 패턴으로 구현됩니다:
1. `domain/entities.py`에 규칙을 메서드로 추가
2. `application/use_cases.py`에 Use Case 추가
3. 필요한 경우 `application/repositories.py`에 추상 메서드 추가
4. `adapters/`에 구현체 추가
5. `main.py`에 엔드포인트 연결

---

*Clean code, clean rules, clean tests.*
