# Spec 02 — 토스페이먼츠 API 계약 (모의 PG 기준)

작성: 2026-11-22. 출처는 토스페이먼츠 공식 문서(`docs.tosspayments.com`)의 reference, reference/using-api/authorization, reference/using-api/webhook-events, guides/v2/webhook, reference/error-codes 입니다.
범위는 `01-pg-reference.md`의 모의 PG 범위(카드 결제 승인·취소·조회·웹훅)입니다.
"문서에 없음"은 공식 문서에서 확인하지 못했다는 뜻이며, 그 항목은 **모의 PG 자체 결정**으로 표시하고 7절에 모았습니다.

## 1. 인증

- Basic 인증: `Authorization: Basic base64("{secretKey}:")` (콜론 필수, 비밀번호 없음)
- 키 접두사: `test_sk_`, `live_sk_`. 모의 PG는 `test_sk_` 형식만 허용합니다.
- 오류: 키 불일치 401 `UNAUTHORIZED_KEY`, 콜론 누락 403 `INCORRECT_BASIC_AUTH_FORMAT`

## 2. 엔드포인트

| 용도 | 메서드·경로 | 비고 |
|---|---|---|
| 승인 | `POST /v1/payments/confirm` | body: `paymentKey`, `orderId`, `amount` (모두 필수) |
| 취소 | `POST /v1/payments/{paymentKey}/cancel` | body: `cancelReason`(필수), `cancelAmount`(선택, 없으면 전액) |
| 조회(paymentKey) | `GET /v1/payments/{paymentKey}` | |
| 조회(orderId) | `GET /v1/payments/orders/{orderId}` | |

- 제약: `paymentKey` 최대 200자, `orderId` 6~64자(영문·숫자·`-`·`_`·`=`), `cancelReason` 최대 200자
- 성공 응답은 Payment 객체이고, 성공 시 HTTP 상태 코드는 문서에 없습니다. 모의 PG는 200을 사용합니다.
- 취소 요청의 `cancelRequestId`는 비동기 결제 전용이라 제외합니다. 가상계좌·환불계좌·`currency` 관련 파라미터도 제외합니다.

## 3. Payment 객체 (카드 결제에 쓰는 필드만)

문서는 필수·선택을 따로 표기하지 않고 nullable 여부만 표시합니다.

| 필드 | 타입 | null | 비고 |
|---|---|---|---|
| `version` | string | 아니오 | 날짜 기반 버저닝 |
| `paymentKey` | string | 아니오 | |
| `type` | string | 아니오 | `NORMAL`만 사용 |
| `orderId`, `orderName` | string | 아니오 | orderName 최대 100자 |
| `mId`, `currency` | string | 아니오 | currency는 `KRW`만 사용 |
| `method` | string | 예 | `카드` |
| `totalAmount` | number | 아니오 | |
| `balanceAmount` | number | 아니오 | 취소 가능 잔액 |
| `status` | string | 아니오 | 4절 참고 |
| `requestedAt`, `approvedAt` | string | approvedAt만 예 | ISO 8601 |
| `lastTransactionKey` | string | 예 | 최대 64자 |
| `suppliedAmount`, `vat`, `taxFreeAmount` | number | 아니오 | |
| `cancels` | array | 예 | Cancel 배열(부분 취소마다 누적) |
| `isPartialCancelable` | boolean | 아니오 | |
| `card` | object | 예 | amount, issuerCode, acquirerCode, number, installmentPlanMonths, approveNo, cardType, ownerType 등 |
| `secret` | string | 예 | 웹훅 검증값, 최대 50자 |
| `failure` | object | 예 | `{code, message}` |
| `metadata` | object | 예 | |

제외: escrow, virtualAccount, mobilePhone, giftCertificate, transfer, easyPay, cashReceipt(s), discount, checkout, receipt, 그리고 deprecated 필드

### Cancel 객체 (`cancels[]`)

`cancelAmount`, `cancelReason`, `taxFreeAmount`, `taxExemptionAmount`, `refundableAmount`(취소 후 잔액), `canceledAt`, `transactionKey`(최대 64자), `receiptKey`(nullable), `cancelStatus`(`DONE`이면 성공)
- `cancelStatus`의 `DONE` 외 값은 문서에 없습니다.

## 4. 결제 상태 (카드 흐름)

