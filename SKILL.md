---
name: expense-limit-reviewer
description: 경비 청구 건의 규정·한도 검토를 수행하는 스킬. 신청서 입력을 수집한 뒤 비즈니스 규칙엔진의 '경비 한도 승인 판단'을 호출해 승인구분/한도 초과 여부를 판정한다. EDU는 1인 연간 2,000,000원 기준(YTD 누적)으로 판정하며, DB 직접 조회·내부 스키마/보안 정책 언급은 금지한다. 규칙명/버전/판정시점을 결과 근거에 명기하고, 규칙엔진 장애 시 '판단 보류'로 처리 후 운영자에 알림, 감사로그를 기록한다.
---

# expense-limit-reviewer

## 언제 쓰나
경비 청구 1건 또는 여러 건에 대해 규정 적합성, 한도 초과 여부, 승인구분(자동승인/팀장승인)을 판정할 때 사용한다. 경비 지출결의 프로세스의 `규정·한도 검토` 활동이 이 스킬을 사용한다.

## 입력(표준)
- 필수: tenant_id, account_code(계정과목 코드, 예: TRV/MEAL/EDU/SMT 등), applicant(신청자 식별자), used_at(사용일, ISO-8601), amount(수치), currency(예: KRW)
- 선택: has_receipt(증빙 보유 여부), evidences(첨부 목록), purpose(사용 목적/메모)
- 매핑 규칙: 사용자 입력의 경비유형 텍스트는 내부 account_code로 매핑한다. 미확정 시 `유형 확인 필요`로 처리하고 규칙을 호출하지 않는다.

## 핵심 원칙
- 한도 판단은 반드시 비즈니스 규칙엔진 evaluate(rule='경비 한도 승인 판단')를 호출해 승인구분을 산정한다. expense_limit, expense_ledger 등 테이블을 직접 조회하지 않는다.
- EDU(교육훈련비)는 월 한도가 아니라 1인 연간 2,000,000원 기준으로 판정한다. 해당 직원의 당해연도 누적 사용액(YTD)을 기준으로 초과 여부가 계산되며, 엔진이 집계기간을 스스로 결정한다.
- 출력물의 근거에는 규칙명('경비 한도 승인 판단'), 규칙 버전, 판정시점(evaluated_at)을 명기한다.
- 결과 텍스트에 시스템 보안 관련 문구(예: RLS)나 내부 스키마/도구명/SQL을 포함하지 않는다.

## 절차(업데이트된 작업 순서)
1) 입력 수집
   - 신청서에서 tenant_id, account_code, applicant, used_at, amount, currency, has_receipt, evidences, purpose를 확보한다.
   - account_code가 불명확하면 `확인 필요`로 표기하고 보완요청으로 종료한다.

2) 규칙 호출
   - 규칙엔진 evaluate(
       rule='경비 한도 승인 판단',
       inputs={ tenant_id, account_code, applicant, used_at, amount, currency }
     )를 호출한다.
   - 엔진이 계정과목별 한도유형(월/연간)과 집계기간을 스스로 결정하도록 한다.
   - EDU의 경우 별도 처리 로직을 구현하지 않는다. 동일 입력을 전달하면 엔진이 1인 연간 한도(2,000,000원)와 당해연도 누적 사용액(YTD)을 적용한다.
   - 호출/응답 전문은 내부 로그에만 남기고, 결과물에는 규칙명/버전/판정시점만 노출한다.

3) 응답 파싱
   - 다음 필드를 수신해 사용한다:
     - approval(승인구분: 자동승인/팀장승인)
     - is_over_limit(초과 여부, boolean)
     - limit_amount(기준 한도금액)
     - period_type(월|연간), period_range(예: 2026-09 또는 2026)
     - used_amount_to_date(당해기간 기사용액), projected_total(반영 후 누적)
     - over_amount(초과금액, 미초과 시 0)
     - rule_name, rule_version, evaluated_at(ISO-8601)

