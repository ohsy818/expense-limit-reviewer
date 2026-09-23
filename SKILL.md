---
name: expense-limit-reviewer
description: >
  경비 관리 에이전트가 '규정·한도 검토' 업무에 사용하는 전용 스킬. 경비 청구 건의 증빙 첨부 여부 확인 → 경비유형/금액 식별 → '경비 한도 승인 판단' 비즈니스 규칙으로 자동승인/팀장승인 판정 → 검토 결과 표 정리 → 누락/초과 사유 명시까지 일련의 절차를 표준화한다.
---

# expense-limit-reviewer

## 언제 쓰나
- 다음 키워드에 반드시 트리거: "규정·한도 검토", "경비 한도", "경비 승인 구분", "expense limit review", "approval routing for expenses" 등.
- 입력이 경비 청구 단건 또는 일괄 다건이어도 동작. 다건 시 각 건별로 독립 판정 후 요약표 병합.

## 입력
- 필수: 청구자(name or id), 경비유형(type), 청구금액(amount.value, amount.currency), 발생일(date), 증빙첨부여부(has_receipt: boolean), 증빙파일목록(evidences: [filename or id])
- 선택: 부서(department), 직책/직급(grade), 프로젝트/코스트센터(cost_center), 지역(region), 결제수단(payment_method), 메모(note)
- 규칙 평가 입력 매핑: { expense_type: 경비유형, amount: 청구금액.value, currency: 청구금액.currency, grade, region, payment_method }

## 절차
1. 증빙 확인: has_receipt가 false이거나 evidences가 비어 있으면 issues에 "증빙 누락"을 추가하고 approval_category를 임시로 "보완요청"으로 설정한다.
2. 경비유형·금액 식별: expense_type이 없거나 공백이면 issues에 "경비유형 미확인"을 추가한다. amount.value가 숫자가 아니면 issues에 "금액 미확인"을 추가한다. 둘 중 하나라도 치명 결함이면 승인구분은 "보완요청"을 유지한다.
3. '경비 한도 승인 판단' 비즈니스 규칙 호출: functions.get_process_list(type='dmn')로 이름에 "경비 한도 승인 판단"이 포함된 항목을 찾고, 있으면 functions.get_process_detail(process_id)로 세부 정보를 확인한 뒤 외부 규칙 엔진을 통해 evaluate 한다. 입력 매핑은 { expense_type, amount, currency, grade, region, payment_method }를 그대로 전달한다. 기대 출력은 { limit: {value, currency}, approval: "자동승인"|"팀장승인", basis: string }이다. DMN 미탑재/호출 실패 시 approval_category는 기존값(보완요청 또는 null)을 유지하고, rationale에 "규칙 미탑재/호출 실패"를 기록하며 rule_limit와 is_over_limit는 null로 둔다. 임의 한도값은 생성하지 않는다.
4. 결과 표준화: rule_limit와 amount.currency가 다르면 통화 변환을 하지 않고 한도 통화만 그대로 표기한다. rule_limit가 존재할 때만 is_over_limit를 계산한다. 승인구분은 규칙 출력 approval을 우선 반영하되, 증빙 누락/결함이 있으면 "보완요청"으로 강등한다.
5. 표 생성: 열 순서 [청구자, 경비유형, 청구금액, 한도, 초과여부, 승인구분, 근거]로 마크다운 표를 만든다. 각 값은 다음과 같이 구성한다: 청구자=claimant, 경비유형=expense_type, 청구금액=`${amount.value} ${amount.currency}`, 한도=rule_limit? `${limit.value} ${limit.currency}` : "-", 초과여부=rule_limit? (is_over_limit? "예":"아니오") : "-", 승인구분=approval_category, 근거=rationale(규칙 basis 또는 누락/오류 요약).

## 출력 계약
- expense_reviews: 표준 JSON 결과 배열. 각 원소는 { claimant, expense_type, amount:{value,currency}, rule_limit:{value,currency}|null, is_over_limit:boolean|null, approval_category:"자동승인"|"팀장승인"|"보완요청", rationale:string, issues:string[] } 스키마를 따른다.
- summary_table: 마크다운 표 문자열. 열 순서 고정 [청구자, 경비유형, 청구금액, 한도, 초과여부, 승인구분, 근거]. 금액 표시는 ko-KR 천단위 구분 권장.

## 주의
- 이 스킬은 한도/승인 로직을 외부 DMN 규칙("경비 한도 승인 판단")에 위임한다. 규칙이 없거나 호출에 실패하면 한도/초과여부를 임의 계산하지 않는다.
- 다건 입력은 각 건 독립 처리 후 표를 병합한다. 누락/오류 사유는 세미콜론(;)으로 합쳐 근거에 1문장 요약으로 기재한다.
- 영수증 대체 인정: DMN basis에 "영수증 대체 인정"이 명시되면 증빙 누락이 있어도 승인구분을 보완요청으로 강등하지 않는다.
- 발생일은 한도 판단의 보조정보일 뿐이며, 달별 한도 판단은 DMN에서 처리한다고 가정한다.