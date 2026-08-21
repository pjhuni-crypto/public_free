# 도메인 정의서 — 카카오톡 + LLM 시설물 사용신청 챗봇

이 문서는 시스템이 다루는 핵심 개념(엔티티)과 용어, 상태값, 관계를 정의한다. 챗봇/LLM 레이어와 예약 API(현재는 Mock, 향후 기존 예약 시스템에 노출될 API) 양쪽 모두 이 정의를 기준으로 구현해야 한다. API의 형식적 정의(요청/응답 스키마)는 구현 시 `openapi/reservation.yaml`로 옮겨지며, 이 문서는 그 스키마가 표현하는 개념을 사람이 읽을 수 있게 설명한다.

## 핵심 엔티티

### Facility (시설)

울산 혁신도시 공공기관이 공용으로 제공하는 예약 대상 시설. 현재 6개 고정(야외공연장, 풋살장, 테니스 코트1/2(본사), 테니스 코트1/2(일산)).

| 속성 | 설명 |
|---|---|
| id | 시설 식별자 |
| name | 시설명 |
| type | 시설 종류(공연장/풋살장/테니스코트 등) |
| weeklyHours | 요일별 이용 가능 시간대(평일 18~21시는 야외공연장만, 휴일 10~17시, 풋살장은 일요일 11시 이후만 등 — 정책 값이므로 수치는 이 문서에 고정하지 않고 API 응답으로 조회) |
| maxHoursPerDayPerPerson | 1인 1일 최대 이용시간(현재 정책: 4시간) |

### Application (신청)

한 사용자가 특정 시설의 특정 날짜·시간대를 이용하겠다고 접수한 요청.

| 속성 | 설명 |
|---|---|
| applicationId | 신청 식별자 |
| facilityId, date, startTime, endTime | 신청 대상 시설·일정(시간은 1시간 단위 다중 선택 가능, 1인 1일 4시간까지) |
| applicant.name, applicant.phone | 신청자 정보 — **카카오인증서로 검증된 값**을 그대로 사용(사용자가 직접 타이핑한 값을 신뢰하지 않음) |
| headcount | 예약인원(명) |
| email | 이메일 |
| groupName | 단체명 |
| workPhone, homePhone | 직장/자택 전화번호(선택) |
| vehicle | 차량번호·종류, 사옥주차장 사용 여부 |
| purpose | 방문목적(자유 텍스트) |
| managePassword | 신청 건 조회/취소용 비밀번호(로그인 없이도 본인 신청을 확인할 수 있게 하는 값 — 실제 웹사이트 폼에도 동일 항목 존재) |
| verifiedIdentity | 스킬서버가 카카오인증서로 인증을 완료하고 넘겨주는 결과(provider, 트랜잭션ID, 검증된 이름·전화번호). 예약서버는 이 값이 스킬서버(신뢰된 클라이언트)에서 왔다는 것만 서버간 인증으로 확인하고, 그 내용 자체를 다시 검증하지 않는다 |
| status | 상태값(아래 상태 전이 참조) |

**상태 전이**:

```
(생성) → PENDING_VERIFICATION → [본인인증 완료] → APPROVED
                              → [본인인증 실패/만료] → REJECTED
```

- "승인(APPROVED)"은 사람이 검토하는 것이 아니라 **정책 조건을 만족하면 시스템이 즉시 자동으로 부여**하는 상태다. 즉 본인인증이 끝나면 곧바로 승인 여부가 결정된다.
- 신청 생성 시점에 정책 위반(아래 Policy 참조)이 있으면 신청 자체가 생성되지 않고 에러코드로 거부된다(신청이 REJECTED 상태로 남는 것이 아니라, 애초에 생성되지 않음).

### IdentityVerification (본인인증)

**카카오 인증서**(카카오써트 전자서명, 앱투앱/채널메시지 인증)를 통해 카카오톡 사용자가 본인임을 확인하는 절차. 하나의 Application에 하나씩 연결된다.

이 엔티티는 **예약서버가 아니라 스킬서버가 소유**한다 — 카카오인증서는 카카오 채널에 종속된 인증수단이라, 카카오톡 사용자 컨텍스트를 이미 갖고 있는 스킬서버가 직접 카카오인증서 API(또는 프로토타입에서는 Mock)와 통신한다. 예약서버는 이 엔티티를 API로 노출하지 않으며, 신청 생성(`POST /applications`) 시점에 이미 검증이 끝난 결과(`verifiedIdentity`)만 전달받는다.

| 속성 | 설명 |
|---|---|
| identityVerificationId | 인증 세션 식별자(스킬서버 내부 관리용) |
| provider | 인증 제공자 — 현재는 `KAKAO_CERT` 고정(프로토타입에서는 `MOCK`) |
| status | `PENDING`(승인 대기) / `VERIFIED`(완료) / `FAILED`(거부/인증 실패) / `EXPIRED`(시간 초과) |
| verifiedName, verifiedPhone | 인증 완료 시 카카오인증서가 돌려주는 검증된 이름·전화번호 |