4) 결과 작성(표준 템플릿)
   - 한도 판단 요약: 승인구분, 초과 여부, 초과 시 초과금액.
   - 계산 내역: 계정과목(account_code), 한도유형/기준기간(period_type/period_range), 기준 한도금액(limit_amount), 당해기간 기사용액(used_amount_to_date), 신청액(amount), 반영 후 누적(projected_total).
   - 근거: 규칙명=경비 한도 승인 판단, 규칙 버전(rule_version), 판정시점(evaluated_at).
   - 금지: DB 테이블명/SQL/보안정책(RLS)·도구 내부 동작 설명 언급 금지.

   표 컬럼(권장):
   `신청자 | 계정과목 | 한도유형/기간 | 기준 한도 | 당해기간 기사용 | 신청액 | 반영 후 누적 | 초과여부 | 초과금액 | 승인구분 | 근거(규칙명/버전/판정시점)`

5) 예외 처리
   - 규칙엔진 오류/타임아웃 시 DB 직접 조회 금지. 승인구분을 추정하지 않는다.
   - 상태를 `판단 보류`로 설정하고 운영자에 알림(재시도/수동 검토 요청). 결과 표에는 `판단 보류` 사유만 기재한다(내부 오류 세부정보/스택트레이스/도구명 금지).

6) 감사지표 기록(필수)
   - 규칙명(rule_name), 규칙 버전(rule_version), 판정시점(evaluated_at), 입력요약(tenant_id, account_code, applicant, used_at, amount, currency), 최종 승인구분(approval)을 감사로그에 저장한다.

## 출력
- 표준 마크다운 표 1개(위 컬럼 순서 권장)와, 표 하단에 팀장 승인 필요 건수/보완·보류 건수를 한 줄 요약.
- 필요 시 JSON 산출물 병행 가능(예시 키):
  {
    "expense_reviews": [
      {
        "applicant": "홍길동",
        "account_code": "EDU",
        "amount": {"value": 199000, "currency": "KRW"},
        "period": {"type": "연간", "range": "2026"},
        "limit_amount": 2000000,
        "used_amount_to_date": 198000,
        "projected_total": 397000,
        "is_over_limit": false,
        "over_amount": 0,
        "approval": "자동승인",
        "basis": {"rule_name": "경비 한도 승인 판단", "rule_version": "vX.Y.Z", "evaluated_at": "2026-09-22T07:28:59Z"}
      }
    ]
  }

## 예시(요약)
- 입력: 계정과목=EDU, 신청자=홍길동, 신청액=199,000원, 사용일=2026-09-21, 통화=KRW
- 규칙 결과: 연간 한도 2,000,000원, YTD 사용 198,000원, 반영 후 397,000원, 미초과, 승인=자동승인, 근거: 경비 한도 승인 판단 vX.Y.Z, 판정시점=2026-09-22T07:28:59Z
- 표 출력 요지: 초과여부=미초과, 초과금액=0원, 승인구분=자동승인, 근거에 규칙명/버전/판정시점 포함

## 품질 체크리스트
- [필수] 규칙엔진 evaluate(rule='경비 한도 승인 판단') 호출 로그 존재. 직접 SQL로 expense_limit/expense_ledger 조회 금지.
- [필수] EDU는 연간 2,000,000원 기준, YTD 합산 근거가 결과에 반영.
- [필수] 근거에 규칙명/버전/판정시점 포함.
- [금지] 시스템 보안/내부 스키마/도구명/SQL 언급.

## 하지 않는 것(안티패턴)
- 엔진을 우회해 월 한도를 자의로 계산하거나 DB에서 합계를 직접 쿼리하는 행위.
- EDU를 월 한도로 판정하는 행위.
- 장애 시 임의 승인구분 추정/가정.
- 결과에 RLS, 테이블명, SQL, 내부 도구 동작을 기재.
