# 1-02. 도메인 모델 설계

세 Entity가 어떻게 관계를 맺는지 먼저 그려봅니다. Loan이 Book과 Member를 연결하는 중심 역할을 합니다.

```
  Book                  Loan                Member
──────────────    ──────────────────    ──────────────────
title, author     book_id, member_id    name
total_copies      borrowed_at, due_at   active_loans_count
available_copies  returned_at           has_overdue_loan
borrow_copy()     extend()              assert_can_borrow()
return_copy()     is_overdue()          register_borrow()
```

> Loan은 Book과 Member의 `id`만 참조합니다 — 서로 다른 Entity를 직접 참조하지 않아 결합도를 낮춥니다.

## 설계 포인트

**Book**: 재고(`available_copies`)를 기준으로 대출 가능 여부를 판단하고, `borrow_copy()`로 줄이고 `return_copy()`로 되돌립니다.

**Member**: 대출 자격을 스스로 판단합니다. 그러려면 현재 대출 권수(`active_loans_count`)와 연체 여부(`has_overdue_loan`)를 알아야 하므로, 이 두 값을 가집니다.

**Loan**: 한 번의 대출 이벤트를 표현합니다. `returned_at`이 `None`이면 현재 대출 중입니다. `extension_count`로 몇 번 연장했는지 추적합니다.

> 여기서는 각 Entity가 **무엇을 알고 무엇을 하는지**(속성·행동)만 봅니다. 이 값들을 실제로 DB에 저장하는지 아니면 계산하는지는 **1-03(DB 구조)** 에서, 왜 이런 필드를 두는지 판단하는 법은 **1-04(STEP 1)** 에서 다룹니다.
