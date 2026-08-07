# 1-9. 예외 처리 — 규칙 위반을 한 곳에서 HTTP로 변환하기

지금까지 코드 곳곳에서 `raise BookNotAvailableError(...)` 같은 예외가 나왔습니다. 이 예외들이 어떻게 사용자에게 전달되는지, 그 흐름을 정리합니다.

핵심은 세 단계입니다.

1. **발생** — 규칙을 아는 곳(Entity·Repository)이 위반을 감지하면 예외를 던진다.
2. **통과** — Use Case는 예외를 잡지 않고 그대로 흘려보낸다.
3. **변환** — 가장 바깥쪽 `main.py`의 핸들러 하나가 HTTP 400으로 바꾼다.

## 모든 도메인 예외는 `DomainError`를 상속한다

```python
# domain/exceptions.py
class DomainError(Exception):
    """모든 도메인 예외의 기반 클래스"""

class BookNotFoundError(DomainError): ...
class BookNotAvailableError(DomainError): ...      # 재고 없음
class MemberCannotBorrowError(DomainError): ...    # 연체·한도 초과
class LoanExtensionError(DomainError): ...         # 연장 불가
```

예외 종류는 여럿이지만 **부모가 하나**입니다. 그리고 이 파일은 상태 코드 `400` 같은 HTTP 개념을 전혀 모릅니다 — "무엇이 규칙 위반인가"만 표현합니다.

## 던지는 곳: 규칙을 아는 Entity·Repository

```python
# domain/entities.py
def borrow_copy(self) -> None:
    if not self.has_available_copy():
        raise BookNotAvailableError(f"'{self.title}'의 대출 가능한 사본이 없습니다")
    self.available_copies -= 1
```

규칙과 예외가 같은 자리에 있습니다. Use Case에는 `try/except`가 없어서, 예외가 나면 그 줄에서 멈추고 호출한 쪽으로 그대로 전달됩니다.

## 변환하는 곳: `main.py` 핸들러 하나

```python
# main.py
@app.exception_handler(DomainError)
def handle_domain_error(request, exc: DomainError):
    from fastapi.responses import JSONResponse
    return JSONResponse(status_code=400, content={"detail": str(exc)})
```

FastAPI는 예외의 **부모 타입**으로도 핸들러를 매칭합니다. 그래서 `DomainError` 하나만 등록하면 `BookNotAvailableError` 등 **모든 자식이 자동으로 여기에 걸려** 400으로 바뀝니다.

## 이렇게 얻는 것

- **도메인은 HTTP를 모른다** — 예외는 규칙 위반만 표현하므로, 같은 도메인을 CLI·배치에서도 그대로 재사용할 수 있습니다.
- **변환은 한 곳에서만** — 상태 코드·응답 형식을 바꾸려면 `main.py` 한 곳만 고치면 됩니다.
- **새 예외도 공짜로 처리** — `DomainError`만 상속하면 핸들러를 손대지 않아도 자동으로 400이 됩니다.
- **버그와 구분** — `DomainError` 계열만 400이고, 예기치 못한 다른 예외는 500으로 남아 "규칙 위반"과 "고쳐야 할 버그"가 나뉩니다.
