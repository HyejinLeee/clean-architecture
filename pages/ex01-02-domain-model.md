# 1-02. 도메인 모델 설계

세 Entity가 어떻게 관계를 맺는지 먼저 그려봅니다. Loan이 Book과 Member를 연결하는 중심 역할을 합니다.

```
  Book                  Loan                Member
──────────────    ──────────────────    ──────────────────
title, author     book_id, member_id    name
total_copies      borrowed_at, due_at   active_loans_count
available_copies  returned_at           has_overdue_loan
borrow_copy()     extend()              can_borrow()
return_copy()     is_overdue()          register_borrow()
```

> Loan은 Book과 Member의 `id`만 참조합니다 — 서로 다른 Entity를 직접 참조하지 않아 결합도를 낮춥니다.

## 설계 포인트

**Book**: 재고(available_copies)를 직접 관리합니다. 대출 시 `borrow_copy()`가 재고를 줄이고, 반납 시 `return_copy()`가 늘립니다.

**Member**: 현재 대출 권수(`active_loans_count`)와 연체 여부(`has_overdue_loan`)를 보관합니다. 이 두 값은 비정규화 필드로, 매번 DB를 집계하지 않고 빠르게 대출 가능 여부를 판단할 수 있게 합니다.

**Loan**: 한 번의 대출 이벤트를 표현합니다. `returned_at`이 `None`이면 현재 대출 중입니다. `extension_count`로 몇 번 연장했는지 추적합니다.