| status | 의미 |
|---|---|
| `READY` | 결제 생성 직후, 인증 전 |
| `IN_PROGRESS` | 인증 완료. 승인 API 호출 대기 |
| `DONE` | 승인 완료 |
| `PARTIAL_CANCELED` | 부분 취소됨 (잔액 > 0) |
| `CANCELED` | 전액 취소됨 |
| `ABORTED` | 승인 실패 |
| `EXPIRED` | 승인 대기 중 유효 시간 경과 |

- 제외: `WAITING_FOR_DEPOSIT` (가상계좌 전용)
- 전이 (공식 문서 "일반 결제 상태(status) 흐름" 그림 기준, 사용자가 스크린샷으로 확인, 2026-11-22):
  - `READY → IN_PROGRESS` (결제 진행 중), `READY → EXPIRED` (유효 시간 만료)
  - `IN_PROGRESS → DONE` (승인 성공), `IN_PROGRESS → ABORTED` (승인 실패), `IN_PROGRESS → EXPIRED` (유효 시간 만료)
  - `DONE → CANCELED` (결제 취소), `DONE → PARTIAL_CANCELED` (결제 부분 취소)
- **웹훅 전송 상태(그림의 파란색):** `DONE`, `ABORTED`, `EXPIRED`, `CANCELED`, `PARTIAL_CANCELED`. `READY`와 `IN_PROGRESS`로의 전이는 웹훅이 없다.
- **그림에 없는 전이(우리의 가정):** 부분 취소 후의 전이(`PARTIAL_CANCELED → CANCELED`, 반복 부분 취소)는 그림에 나오지 않는다. 토스가 부분 취소를 여러 번 허용하고 `cancels`에 누적한다고 했으므로 모의 PG는 이를 허용하지만, 토스의 실제 동작은 확인하지 못했다.
- 그림에는 만료 시간 값이 없어 10분 vs 30분 불일치는 그대로다. `READY`의 만료 시간도 문서에 없어 모의 PG 설정값으로 둔다.
- **문서 내 불일치:** 승인 설명은 "인증 후 10분 안에 승인하지 않으면 만료"라고 하고 status 설명(EXPIRED)은 30분이라고 합니다. 문서에 이유가 없으므로 5절에서 모의 PG 값을 정합니다.

## 5. 멱등성 (`Idempotency-Key`)

| 항목 | 문서 내용 |
|---|---|
| 헤더 | `Idempotency-Key`, 최대 300자(UUID 권장) |
| 보관 기간 | 최초 요청일부터 15일 |
| 대상 | 모든 POST. GET에서는 무시 |
| 같은 키 재요청 | 재처리 없이 첫 응답을 반환 |
| 키 범위 | 상점 API 키 + API 주소 + HTTP 메서드 조합 |
| 길이 초과 | 400 `INVALID_IDEMPOTENCY_KEY` |
| 첫 요청 처리 중 | 409 `IDEMPOTENT_REQUEST_PROCESSING` |
| 에러 후 키를 바꿔 재시도 | 위험하다고 명시 (원인 확인 후 재요청) |
| 같은 키에 본문이 다름 | 문서에 없음 |

- 주의: `IDEMPOTENT_REQUEST_PROCESSING`과 `INVALID_IDEMPOTENCY_KEY`는 authorization 페이지에만 나오고 error-codes 페이지에는 없습니다. error-codes 페이지에는 `ALREADY_PROCESSING_REQUEST`(400)와 `DUPLICATED_REQUEST`(400)가 있지만 멱등키와 연결된다는 설명은 없습니다. 모의 PG는 authorization 페이지 기준을 따릅니다.

## 6. 웹훅 (카드 결제에 해당하는 부분)

