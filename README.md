# 클린 아키텍처 with 파이썬

> FastAPI · SQLAlchemy · pytest로 직접 구현하는 도서 대출 관리 시스템  
> WikiDocs 전자책: [wikidocs.net](https://wikidocs.net)

## 이 책에서 다루는 것

요구사항 정의부터 FastAPI 배포까지, 클린 아키텍처의 4개 레이어를 실제 프로젝트에 직접 적용합니다.

- 비즈니스 규칙을 순수 Python 코드(Entities)로 표현하는 방법
- Use Case가 Entity를 조율하는 방식
- Repository 인터페이스로 DB 의존성을 역전시키는 DIP 원칙
- SQLAlchemy ORM 모델과 도메인 Entity를 분리하는 이유
- DB 없이 0.05초 만에 20개 테스트를 통과하는 테스트 전략

## 목차

| 챕터 | 제목 |
|---|---|
| 01 | 클린 아키텍처란 무엇인가? |
| 02 | 4개 레이어, 우리 프로젝트에 대입하면? |
| 03 | 비즈니스 규칙 정의 |
| 04 | 도메인 모델 설계 |
| 05 | STEP 1 · Entities |
| 06 | STEP 2 · Use Cases |
| 07 | STEP 3 · Interface (Repository 추상화) |
| 08 | STEP 4 · Adapters (SQLAlchemy 구현) |
| 09 | STEP 5 · Frameworks & Drivers (FastAPI) |
| 10 | STEP 6 · Testing (DB 없는 테스트) |
| 11 | 완성 — 최종 프로젝트 구조 |
| 12 | 직접 실행하고, 확장해보세요 |

## 예제 프로젝트

책에서 다루는 `library_app` 전체 코드:

```bash
git clone https://github.com/hyejin/library_app
cd library_app
pip install -r requirements.txt
python seed_data.py
uvicorn main:app --reload   # http://127.0.0.1:8000/docs
pytest -v                   # 20개 테스트 즉시 통과
```

## 이 리포지토리 구조

```
clean-architecture/
├── TOC.md          # WikiDocs 목차 정의
├── pages/          # 챕터별 마크다운 파일
└── README.md
```

`TOC.md`를 수정하고 `pages/`에 파일을 추가한 뒤 push하면 WikiDocs에 자동 반영됩니다.