**용어 정의**: 이 시스템에서 "인증"은 카카오 인증서를 통한 본인인증을 의미하며, 브라우저로 이동하는 방식이 **아니다** — 카카오톡 앱 안에서 알림을 받고 지문/PIN으로 승인하면 끝난다(NICE처럼 통신사 선택·문자 인증번호 입력 등의 단계가 없음). 그렇다고 마찰이 전혀 없는 것은 아니며, "알림 인지 → 생체인증/PIN 승인"이라는 본인확인 액션 자체는 남는다 — 이게 시니어에게 실제로 얼마나 쉬운지는 Mock으로 확인할 수 없고 실기기 테스트가 필요하다(`0-PRD.md` 열린 리스크 참조).

### ConversationSession (대화 세션)

카카오톡 사용자 한 명의 진행 중인 대화 상태. 여러 Application을 순차적으로 다룰 수 있지만, 한 시점에는 하나의 진행 중인 신청 흐름만 갖는다.

| 속성 | 설명 |
|---|---|
| step | 현재 대화 단계 — `FACILITY`(시설선택) / `TIME`(시간선택) / `APPLICANT_INFO`(신청자정보) / `CONFIRM`(확인) / `AUTH_PENDING`(본인인증 대기) / `DONE`(완료) |
| facilityId, date, startTime, endTime, applicantName, applicantPhone | 진행 중 수집된 값 |
| applicationId, identityVerificationId | 생성된 하위 엔티티 참조 |

세션은 사용자별로 유일하며(카카오 `userRequest.user.id` 기준), 30분간 응답이 없으면 만료된다.

### Policy (정책 — 값이 아니라 규칙의 이름)

아래 규칙들은 기존 웹 신청과 **동일하게 유지**되며, 최종 판단은 항상 예약 API(백엔드)가 내린다. 챗봇은 API가 돌려주는 에러코드를 사람말로 옮기기만 한다.

| 규칙 | 위반 시 에러코드 |
|---|---|
| 1인 1일 최대 이용시간(4시간) 초과 | `DAILY_LIMIT_EXCEEDED` |
| 월 단위 이용 한도 초과 | `MONTHLY_QUOTA_EXCEEDED` |
| 매월 25일 13시 이전에 다음 신청 기간 접수 시도 | `APPLICATION_WINDOW_NOT_OPEN` |
| 시설별 이용 가능 요일/시간대 밖의 신청 | `OUTSIDE_OPERATING_HOURS` |
| 이미 다른 신청으로 찬 시간대 신청 | `SLOT_UNAVAILABLE` |
| 본인인증 거부/재시도 초과/유효시간 초과 | `IDENTITY_VERIFICATION_FAILED` (인증 세션에 한정, 스킬서버가 관리) |

## 엔티티 관계

```
Facility 1 ──< Application ──(verifiedIdentity 필드로 결과만 전달)── IdentityVerification
                   ▲                                                      │
                   │ (참조)                                    스킬서버가 소유·관리
            ConversationSession                              (예약서버는 이 엔티티를 모름)
```

- 하나의 Facility는 여러 Application을 가질 수 있다(다른 날짜/시간대).
- 하나의 Application은 정확히 하나의 IdentityVerification을 가진다(재시도 시에도 상태를 갈아끼우는 것이지 여러 개가 동시에 존재하지 않는다). 다만 IdentityVerification은 **예약서버의 리소스가 아니라 스킬서버 내부 개념**이라, 예약서버 관점에서는 Application에 `verifiedIdentity`라는 필드로만 결과가 보인다.
- ConversationSession은 진행 중인 Application id와 스킬서버 내부의 identityVerificationId만 참조하며, 그 자체가 예약 데이터의 정본(source of truth)은 아니다 — 정본은 항상 예약 API(기존 시스템) 쪽에 있다.

## API 계약 요약

형식적 정의는 구현 시 `openapi/reservation.yaml`에 작성한다. **예약서버(백엔드)가 구현하는 API**는 다음과 같고, 카카오인증서 연동은 여기 포함되지 않는다(스킬서버가 별도로 직접 통신).

- `GET /facilities`
- `GET /facilities/{facilityId}/availability?date=`
- `GET /applicants/{phone}/quota?month=`
- `POST /applications` — body에 위 Application 표의 모든 필드 + `verifiedIdentity` 포함. 이 호출 자체는 스킬서버만 할 수 있도록 **서버간 API 키로 인증**한다.
- `GET /applications/{applicationId}`

**스킬서버가 직접 호출하는 외부 API**(예약 API 계약과 별개, 카카오써트 문서 기준):
- 카카오인증서 인증 요청 API — 사용자에게 인증 알림 발송
- 카카오인증서 인증 결과 조회 API — 인증 완료/실패 여부 확인

각 엔드포인트의 요청/응답 필드는 위 엔티티 표의 속성과 1:1로 대응한다.
