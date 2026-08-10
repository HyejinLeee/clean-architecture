# 1-02. 도메인 모델 설계

앞의 **1-01**에서 정한 규칙들은 결국 **누군가 책임지고 지켜야** 합니다. 규칙을 대상별로 묶어 맡기면, 자연스럽게 세 개의 Entity가 나옵니다.

| 1-01의 규칙 | 이 규칙을 책임질 Entity |
|---|---|
| 재고 관리 | **Book** |
| 회원당 한도 · 연체 제한 | **Member** |
| 대출/연장 기간 | **Loan** |

그럼 이 세 Entity가 어떻게 관계를 맺는지 그려봅니다. Loan이 Book과 Member를 연결하는 중심 역할을 합니다.

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

## 이 필드들은 어떻게 나왔나

위 다이어그램의 필드들은 두 가지 질문에서 나옵니다.

**질문 1. 현실에서 이걸 뭐라고 설명하지?** — 대상 자체를 나타내는 기본 정보입니다.

- 책 → 제목(`title`) · 저자(`author`) · 몇 권 보유(`total_copies`)
- 회원 → 이름(`name`)
- 대출 한 건 → 누가·무엇을·언제(`member_id` · `book_id` · `borrowed_at` · `due_at`)

현실의 대상을 그대로 옮긴 것이라 떠올리기 쉽습니다.

**질문 2. 1-01의 규칙을 지키려면 뭘 더 알아야 하지?** — 규칙에서 역산해 나오는 값입니다.

- "재고 없으면 불가" → 지금 빌려줄 수 있는 권수 → `available_copies`
- "최대 5권" → 이 회원이 지금 몇 권 빌렸나 → `active_loans_count`
- "연체 중이면 불가" → 이 회원이 연체 중인가 → `has_overdue_loan`
- "연장 2회까지" → 몇 번 연장했나 → `extension_count`

질문 1은 쉽지만, 이 예제의 진짜 핵심은 **질문 2**입니다. 규칙에서 필드를 역산하는 구체적인 방법은 **1-04(STEP 1)** 에서 더 파고듭니다.

## 설계 포인트

앞에서 각 Entity가 **어떤 필드를 왜 가지는지** 봤으니, 여기서는 각 Entity가 **무엇을 하는지**(동작·상태 의미)를 짚습니다.

**Book**: `borrow_copy()`로 재고를 줄이고 `return_copy()`로 되돌립니다.

**Member**: 대출 시 `assert_can_borrow()`로 자격을 확인하고, 대출·반납에 따라 대출 권수를 갱신합니다.

**Loan**: 한 번의 대출 이벤트를 표현합니다. `returned_at`이 `None`이면 현재 대출 중이고, `extension_count`로 몇 번 연장했는지 추적합니다.

> 참고로 질문 2의 값들(`available_copies` 등)을 실제로 DB에 저장하는지, 아니면 그때그때 계산하는지는 바로 다음 **1-03(DB 구조)** 에서 다룹니다.