- 이벤트: `PAYMENT_STATUS_CHANGED` 하나만 구현합니다. 카드 결제는 상태 변경 시(DONE, 취소, EXPIRED 등) 이 이벤트로 전달됩니다. `CANCEL_STATUS_CHANGED`는 비동기 해외 간편결제 전용이라 제외합니다.
- 본문: `eventType`, `createdAt`(`yyyy-MM-dd'T'HH:mm:ss.SSSSSS`, 오프셋이 없으면 KST), `data`(Payment 객체)
- **`PAYMENT_STATUS_CHANGED`에는 `eventId`가 없습니다.** 문서가 권하는 중복 판별 키(`eventId`)를 쓸 수 없으므로, 수신 측은 `paymentKey` + `status` + `lastTransactionKey` 같은 조합으로 멱등 처리를 설계해야 합니다. 이 부분은 이 프로젝트에서 보여줄 만한 설계 주제입니다. (조합 자체는 제 제안이며 토스 문서의 권고가 아닙니다.)
- 전송: HTTP POST(JSON), 헤더 `tosspayments-webhook-transmission-time`, `tosspayments-webhook-transmission-retried-count`, `tosspayments-webhook-transmission-id`(전송마다 새로 발급되므로 중복 판별에 쓰지 말 것), `tosspayments-webhook-signature`
- 서명: `{payload}:{transmission-time}`을 보안 키로 HMAC SHA-256 → 헤더의 `v1:` 뒤 값(여러 개 가능, 쉼표 구분)을 base64 디코딩해 비교. 하나라도 일치하면 유효합니다. 보안 키의 발급 위치는 읽은 페이지에 없습니다.
- **주의(정정):** 위 서명 방식이 카드 결제 웹훅(`PAYMENT_STATUS_CHANGED`)에 적용되는지는 확인하지 못했다. 보안 키 설명이 지급대행 가이드(`/guides/v2/payouts#보안-키`)로 연결되어 있어 해당 이벤트에만 적용될 가능성이 있다. 모의 PG는 이 방식을 차용해 모든 웹훅에 서명하지만, 이것을 토스 규격이라고 말하지 않는다(`03`의 4-7절).
- 응답과 재전송: 10초 안에 HTTP 200이면 성공이고, 문서는 다른 2xx에 대해 언급하지 않습니다. 실패 시 최대 7회 재전송하며 간격(분)은 1, 4, 16, 64, 256, 1024, 4096입니다(마지막 재전송은 최초 전송 후 약 3일 19시간).
- 순서 보장: `PAYMENT_STATUS_CHANGED`에 대한 언급은 문서에 없습니다. (`escrow.changed`만 순서 비보장을 명시하고 "상태 역행 없음, 늦게 온 이벤트는 폐기" 원칙을 설명합니다. 이 원칙을 참고할 수는 있습니다.)
- 조회 API로 재검증하라는 권고는 읽은 페이지에 없습니다. 수신 측에서 재조회해 검증하는 것은 이 프로젝트의 설계 선택으로 둡니다.

## 7. 에러 응답과 에러 코드

본문은 `{ "code": "...", "message": "..." }` 형태입니다. 메시지는 한국어 원문을 그대로 씁니다.

### 승인
| HTTP | code | 모의 PG 사용 |
|---|---|---|
| 400 | `ALREADY_PROCESSED_PAYMENT` | 이미 승인된 결제에 재승인 |
| 400 | `ALREADY_PROCESSING_REQUEST` | 동시 승인 요청 |
| 400 | `INVALID_REQUEST` | 파라미터 오류, 금액 불일치(7-1 참고) |
| 400 | `BELOW_MINIMUM_AMOUNT` | 100원 미만 |
| 400 | `PROVIDER_ERROR` | 일시 오류(재시도 가능) |
| 401 | `UNAUTHORIZED_KEY` | 키 인증 실패 |
| 403 | `REJECT_CARD_PAYMENT` | 한도초과·잔액부족 |
| 403 | `REJECT_CARD_COMPANY` | 카드사 거절 |
| 404 | `NOT_FOUND_PAYMENT_SESSION` | 세션 만료 |
| 500 | `FAILED_INTERNAL_SYSTEM_PROCESSING` | 내부 오류 |
| 500 | `FAILED_CARD_COMPANY_RESPONSE` | 카드사 응답 오류 |
| 500 | `UNKNOWN_PAYMENT_ERROR` | 원인 미상 |

### 취소
| HTTP | code | 모의 PG 사용 |
|---|---|---|
| 400 | `ALREADY_CANCELED_PAYMENT` | 이미 전액 취소됨 |
| 400 | `INVALID_REQUEST` | 파라미터 오류 |
| 400 | `PROVIDER_ERROR` | 일시 오류 |
| 403 | `NOT_CANCELABLE_AMOUNT` | 취소 금액이 잔액 초과 (7-1 참고) |
| 403 | `NOT_CANCELABLE_PAYMENT` | 취소할 수 없는 상태 |
| 403 | `EXCEED_CANCEL_LIMIT` | 취소 한도 초과 |
| 404 | `NOT_FOUND_PAYMENT` | 결제 없음 |
| 500 | `FAILED_METHOD_HANDLING_CANCEL` | 결제수단 처리 중 오류 |

