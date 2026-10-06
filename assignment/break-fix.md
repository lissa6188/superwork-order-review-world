# Break → Fix → Re-run

> **I changed the World, not the prompt.**

## 1. BREAK — World를 깨뜨려 보기

여러분의 World가 막아야 하는 일을 하나 골라, 그 일이 **일어나도록** 시도하는 시나리오를 씁니다.
(`../scenarios/adversarial.yaml`)

공격 유형 예시:

- 권한 없는 행위자 — 그 역할이 없는 사람이 행동한다
- 셀프 승인 — 요청한 사람이 스스로 승인한다
- 잘못된 상태에서의 전이 — 아직 도달하지 않은 상태에서 행동한다
- 한도 초과 — 금액 · 수량이 정책 한도를 넘는다
- 선행 조건 누락 — 필요한 단계(검토 등)를 건너뛴다
- 중복 행동 / 중복 효과 — 같은 행동이나 외부 효과가 두 번 일어난다

## 2. BEFORE

| 항목 | 기록 |
|---|---|
| 시나리오 (파일 · 이름) | `scenarios/adversarial.yaml` / `confirm-rule-without-evidence` (공격 유형: 선행 조건 누락) |
| 지키려는 안전 속성 | 확정되는 검토규칙은 과거 검토의견서에서 지지 근거가 1건 이상 있어야 합니다. 근거 없는 규칙이 검토의견서 초안에 쓰이면 안 되기 때문입니다. |
| 시작 레코드와 상태 | `R-B9` (ReviewRule), `rule_in_review`, `supporting_evidence_count: 0`, 소속 세트 `RS-002` (`set_learning`) |
| 행위자 / 역할 | `sapjiler` (빌더) / `rule_reviewer` |
| 시도한 Transition | `rule_confirm` (rule_in_review에서 rule_confirmed로) |
| 관련 Constraint | 없음 — `rule_confirm`의 guards가 `[]` |
| **예상 판정** | **ALLOW** |
| 이유 (desk-check) | 상태는 R-B9가 rule_in_review라서 rule_confirm.from과 일치하고, 역할은 sapjiler가 rule_reviewer를 가졌고, guard는 없으니 ALLOW입니다. |
| 문제 | 근거 수를 확인하는 규칙이 빠져 있었습니다. 지지 근거가 0건인 규칙이 확정되고, effect `decrease_pending_review_rule_count`로 RS-002의 검수대기 수가 줄어서 운영 전환 승인 조건(`C2_set_no_pending_review`)까지 채워질 수 있습니다. 검수자가 사람이어도 실수하거나 지쳐서 근거를 확인하지 않고 확정할 수 있는데, World가 그걸 막지 못했습니다. |

## 3. FIX — World 규칙을 바꾸기

프롬프트나 코드가 아니라 **`world.yaml`의 규칙**을 바꿉니다.

- 바꾼 것: 새 Constraint `C13_rule_has_supporting_evidence`를 추가하고 `rule_confirm`의 guard에 붙였습니다.
- 변경 전 → 변경 후 (해당 부분만 붙여 넣기):

```yaml
# before
transitions:
  rule_confirm:
    from: rule_in_review
    to: rule_confirmed
    principal_role: rule_reviewer
    guards: []
    effects: [decrease_pending_review_rule_count]

# after
transitions:
  rule_confirm:
    from: rule_in_review
    to: rule_confirmed
    principal_role: rule_reviewer
    guards: [C13_rule_has_supporting_evidence]
    effects: [decrease_pending_review_rule_count]
constraints:
  C13_rule_has_supporting_evidence:
    expr: ReviewRule.supporting_evidence_count > 0
    message: 지지 근거가 1건 이상인 규칙만 확정할 수 있다.
```

- `npm run validate` 결과 (수정 후) → `../evidence/`에 저장
  - `evidence/validate-before.txt`: Constraints 12, 전 항목 PASS, READY FOR STUDIO IMPORT ✓
  - `evidence/validate-after.txt`: Constraints 13, 전 항목 PASS, READY FOR STUDIO IMPORT ✓
  - 수정 전 World도 validate를 통과했습니다. valid한 World라고 해서 안전한 건 아닙니다.

## 4. AFTER — 같은 시나리오, 다시 판정

| 항목 | 기록 |
|---|---|
| 같은 시나리오 | `confirm-rule-without-evidence` — sapjiler가 R-B9(rule_in_review, 지지 근거 0건)에 rule_confirm |
| **새 예상 판정** | **DENY (C13_rule_has_supporting_evidence)** |
| 이유 — 이제 어떤 규칙이 판정을 바꾸는가 | 상태와 역할은 그대로 통과하지만, guard C13의 `ReviewRule.supporting_evidence_count (0) > 0`이 거짓이라 DENY입니다. 확정되지 않으므로 effect `decrease_pending_review_rule_count`도 일어나지 않고, RS-002의 검수대기 수는 그대로 남아 C2가 운영 전환 승인을 계속 막습니다. |

## 5. 리뷰에서 (강사 기록란 — 비워 두세요)

| | 여러분의 예측 | 실제 Runtime 결과 |
|---|---|---|
| BEFORE | | |
| AFTER | | |

---

## 예시 (작성 방법 참고용)

구매 요청 World에서 `approve`(submitted → approved, 역할 manager)에 금액 한도 `C2_amount_limit`만 있었습니다.
Alex는 employee이면서 manager입니다.

- **BEFORE** — Alex가 자신이 제출한 R-2001(submitted, 1,200)을 approve.
  상태 일치(submitted) · 역할 일치(manager) · C2(1,200 ≤ 5,000) 통과 → **예상 ALLOW**.
  문제: 요청자와 승인자가 같은지 확인하는 규칙이 없다.
- **FIX** — 새 Constraint를 추가하고 `approve`의 guard에 붙였다.

  ```yaml
  constraints:
    C3_no_self_approval:
      expr: principal.id != request.requester
      message: Nobody can approve their own request.
  transitions:
    approve:
      guards: [C2_amount_limit, C3_no_self_approval]
  ```

- **AFTER** — 같은 시나리오. C3에서 `alex != alex`가 거짓 → **예상 DENY (C3_no_self_approval)**.

수정 전 World도 `npm run validate`를 통과했다는 점에 주목하세요 — **valid한 World도 안전하지 않을 수 있습니다.**