### 조회
`404 NOT_FOUND_PAYMENT`, `401 UNAUTHORIZED_KEY`, `403 FORBIDDEN_CONSECUTIVE_REQUEST`(반복 요청 제한)

### 7-0. 확정된 결정 (2026-11-22, 사용자 확인)

| 항목 | 결정 | 비고 |
|---|---|---|
| 같은 멱등키 + 다른 본문 | **422 거부**, 첫 응답은 재생하지 않음. 에러 코드는 모의 PG 자체 정의(`IDEMPOTENCY_KEY_REUSED`) | 토스 문서에 동작 없음. Stripe 방식을 참고 |
| 멱등키가 `IN_PROGRESS`로 남은 경우 | **타임아웃 후 상태 확인**: N초(초기값 30초) 경과 후 재요청이 오면 결제의 실제 상태를 조회해 `COMPLETED`로 확정하거나, 변경이 없었으면 재처리 | 서버 크래시 복구 시나리오로 테스트 |
| 웹훅 중복 판별 키 | **`paymentKey + status + lastTransactionKey`** 조합에 DB 유니크 제약 | 토스 권고가 아닌 이 프로젝트의 설계. 토스 `PAYMENT_STATUS_CHANGED`에는 `eventId`가 없음 |
| 승인 시 금액 불일치 | `INVALID_REQUEST` (400) | 토스에 전용 코드 없음. 실제 동작은 미확인 |

| 승인 만료 시간 | **10분 (설정값)** | 승인 API 설명 기준. status 설명의 30분과 문서가 불일치하여 직접 규정한 쪽을 택함 |

| 취소 가능 금액 초과 | **`NOT_CANCELABLE_AMOUNT` (403)** | 의미: 잔액(`balanceAmount`)보다 큰 `cancelAmount`로 취소 요청. 토스 문서에 이 경우의 코드가 명시되어 있지 않아 이름이 가장 직접적인 코드를 택함(실제 토스 동작은 미확인). 403은 토스 문서의 HTTP 상태를 따른 것이며, 서비스 내부는 어댑터에서 `code` 기준 도메인 예외로 변환 |

7-1절의 미결 항목은 모두 확정됨 (성공 응답 200, 웹훅 보안 키는 모의 PG 설정값 포함)

### 7-1. 문서에 없어서 모의 PG가 정해야 하는 것 (원래 제안, 위 7-0에서 확정된 항목 포함)

- **금액 불일치 전용 코드가 없습니다.** 승인 시 요청 `amount`가 서버의 주문 금액과 다를 때는 `INVALID_REQUEST`(400)를 쓰기로 제안합니다. 실제 토스가 어떻게 하는지는 확인하지 못했습니다.
- **취소 가능 금액 초과 전용 코드가 없습니다.** `NOT_CANCELABLE_AMOUNT`(403)를 쓰기로 제안합니다. 문서가 어느 코드가 이 경우인지 명시하지 않으므로 실제와 다를 수 있습니다.
- 같은 멱등키에 다른 본문을 보낸 경우의 동작: 문서에 없음 → 제안은 409 또는 400 에러로 거부하는 것이며 이 결정은 다음 단계에서 확정합니다.
- 승인 만료 시간(10분 vs 30분): 제안은 10분이며 설정값으로 둡니다. 근거는 승인 API 설명이 직접 규정한 값이라는 점입니다.
- 성공 응답 HTTP 코드: 200
- 웹훅 보안 키: 모의 PG 설정값으로 정의
- 웹훅 순서 뒤바뀜·중복 전송은 장애 주입으로 재현하고, 수신 측에서 상태 역행을 폐기하는 규칙은 `escrow.changed` 설명을 차용합니다.

## 8. 남은 확인 사항

- ~~상태 전이 흐름도 원문 대조~~ → 4절에 반영(사용자가 공식 문서 그림 스크린샷으로 확인). 남은 것: 부분 취소 이후 전이와 `READY` 만료 시간은 문서에 없음
- 성공 응답 코드, `Idempotency-Key`의 다른 본문 충돌 동작은 실제 샌드박스로 확인하지 않는 한 확정할 수 없습니다 (실제 PG 샌드박스 연동은 범위 밖).
- 에러 코드 문서와 authorization 문서의 멱등성 코드 불일치
