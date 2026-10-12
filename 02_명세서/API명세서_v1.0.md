# 병동 근무표 서비스 API 명세서

| 문서 정보 | 내용 |
| --- | --- |
| 버전 | 1.0 |
| 기준 문서 | 병동 근무표 서비스 기능명세서 v1.0 |
| 기준일 | 2026년 10월 6일 |
| 문서 상태 | 기준안 |
| API 형식 | REST JSON over HTTPS |
| 개정 | 2026년 10월 7일 소셜 로그인(OAuth) 추가: 1.6, API-AUTH-06~10 |
| 기본 경로 | `/api/v1` |

이 문서는 기능명세서의 사용자 흐름을 서버 API 계약으로 변환한 기준안이다. 클라이언트 구현, 서버 구현, QA 시나리오가 같은 권한과 상태 규칙을 사용하도록 요청과 응답, 오류, 동시성 조건을 함께 정의한다.

## 1 공통 규칙

### 1.1 인증과 쿠키

- 인증은 액세스 토큰과 리프레시 토큰을 `HttpOnly`, `Secure`, `SameSite` 쿠키로 전달한다.
- 토큰을 요청 본문, URL 쿼리 또는 브라우저 저장소에 전달하지 않는다.
- 상태를 변경하는 요청은 `X-CSRF-Token` 헤더를 요구한다.
- 서버는 역할을 요청 본문에서 받지 않고 인증 세션과 병동 소속에서 판정한다.
- 소셜 로그인 콜백(`GET /auth/oauth/{provider}/callback`)은 제공자가 호출하므로 CSRF 헤더 대신 `state` 값으로 검증한다. 소셜 가입 완료(`POST /auth/oauth/signup`)는 회원가입·로그인과 같이 CSRF 헤더 없이 허용하고 가입 티켓 쿠키로 검증한다.

### 1.2 병동 범위 접근 통제

- 병동 리소스 경로는 `/wards/me`를 사용한다. 클라이언트가 임의의 병동 ID를 선택하지 못하게 한다.
- 서버는 요청자의 병동과 대상 리소스의 병동이 일치하는지 모든 요청에서 검사한다.
- 다른 병동의 리소스이거나 조회 권한이 없으면 존재 여부를 감추기 위해 `404`를 반환한다.
- 일반 간호사 응답에서는 타인의 숙련도, 경력, 건강 상태, 신청 사유, 개인 통계와 초안 데이터를 제외한다.

### 1.3 요청과 응답 형식

성공 응답은 단일 리소스일 때 `{ "data": {...} }`, 목록일 때 `{ "data": [...], "meta": PageMeta }` 형식을 사용한다. `204` 응답에는 본문이 없다. 날짜는 `YYYY-MM-DD`, 연월은 `YYYY-MM`, 시각은 UTC ISO 8601 형식으로 전달한다.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "요청 값을 확인해 주세요.",
    "traceId": "01J...",
    "fieldErrors": [{ "field": "email", "reason": "INVALID_FORMAT" }]
  }
}
```

### 1.4 페이지네이션과 정렬

- `page`는 0부터 시작하며 기본값은 0이다.
- `size` 기본값은 20, 최댓값은 100이다.
- `sort`는 `field,asc` 또는 `field,desc` 형식을 사용한다.
- 목록 API는 `PageMeta`를 반환한다.

### 1.5 멱등성과 동시성

- 생성, 승인, 확정, 자동 생성 시작 등 중복 실행 위험이 있는 `POST` 요청은 `Idempotency-Key` 헤더를 지원한다. 동일 키와 동일 요청은 동일 결과를 반환한다.
- 근무표 변경은 `baseVersion`을 사용한다. 서버 버전이 다르면 `409 VERSION_CONFLICT`를 반환한다.
- 편집 잠금 기능이 활성화된 경우 근무표 변경 요청은 `X-Schedule-Lock-Token`을 요구한다.
- 일괄 셀 변경은 전체 성공 또는 전체 실패로 처리한다.

### 1.6 소셜 로그인 (OAuth 2.0)

- 제공자는 `{provider}` 경로 값으로 구분한다. 허용값은 `google`, `kakao`이며 실제 제공 여부는 기능명세서 D-03에 따른다. 설정되지 않은 제공자는 `404 OAUTH_PROVIDER_NOT_SUPPORTED`를 반환한다.
- Authorization Code 방식에 PKCE(S256)를 함께 사용한다. 인가 코드 교환과 제공자 사용자 조회는 서버에서만 한다. 클라이언트 시크릿과 제공자 토큰은 프론트엔드로 보내지 않는다.
- 흐름은 브라우저 페이지 이동으로 진행한다. 프론트엔드는 `fetch`가 아니라 `window.location`으로 시작 API를 연다.
  1. 프론트엔드가 `GET /auth/oauth/{provider}/authorize`로 이동한다.
  2. 서버가 `state`와 PKCE `code_verifier`를 만들어 `OAUTH_STATE` 쿠키(HttpOnly, Secure, `SameSite=Lax`, Path=`/api/v1/auth/oauth`, 10분)에 담고 제공자 인증 화면으로 `302` 리다이렉트한다.
  3. 제공자가 `GET /auth/oauth/{provider}/callback?code=...&state=...`로 돌려보낸다. 서버는 `state`를 쿠키와 대조한 뒤 즉시 `OAUTH_STATE` 쿠키를 만료시킨다.
  4. 결과에 따라 프론트엔드 경로로 `302` 리다이렉트한다.

| 콜백 결과 | 서버 처리 | 리다이렉트 경로 |
| --- | --- | --- |
| 연결된 계정 있음 | 인증 쿠키 발급 (1.1과 같음) | `redirectTo` 또는 `/` |
| 연결된 계정 없음 (LOGIN) | `OAUTH_SIGNUP_TICKET` 쿠키 발급 (HttpOnly, Secure, `SameSite=Lax`, Path=`/api/v1/auth/oauth`, 10분) | `/signup/social` |
| 계정 연결 성공 (LINK) | 현재 계정에 소셜 계정 연결 | `redirectTo` 또는 `/settings/account` |
| 실패 | 쿠키를 발급하지 않음 | `/login?error={오류 코드}` (LINK면 `/settings/account?error={오류 코드}`) |

- 리다이렉트 경로의 `error` 값은 오류 코드만 담고 토큰, 인가 코드, 이메일 같은 값은 담지 않는다.
- `redirectTo`는 `/`로 시작하는 서비스 내부 경로만 허용한다. `//` 또는 `http`로 시작하는 값은 무시하고 기본 경로를 사용한다(오픈 리다이렉트 방지).
- 인증 쿠키와 `OAUTH_STATE`, `OAUTH_SIGNUP_TICKET` 쿠키는 제공자에서 돌아오는 페이지 이동에 실려야 하므로 `SameSite=Lax`를 사용한다.
- 제공자 사용자 식별자(`sub`, 카카오 회원번호)로 계정을 찾는다. 이메일은 식별에 쓰지 않는다. 같은 이메일의 기존 계정이 있어도 자동으로 연결하지 않는다(기능명세서 D-18).

## 2 API 데이터베이스

| API ID | 영역 | API명 | 기능 ID | Method | Path | 권한 | 우선순위 | 결정 필요 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| API-AUTH-01 | 인증과 소속 | 회원가입 | AUTH-01 | POST | /auth/signup | 공개 | P1 | 아니요 |
| API-AUTH-02 | 인증과 소속 | 로그인 | AUTH-02 | POST | /auth/login | 공개 | P1 | 아니요 |
| API-AUTH-03 | 인증과 소속 | 세션 갱신 | AUTH-02 | POST | /auth/refresh | 리프레시 쿠키 | P1 | 예 |
| API-AUTH-04 | 인증과 소속 | 로그아웃 | AUTH-02 | POST | /auth/logout | 인증 사용자 | P1 | 아니요 |
| API-AUTH-05 | 인증과 소속 | 내 계정과 소속 조회 | AUTH-02 | GET | /me | 인증 사용자 | P1 | 아니요 |
| API-AUTH-06 | 인증과 소속 | 소셜 로그인 시작 | AUTH-08, AUTH-09 | GET | /auth/oauth/{provider}/authorize | 공개 (LINK는 인증 사용자) | P1 | 예 |
| API-AUTH-07 | 인증과 소속 | 소셜 로그인 콜백 | AUTH-08, AUTH-09 | GET | /auth/oauth/{provider}/callback | 공개 (state 쿠키) | P1 | 아니요 |
| API-AUTH-08 | 인증과 소속 | 소셜 가입 완료 | AUTH-08 | POST | /auth/oauth/signup | 가입 티켓 쿠키 | P1 | 예 |
| API-AUTH-09 | 인증과 소속 | 연결된 소셜 계정 목록 | AUTH-09 | GET | /me/social-accounts | 인증 사용자 | P2 | 아니요 |
| API-AUTH-10 | 인증과 소속 | 소셜 계정 연결 해제 | AUTH-09 | DELETE | /me/social-accounts/{provider} | 인증 사용자 | P2 | 아니요 |
| API-WARD-01 | 인증과 소속 | 병동 개설 | AUTH-03 | POST | /wards | 소속 없는 사용자 | P1 | 예 |
| API-WARD-02 | 인증과 소속 | 내 병동 조회 | AUTH-03 | GET | /wards/me | 병동 구성원 | P1 | 아니요 |
| API-WARD-03 | 인증과 소속 | 병동 가입 신청 | AUTH-04 | POST | /ward-membership-requests | 소속 없는 사용자 | P1 | 예 |
| API-WARD-04 | 인증과 소속 | 가입 신청 목록 | AUTH-06 | GET | /wards/me/membership-requests | HEAD_NURSE | P1 | 아니요 |
| API-WARD-05 | 인증과 소속 | 가입 승인 | AUTH-06 | POST | /wards/me/membership-requests/{requestId}/approve | HEAD_NURSE | P1 | 예 |
| API-WARD-06 | 인증과 소속 | 가입 반려 | AUTH-06 | POST | /wards/me/membership-requests/{requestId}/reject | HEAD_NURSE | P1 | 아니요 |
| API-WARD-07 | 인증과 소속 | 가입 코드 조회 | AUTH-05 | GET | /wards/me/join-code | HEAD_NURSE | P1 | 아니요 |
| API-WARD-08 | 인증과 소속 | 가입 코드 재발급 | AUTH-05 | POST | /wards/me/join-code/rotate | HEAD_NURSE | P1 | 아니요 |
| API-WARD-09 | 인증과 소속 | 수간호사 추가 부여 | AUTH-07 | POST | /wards/me/head-nurses/{nurseId}/grant | HEAD_NURSE | P1 | 아니요 |
| API-WARD-10 | 인증과 소속 | 수간호사 권한 이관 | AUTH-07 | POST | /wards/me/head-nurse-transfer | HEAD_NURSE | P1 | 아니요 |
| API-NUR-01 | 간호사 관리 | 간호사 등록 | NUR-01 | POST | /wards/me/nurses | HEAD_NURSE | P1 | 아니요 |
| API-NUR-02 | 간호사 관리 | 간호사 목록 | NUR-02 | GET | /wards/me/nurses | 병동 구성원 | P1 | 아니요 |
| API-NUR-03 | 간호사 관리 | 간호사 상세 | NUR-02 | GET | /wards/me/nurses/{nurseId} | 병동 구성원 | P1 | 아니요 |
| API-NUR-04 | 간호사 관리 | 간호사 수정 | NUR-03, NUR-04, NUR-05 | PATCH | /wards/me/nurses/{nurseId} | HEAD_NURSE | P1 | 아니요 |
| API-NUR-05 | 간호사 관리 | 퇴사 처리 | NUR-06 | POST | /wards/me/nurses/{nurseId}/retire | HEAD_NURSE | P1 | 아니요 |
| API-RULE-01 | 근무 규칙 | 규칙 목록 | RULE-01 | GET | /wards/me/rules | HEAD_NURSE | P1 | 아니요 |
| API-RULE-02 | 근무 규칙 | 규칙 수정 | RULE-01 | PATCH | /wards/me/rules/{ruleId} | HEAD_NURSE | P1 | 아니요 |
| API-RULE-03 | 근무 규칙 | 규칙 프리셋 적용 | RULE-04 | POST | /wards/me/rule-presets/{presetId}/apply | HEAD_NURSE | P1 | 아니요 |
| API-RULE-04 | 근무 규칙 | 월 공휴일 조회 | RULE-02 | GET | /wards/me/holidays | HEAD_NURSE | P1 | 아니요 |
| API-RULE-05 | 근무 규칙 | 공휴일 보정 | RULE-02 | PUT | /wards/me/holidays/{date} | HEAD_NURSE | P1 | 아니요 |
| API-RULE-06 | 근무 규칙 | 월 OFF 목표 조회 | RULE-03 | GET | /wards/me/off-targets/{yearMonth} | HEAD_NURSE | P1 | 예 |
| API-RULE-07 | 근무 규칙 | 월 OFF 목표 수정 | RULE-03 | PATCH | /wards/me/off-targets/{yearMonth} | HEAD_NURSE | P1 | 예 |
| API-SCH-01 | 근무표 | 월 근무표 생성 | SCH-01 | POST | /wards/me/schedules | HEAD_NURSE | P1 | 아니요 |
| API-SCH-02 | 근무표 | 연월별 근무표 조회 | SCH-08 | GET | /wards/me/schedules | 병동 구성원 | P1 | 아니요 |
| API-SCH-03 | 근무표 | 근무표 상세 | SCH-08 | GET | /wards/me/schedules/{scheduleId} | 병동 구성원 | P1 | 아니요 |
| API-SCH-04 | 근무표 | 셀 일괄 변경 | SCH-02, SCH-03 | PATCH | /wards/me/schedules/{scheduleId}/cells | HEAD_NURSE | P1 | 아니요 |
| API-SCH-05 | 근무표 | 커버리지 조회 | SCH-04 | GET | /wards/me/schedules/{scheduleId}/coverage | 병동 구성원 | P1 | 예 |
| API-SCH-06 | 근무표 | 규칙 위반 조회 | SCH-05 | GET | /wards/me/schedules/{scheduleId}/violations | HEAD_NURSE | P1 | 아니요 |
| API-SCH-07 | 근무표 | 근무표 확정 | SCH-07 | POST | /wards/me/schedules/{scheduleId}/confirm | HEAD_NURSE | P1 | 아니요 |
| API-SCH-08 | 근무표 | 근무표 확정 취소 | SCH-07 | POST | /wards/me/schedules/{scheduleId}/confirmation-cancellations | HEAD_NURSE | P1 | 예 |
| API-SCH-09 | 근무표 | 엑셀 내보내기 | SCH-09 | GET | /wards/me/schedules/{scheduleId}/export.xlsx | HEAD_NURSE | P1 | 아니요 |
| API-SCH-10 | 근무표 | 엑셀 가져오기 미리보기 | SCH-10 | POST | /wards/me/schedules/{scheduleId}/import-previews | HEAD_NURSE | P2 | 예 |
| API-SCH-11 | 근무표 | 가져오기 매핑 수정 | SCH-10 | PATCH | /wards/me/schedules/{scheduleId}/import-previews/{previewId} | HEAD_NURSE | P2 | 아니요 |
| API-SCH-12 | 근무표 | 엑셀 가져오기 반영 | SCH-10 | POST | /wards/me/schedules/{scheduleId}/import-previews/{previewId}/apply | HEAD_NURSE | P2 | 아니요 |
| API-SCH-13 | 근무표 | 편집 잠금 획득 | SCH-11 | POST | /wards/me/schedules/{scheduleId}/lock | HEAD_NURSE | P2 | 예 |
| API-SCH-14 | 근무표 | 편집 잠금 해제 | SCH-11 | DELETE | /wards/me/schedules/{scheduleId}/lock | 잠금 보유 HEAD_NURSE | P2 | 아니요 |
| API-SCH-15 | 근무표 | 편집 잠금 강제 인수 | SCH-11 | POST | /wards/me/schedules/{scheduleId}/lock/takeover | HEAD_NURSE | P2 | 예 |
| API-GEN-01 | 자동 생성 | 자동 생성 시작 | GEN-01 | POST | /wards/me/schedules/{scheduleId}/generations | HEAD_NURSE | P1 | 예 |
| API-GEN-02 | 자동 생성 | 자동 생성 상태 조회 | GEN-02 | GET | /wards/me/generations/{jobId} | HEAD_NURSE | P1 | 예 |
| API-GEN-03 | 자동 생성 | 자동 생성 중단 | GEN-02 | POST | /wards/me/generations/{jobId}/stop | HEAD_NURSE | P1 | 아니요 |
| API-GEN-04 | 자동 생성 | 완화안 적용 후 재실행 | GEN-04 | POST | /wards/me/generations/{jobId}/relaxations | HEAD_NURSE | P1 | 예 |
| API-GEN-05 | 자동 생성 | 부분 해 반영 | GEN-04 | POST | /wards/me/generations/{jobId}/partial-result/apply | HEAD_NURSE | P1 | 아니요 |
| API-REQ-01 | 신청 | 신청 등록 | REQ-01 | POST | /wards/me/requests | 병동 구성원 | P1 | 예 |
| API-REQ-02 | 신청 | 내 신청 목록 | REQ-04 | GET | /wards/me/requests/me | 병동 구성원 | P1 | 아니요 |
| API-REQ-03 | 신청 | 병동 신청 목록 | REQ-04 | GET | /wards/me/requests | HEAD_NURSE | P1 | 아니요 |
| API-REQ-04 | 신청 | 신청 상세 | REQ-04 | GET | /wards/me/requests/{requestId} | 신청자 또는 HEAD_NURSE | P1 | 아니요 |
| API-REQ-05 | 신청 | 신청 승인 | REQ-02 | POST | /wards/me/requests/{requestId}/approve | HEAD_NURSE | P1 | 아니요 |
| API-REQ-06 | 신청 | 신청 반려 | REQ-02 | POST | /wards/me/requests/{requestId}/reject | HEAD_NURSE | P1 | 아니요 |
| API-REQ-07 | 신청 | 신청 취소 | REQ-03 | POST | /wards/me/requests/{requestId}/cancel | 신청자 본인 | P1 | 아니요 |
| API-NOTI-01 | 알림 | 알림 목록 | NOTI-01 | GET | /me/notifications | 인증 사용자 | P2 | 예 |
| API-NOTI-02 | 알림 | 알림 읽음 처리 | NOTI-01 | PATCH | /me/notifications/{notificationId} | 알림 소유자 | P2 | 아니요 |
| API-NOTI-03 | 알림 | 알림 설정 조회 | NOTI-01 | GET | /me/notification-settings | 인증 사용자 | P2 | 아니요 |
| API-NOTI-04 | 알림 | 알림 설정 수정 | NOTI-01 | PATCH | /me/notification-settings | 인증 사용자 | P2 | 예 |
| API-SEC-01 | 보안과 감사 | 감사 로그 조회 | SEC-03 | GET | /wards/me/audit-logs | HEAD_NURSE | P1 | 아니요 |

## 3 엔드포인트 상세

### 인증과 소속

#### API-AUTH-01 회원가입

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-01 |
| 요청 | `POST /auth/signup` |
| 권한 | 공개 |
| 입력 | name, email, password, termsAgreed |
| 성공 응답 | 201 UserSummary |
| 주요 오류 | 400 VALIDATION_ERROR, 409 EMAIL_ALREADY_EXISTS |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-AUTH-02 로그인

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-02 |
| 요청 | `POST /auth/login` |
| 권한 | 공개 |
| 입력 | email, password |
| 성공 응답 | 200 UserSummary + 인증 쿠키 |
| 주요 오류 | 401 INVALID_CREDENTIALS, 423 ACCOUNT_DISABLED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-AUTH-03 세션 갱신

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-02 |
| 요청 | `POST /auth/refresh` |
| 권한 | 리프레시 쿠키 |
| 입력 | 본문 없음 |
| 성공 응답 | 200 새 인증 쿠키 |
| 주요 오류 | 401 REFRESH_TOKEN_INVALID |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-AUTH-04 로그아웃

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-02 |
| 요청 | `POST /auth/logout` |
| 권한 | 인증 사용자 |
| 입력 | 본문 없음 |
| 성공 응답 | 204 + 인증 쿠키 만료 |
| 주요 오류 | 401 UNAUTHENTICATED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-AUTH-05 내 계정과 소속 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-02 |
| 요청 | `GET /me` |
| 권한 | 인증 사용자 |
| 입력 | 없음 |
| 성공 응답 | 200 MeResponse |
| 주요 오류 | 401 UNAUTHENTICATED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-AUTH-06 소셜 로그인 시작

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-08, AUTH-09 |
| 요청 | `GET /auth/oauth/{provider}/authorize?intent=LOGIN&redirectTo=/schedules` |
| 권한 | 공개. `intent=LINK`이면 인증 사용자 |
| 입력 | provider `google`\|`kakao`, intent `LOGIN`\|`LINK` (기본 LOGIN), redirectTo 서비스 내부 경로 (선택) |
| 성공 응답 | 302 제공자 인증 화면 + `OAUTH_STATE` 쿠키 |
| 주요 오류 | 404 OAUTH_PROVIDER_NOT_SUPPORTED, 401 UNAUTHENTICATED (LINK인데 세션 없음) |
| 우선순위 | P1 |
| 결정 필요 | 예. 제공자 목록(D-03) |

#### API-AUTH-07 소셜 로그인 콜백

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-08, AUTH-09 |
| 요청 | `GET /auth/oauth/{provider}/callback?code=...&state=...` |
| 권한 | 공개. `OAUTH_STATE` 쿠키 필요. 제공자가 호출하며 프론트엔드가 직접 호출하지 않는다 |
| 입력 | code, state. 사용자가 동의를 취소하면 제공자가 error를 보낸다 |
| 성공 응답 | 302 프론트엔드 경로 + 인증 쿠키 또는 `OAUTH_SIGNUP_TICKET` 쿠키 (1.6 표 참고) |
| 주요 오류 | 302 `/login?error=`로 전달: OAUTH_STATE_INVALID, OAUTH_ACCESS_DENIED, OAUTH_PROVIDER_ERROR, ACCOUNT_DISABLED, SOCIAL_ACCOUNT_ALREADY_LINKED, SOCIAL_PROVIDER_ALREADY_LINKED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-AUTH-08 소셜 가입 완료

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-08 |
| 요청 | `POST /auth/oauth/signup` |
| 권한 | `OAUTH_SIGNUP_TICKET` 쿠키 |
| 입력 | name, termsAgreed. 이메일과 비밀번호는 받지 않는다 |
| 성공 응답 | 201 UserSummary + 인증 쿠키. 가입 티켓 쿠키는 만료 |
| 주요 오류 | 400 VALIDATION_ERROR, 401 OAUTH_SIGNUP_TICKET_INVALID, 409 EMAIL_ALREADY_EXISTS (제공자 이메일이 기존 계정과 같음, D-18) |
| 우선순위 | P1 |
| 결정 필요 | 예. 이메일 충돌 처리(D-18), 이메일 미제공 처리(D-19) |

가입 화면에 보여줄 값이 필요하면 프론트엔드는 `GET /auth/oauth/signup`으로 `{ provider, email, suggestedName }`을 조회한다. 같은 가입 티켓 쿠키로 검증하며 티켓이 없거나 만료되면 401 OAUTH_SIGNUP_TICKET_INVALID를 반환한다.

#### API-AUTH-09 연결된 소셜 계정 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-09 |
| 요청 | `GET /me/social-accounts` |
| 권한 | 인증 사용자 |
| 입력 | 없음 |
| 성공 응답 | 200 SocialAccount 배열 |
| 주요 오류 | 401 UNAUTHENTICATED |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

소셜 계정 연결은 별도 API 없이 API-AUTH-06에 `intent=LINK`를 주어 시작한다.

#### API-AUTH-10 소셜 계정 연결 해제

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-09 |
| 요청 | `DELETE /me/social-accounts/{provider}` |
| 권한 | 인증 사용자 |
| 입력 | provider |
| 성공 응답 | 204 |
| 주요 오류 | 401 UNAUTHENTICATED, 404 RESOURCE_NOT_FOUND, 409 LAST_LOGIN_METHOD |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

#### API-WARD-01 병동 개설

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-03 |
| 요청 | `POST /wards` |
| 권한 | 소속 없는 사용자 |
| 입력 | hospitalName, wardName, requiredStaff, rulePreset |
| 성공 응답 | 201 Ward + Membership + joinCode |
| 주요 오류 | 409 MEMBERSHIP_ALREADY_EXISTS, 422 INVALID_STAFFING |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-WARD-02 내 병동 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-03 |
| 요청 | `GET /wards/me` |
| 권한 | 병동 구성원 |
| 입력 | 없음 |
| 성공 응답 | 200 Ward |
| 주요 오류 | 404 WARD_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-WARD-03 병동 가입 신청

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-04 |
| 요청 | `POST /ward-membership-requests` |
| 권한 | 소속 없는 사용자 |
| 입력 | joinCode |
| 성공 응답 | 201 MembershipRequest |
| 주요 오류 | 404 JOIN_CODE_NOT_FOUND, 409 REQUEST_ALREADY_EXISTS, 429 RATE_LIMITED |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-WARD-04 가입 신청 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-06 |
| 요청 | `GET /wards/me/membership-requests` |
| 권한 | HEAD_NURSE |
| 입력 | status, page, size |
| 성공 응답 | 200 MembershipRequest[] + PageMeta |
| 주요 오류 | 403 FORBIDDEN |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-WARD-05 가입 승인

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-06 |
| 요청 | `POST /wards/me/membership-requests/{requestId}/approve` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 Membership |
| 주요 오류 | 404 REQUEST_NOT_FOUND, 409 REQUEST_ALREADY_PROCESSED |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-WARD-06 가입 반려

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-06 |
| 요청 | `POST /wards/me/membership-requests/{requestId}/reject` |
| 권한 | HEAD_NURSE |
| 입력 | reason |
| 성공 응답 | 200 MembershipRequest |
| 주요 오류 | 400 REASON_REQUIRED, 409 REQUEST_ALREADY_PROCESSED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-WARD-07 가입 코드 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-05 |
| 요청 | `GET /wards/me/join-code` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 JoinCode |
| 주요 오류 | 403 FORBIDDEN |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-WARD-08 가입 코드 재발급

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-05 |
| 요청 | `POST /wards/me/join-code/rotate` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 JoinCode |
| 주요 오류 | 409 ROTATION_IN_PROGRESS |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-WARD-09 수간호사 추가 부여

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-07 |
| 요청 | `POST /wards/me/head-nurses/{nurseId}/grant` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 Membership |
| 주요 오류 | 404 NURSE_NOT_FOUND, 409 ROLE_ALREADY_ASSIGNED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-WARD-10 수간호사 권한 이관

| 항목 | 내용 |
| --- | --- |
| 기능 ID | AUTH-07 |
| 요청 | `POST /wards/me/head-nurse-transfer` |
| 권한 | HEAD_NURSE |
| 입력 | targetNurseId |
| 성공 응답 | 200 TransferResult |
| 주요 오류 | 404 NURSE_NOT_FOUND, 409 LAST_HEAD_NURSE_CONFLICT |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

### 간호사 관리

#### API-NUR-01 간호사 등록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NUR-01 |
| 요청 | `POST /wards/me/nurses` |
| 권한 | HEAD_NURSE |
| 입력 | NurseCreate |
| 성공 응답 | 201 Nurse |
| 주요 오류 | 400 VALIDATION_ERROR, 409 NURSE_ALREADY_EXISTS |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-NUR-02 간호사 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NUR-02 |
| 요청 | `GET /wards/me/nurses` |
| 권한 | 병동 구성원 |
| 입력 | q, role, dutyRole, status, includeRetired, sort, page, size |
| 성공 응답 | 200 Nurse[] + PageMeta, 역할별 필드 차등 |
| 주요 오류 | 400 INVALID_FILTER |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-NUR-03 간호사 상세

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NUR-02 |
| 요청 | `GET /wards/me/nurses/{nurseId}` |
| 권한 | 병동 구성원 |
| 입력 | 없음 |
| 성공 응답 | 200 Nurse, 역할별 필드 차등 |
| 주요 오류 | 404 NURSE_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-NUR-04 간호사 수정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NUR-03, NUR-04, NUR-05 |
| 요청 | `PATCH /wards/me/nurses/{nurseId}` |
| 권한 | HEAD_NURSE |
| 입력 | NursePatch, role 제외 |
| 성공 응답 | 200 Nurse + Violation[] |
| 주요 오류 | 400 VALIDATION_ERROR, 409 VERSION_CONFLICT |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-NUR-05 퇴사 처리

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NUR-06 |
| 요청 | `POST /wards/me/nurses/{nurseId}/retire` |
| 권한 | HEAD_NURSE |
| 입력 | affiliationEnd |
| 성공 응답 | 200 Nurse + Violation[] |
| 주요 오류 | 409 LAST_HEAD_NURSE, 422 INVALID_END_DATE |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

### 근무 규칙

#### API-RULE-01 규칙 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-01 |
| 요청 | `GET /wards/me/rules` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 Rule[] |
| 주요 오류 | 403 FORBIDDEN |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-RULE-02 규칙 수정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-01 |
| 요청 | `PATCH /wards/me/rules/{ruleId}` |
| 권한 | HEAD_NURSE |
| 입력 | enabled, severity, parameters, reason |
| 성공 응답 | 200 Rule + ViolationSummary[] |
| 주요 오류 | 400 INVALID_RULE_VALUE, 409 RULE_CONFLICT |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-RULE-03 규칙 프리셋 적용

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-04 |
| 요청 | `POST /wards/me/rule-presets/{presetId}/apply` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 Rule[] |
| 주요 오류 | 404 PRESET_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-RULE-04 월 공휴일 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-02 |
| 요청 | `GET /wards/me/holidays` |
| 권한 | HEAD_NURSE |
| 입력 | yearMonth |
| 성공 응답 | 200 Holiday[] |
| 주요 오류 | 400 INVALID_YEAR_MONTH |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-RULE-05 공휴일 보정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-02 |
| 요청 | `PUT /wards/me/holidays/{date}` |
| 권한 | HEAD_NURSE |
| 입력 | isHoliday, name, reason |
| 성공 응답 | 200 Holiday |
| 주요 오류 | 422 DATE_OUT_OF_SCOPE |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-RULE-06 월 OFF 목표 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-03 |
| 요청 | `GET /wards/me/off-targets/{yearMonth}` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 OffTarget |
| 주요 오류 | 404 TARGET_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-RULE-07 월 OFF 목표 수정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | RULE-03 |
| 요청 | `PATCH /wards/me/off-targets/{yearMonth}` |
| 권한 | HEAD_NURSE |
| 입력 | targetCount, reason |
| 성공 응답 | 200 OffTarget |
| 주요 오류 | 400 INVALID_TARGET, 409 SCHEDULE_CONFIRMED |
| 우선순위 | P1 |
| 결정 필요 | 예 |

### 근무표

#### API-SCH-01 월 근무표 생성

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-01 |
| 요청 | `POST /wards/me/schedules` |
| 권한 | HEAD_NURSE |
| 입력 | yearMonth |
| 성공 응답 | 201 Schedule |
| 주요 오류 | 409 SCHEDULE_ALREADY_EXISTS, 422 NO_ACTIVE_NURSES |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-02 연월별 근무표 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-08 |
| 요청 | `GET /wards/me/schedules` |
| 권한 | 병동 구성원 |
| 입력 | yearMonth |
| 성공 응답 | 200 Schedule, 역할과 상태에 따른 필드 차등 |
| 주요 오류 | 404 SCHEDULE_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-03 근무표 상세

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-08 |
| 요청 | `GET /wards/me/schedules/{scheduleId}` |
| 권한 | 병동 구성원 |
| 입력 | 없음 |
| 성공 응답 | 200 Schedule |
| 주요 오류 | 404 SCHEDULE_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-04 셀 일괄 변경

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-02, SCH-03 |
| 요청 | `PATCH /wards/me/schedules/{scheduleId}/cells` |
| 권한 | HEAD_NURSE |
| 입력 | changes[], baseVersion + X-Schedule-Lock-Token |
| 성공 응답 | 200 changedCells + version + Coverage + Violation[] |
| 주요 오류 | 409 VERSION_CONFLICT, 423 SCHEDULE_LOCKED, 422 PROTECTED_CELL |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-05 커버리지 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-04 |
| 요청 | `GET /wards/me/schedules/{scheduleId}/coverage` |
| 권한 | 병동 구성원 |
| 입력 | 없음 |
| 성공 응답 | 200 Coverage[] |
| 주요 오류 | 404 SCHEDULE_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-SCH-06 규칙 위반 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-05 |
| 요청 | `GET /wards/me/schedules/{scheduleId}/violations` |
| 권한 | HEAD_NURSE |
| 입력 | severity, nurseId, ruleId, page, size |
| 성공 응답 | 200 Violation[] + PageMeta |
| 주요 오류 | 404 SCHEDULE_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-07 근무표 확정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-07 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/confirm` |
| 권한 | HEAD_NURSE |
| 입력 | acknowledgedSoftViolationIds[] |
| 성공 응답 | 200 Schedule |
| 주요 오류 | 409 HARD_VIOLATIONS_EXIST, 409 INVALID_SCHEDULE_STATE |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-08 근무표 확정 취소

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-07 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/confirmation-cancellations` |
| 권한 | HEAD_NURSE |
| 입력 | reason |
| 성공 응답 | 200 Schedule |
| 주요 오류 | 400 REASON_REQUIRED, 409 ARCHIVED_SCHEDULE |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-SCH-09 엑셀 내보내기

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-09 |
| 요청 | `GET /wards/me/schedules/{scheduleId}/export.xlsx` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 application/vnd.openxmlformats-officedocument.spreadsheetml.sheet |
| 주요 오류 | 404 SCHEDULE_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-SCH-10 엑셀 가져오기 미리보기

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-10 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/import-previews` |
| 권한 | HEAD_NURSE |
| 입력 | multipart file |
| 성공 응답 | 202 ImportPreview |
| 주요 오류 | 413 FILE_TOO_LARGE, 422 INVALID_EXCEL_FORMAT |
| 우선순위 | P2 |
| 결정 필요 | 예 |

#### API-SCH-11 가져오기 매핑 수정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-10 |
| 요청 | `PATCH /wards/me/schedules/{scheduleId}/import-previews/{previewId}` |
| 권한 | HEAD_NURSE |
| 입력 | nurseMappings, dutyMappings |
| 성공 응답 | 200 ImportPreview |
| 주요 오류 | 404 PREVIEW_NOT_FOUND, 422 INVALID_MAPPING |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

#### API-SCH-12 엑셀 가져오기 반영

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-10 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/import-previews/{previewId}/apply` |
| 권한 | HEAD_NURSE |
| 입력 | baseVersion + X-Schedule-Lock-Token |
| 성공 응답 | 200 Schedule + Violation[] |
| 주요 오류 | 409 VERSION_CONFLICT, 422 UNRESOLVED_MAPPING |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

#### API-SCH-13 편집 잠금 획득

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-11 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/lock` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 201 ScheduleLock + 잠금 토큰 |
| 주요 오류 | 409 LOCK_ALREADY_HELD |
| 우선순위 | P2 |
| 결정 필요 | 예 |

#### API-SCH-14 편집 잠금 해제

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-11 |
| 요청 | `DELETE /wards/me/schedules/{scheduleId}/lock` |
| 권한 | 잠금 보유 HEAD_NURSE |
| 입력 | X-Schedule-Lock-Token |
| 성공 응답 | 204 |
| 주요 오류 | 403 LOCK_TOKEN_INVALID |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

#### API-SCH-15 편집 잠금 강제 인수

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SCH-11 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/lock/takeover` |
| 권한 | HEAD_NURSE |
| 입력 | reason |
| 성공 응답 | 200 ScheduleLock + 새 잠금 토큰 |
| 주요 오류 | 400 REASON_REQUIRED |
| 우선순위 | P2 |
| 결정 필요 | 예 |

### 자동 생성

#### API-GEN-01 자동 생성 시작

| 항목 | 내용 |
| --- | --- |
| 기능 ID | GEN-01 |
| 요청 | `POST /wards/me/schedules/{scheduleId}/generations` |
| 권한 | HEAD_NURSE |
| 입력 | fixedCells[], maxSeconds |
| 성공 응답 | 202 GenerationJob |
| 주요 오류 | 409 GENERATION_ALREADY_RUNNING, 422 PRECONDITION_FAILED |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-GEN-02 자동 생성 상태 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | GEN-02 |
| 요청 | `GET /wards/me/generations/{jobId}` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 GenerationJob |
| 주요 오류 | 404 JOB_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-GEN-03 자동 생성 중단

| 항목 | 내용 |
| --- | --- |
| 기능 ID | GEN-02 |
| 요청 | `POST /wards/me/generations/{jobId}/stop` |
| 권한 | HEAD_NURSE |
| 입력 | applyBestResult |
| 성공 응답 | 200 GenerationJob 또는 Schedule |
| 주요 오류 | 409 JOB_ALREADY_FINISHED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-GEN-04 완화안 적용 후 재실행

| 항목 | 내용 |
| --- | --- |
| 기능 ID | GEN-04 |
| 요청 | `POST /wards/me/generations/{jobId}/relaxations` |
| 권한 | HEAD_NURSE |
| 입력 | relaxationIds[] |
| 성공 응답 | 202 새 GenerationJob |
| 주요 오류 | 422 RELAXATION_NOT_ALLOWED |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-GEN-05 부분 해 반영

| 항목 | 내용 |
| --- | --- |
| 기능 ID | GEN-04 |
| 요청 | `POST /wards/me/generations/{jobId}/partial-result/apply` |
| 권한 | HEAD_NURSE |
| 입력 | baseVersion |
| 성공 응답 | 200 Schedule + Violation[] |
| 주요 오류 | 409 VERSION_CONFLICT, 404 PARTIAL_RESULT_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

### 신청

#### API-REQ-01 신청 등록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-01 |
| 요청 | `POST /wards/me/requests` |
| 권한 | 병동 구성원 |
| 입력 | type, targetDates, reasonCode, reasonDetail, preferredDuty |
| 성공 응답 | 201 WorkRequest |
| 주요 오류 | 409 DUPLICATE_REQUEST, 422 REQUEST_LIMIT_EXCEEDED |
| 우선순위 | P1 |
| 결정 필요 | 예 |

#### API-REQ-02 내 신청 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-04 |
| 요청 | `GET /wards/me/requests/me` |
| 권한 | 병동 구성원 |
| 입력 | yearMonth, type, status, page, size |
| 성공 응답 | 200 WorkRequest[] + PageMeta |
| 주요 오류 | 400 INVALID_FILTER |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-REQ-03 병동 신청 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-04 |
| 요청 | `GET /wards/me/requests` |
| 권한 | HEAD_NURSE |
| 입력 | applicantId, yearMonth, type, status, page, size |
| 성공 응답 | 200 WorkRequest[] + PageMeta |
| 주요 오류 | 403 FORBIDDEN |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-REQ-04 신청 상세

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-04 |
| 요청 | `GET /wards/me/requests/{requestId}` |
| 권한 | 신청자 또는 HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 WorkRequest |
| 주요 오류 | 404 REQUEST_NOT_FOUND |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-REQ-05 신청 승인

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-02 |
| 요청 | `POST /wards/me/requests/{requestId}/approve` |
| 권한 | HEAD_NURSE |
| 입력 | 없음 |
| 성공 응답 | 200 WorkRequest + scheduleImpact |
| 주요 오류 | 409 REQUEST_ALREADY_PROCESSED, 409 SCHEDULE_CONFIRMED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-REQ-06 신청 반려

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-02 |
| 요청 | `POST /wards/me/requests/{requestId}/reject` |
| 권한 | HEAD_NURSE |
| 입력 | reason |
| 성공 응답 | 200 WorkRequest |
| 주요 오류 | 400 REASON_REQUIRED, 409 REQUEST_ALREADY_PROCESSED |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

#### API-REQ-07 신청 취소

| 항목 | 내용 |
| --- | --- |
| 기능 ID | REQ-03 |
| 요청 | `POST /wards/me/requests/{requestId}/cancel` |
| 권한 | 신청자 본인 |
| 입력 | 없음 |
| 성공 응답 | 200 WorkRequest |
| 주요 오류 | 409 SCHEDULE_CONFIRMED, 409 REQUEST_NOT_CANCELLABLE |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

### 알림

#### API-NOTI-01 알림 목록

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NOTI-01 |
| 요청 | `GET /me/notifications` |
| 권한 | 인증 사용자 |
| 입력 | unreadOnly, type, page, size |
| 성공 응답 | 200 Notification[] + PageMeta |
| 주요 오류 | 400 INVALID_FILTER |
| 우선순위 | P2 |
| 결정 필요 | 예 |

#### API-NOTI-02 알림 읽음 처리

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NOTI-01 |
| 요청 | `PATCH /me/notifications/{notificationId}` |
| 권한 | 알림 소유자 |
| 입력 | read |
| 성공 응답 | 200 Notification |
| 주요 오류 | 404 NOTIFICATION_NOT_FOUND |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

#### API-NOTI-03 알림 설정 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NOTI-01 |
| 요청 | `GET /me/notification-settings` |
| 권한 | 인증 사용자 |
| 입력 | 없음 |
| 성공 응답 | 200 NotificationSettings |
| 주요 오류 | 401 UNAUTHENTICATED |
| 우선순위 | P2 |
| 결정 필요 | 아니요 |

#### API-NOTI-04 알림 설정 수정

| 항목 | 내용 |
| --- | --- |
| 기능 ID | NOTI-01 |
| 요청 | `PATCH /me/notification-settings` |
| 권한 | 인증 사용자 |
| 입력 | scheduleConfirmed, scheduleCancelled, requestResult, dutyReminder, webPush |
| 성공 응답 | 200 NotificationSettings |
| 주요 오류 | 422 PUSH_NOT_SUPPORTED |
| 우선순위 | P2 |
| 결정 필요 | 예 |

### 보안과 감사

#### API-SEC-01 감사 로그 조회

| 항목 | 내용 |
| --- | --- |
| 기능 ID | SEC-03 |
| 요청 | `GET /wards/me/audit-logs` |
| 권한 | HEAD_NURSE |
| 입력 | from, to, actorId, actionType, page, size |
| 성공 응답 | 200 AuditLog[] + PageMeta |
| 주요 오류 | 400 INVALID_DATE_RANGE, 403 FORBIDDEN |
| 우선순위 | P1 |
| 결정 필요 | 아니요 |

## 4 공통 데이터 모델

| 스키마 | 필드 |
| --- | --- |
| UserSummary | id UUID, name, email (소셜 가입이면 null 가능), accountStatus, hasPassword, socialProviders[] |
| SocialAccount | provider GOOGLE\|KAKAO, email 또는 null, linkedAt |
| MeResponse | user UserSummary, membership Membership 또는 null, ward Ward 또는 null |
| Membership | id, wardId, userId, role HEAD_NURSE\|NURSE, status, joinedAt |
| Ward | id, hospitalName, wardName, requiredStaff {D,E,N}, createdAt |
| Nurse | id, name, role, dutyRole, status, joinedAt, careerMonths, skillLevel, affiliationStart, affiliationEnd, preceptorOf, version |
| Rule | id, code, name, severity HARD\|SOFT, enabled, parameters, version |
| Schedule | id, yearMonth, status, version, nurses[], cells[], coverage[], statistics[], confirmedAt |
| ScheduleCell | nurseId, date, dutyCode D\|E\|N\|O\|AL\|ED\|null, editable |
| Violation | id, severity, ruleId, nurseId, date, currentValue, message |
| Coverage | date, dutyCode, actualCount, requiredCount, status UNDER\|MET\|OVER |
| GenerationJob | id, scheduleId, status, stage, elapsedSeconds, progress, hardViolationCount, metrics, conflicts, relaxations |
| WorkRequest | id, applicantId, type, targetDates, reasonCode, reasonDetail, preferredDuty, status, processorId, processedAt, rejectionReason |
| Notification | id, type, title, body, read, createdAt, resourceType, resourceId |
| AuditLog | id, occurredAt, actorId, actionType, targetType, targetId, before, after |
| PageMeta | page, size, totalElements, totalPages |
| ErrorResponse | timestamp, status, code, message, traceId, fieldErrors[] |

### 4.1 주요 요청 필드

| 요청 모델 | 필드 | 타입 | 필수 | 제약 |
| --- | --- | --- | --- | --- |
| SignupRequest | name | string | 예 | 1자부터 50자 |
| SignupRequest | email | string | 예 | 이메일 형식, 서비스 전체에서 고유 |
| SignupRequest | password | string | 예 | 8자 이상, 영문과 숫자 포함 |
| SignupRequest | termsAgreed | boolean | 예 | 반드시 true |
| SocialSignupRequest | name | string | 예 | 1자부터 50자 |
| SocialSignupRequest | termsAgreed | boolean | 예 | 반드시 true |
| NurseCreate | name | string | 예 | 1자부터 50자 |
| NurseCreate | dutyRole | enum | 예 | CHARGE, PRECEPTOR, NEW, GENERAL |
| NurseCreate | status | enum | 예 | ACTIVE, PREGNANT, ON_LEAVE, RETIRED |
| NurseCreate | joinedAt | date | 예 | YYYY-MM-DD |
| NurseCreate | careerMonths | integer | 예 | 0 이상 |
| NurseCreate | skillLevel | integer | 예 | 1부터 5 |
| NurseCreate | affiliationStart | date | 예 | YYYY-MM-DD |
| NurseCreate | affiliationEnd | date 또는 null | 아니요 | 시작일 이후 |
| NurseCreate | preceptorOf | UUID 배열 | 아니요 | 같은 병동의 NEW 간호사만 |
| NursePatch | 변경 필드 | 부분 객체 | 예 | role은 허용하지 않음 |
| ScheduleCellBulkPatch | baseVersion | integer | 예 | 현재 근무표 버전과 일치 |
| ScheduleCellBulkPatch | changes | 객체 배열 | 예 | 1건 이상, 원자적 반영 |
| ScheduleCellChange | nurseId | UUID | 예 | 같은 근무표 행 |
| ScheduleCellChange | date | date | 예 | 근무표 대상 월 |
| ScheduleCellChange | dutyCode | enum 또는 null | 예 | D, E, N, O, AL, ED, null. AL 직접 입력 금지 |
| WorkRequestCreate | type | enum | 예 | ANNUAL_LEAVE, PREFERRED_OFF, PREFERRED_SHIFT |
| WorkRequestCreate | targetDates | date 배열 | 예 | 중복 불가, 같은 대상 월 |
| WorkRequestCreate | reasonCode | string | 예 | 운영 코드 목록 값 |
| WorkRequestCreate | reasonDetail | string | 조건부 | ETC인 경우 필수 |
| WorkRequestCreate | preferredDuty | enum | 조건부 | PREFERRED_SHIFT인 경우 D, E, N 중 하나 |
| GenerationStart | fixedCells | 셀 참조 배열 | 아니요 | 해당 근무표의 셀만 가능 |
| GenerationStart | maxSeconds | integer | 아니요 | 서버 설정 상한 이내 |

### 4.2 근무표 셀 일괄 변경 예시

```json
{
  "baseVersion": 12,
  "changes": [
    { "nurseId": "4c42...", "date": "2026-10-12", "dutyCode": "N" },
    { "nurseId": "4c42...", "date": "2026-10-13", "dutyCode": "O" }
  ]
}
```

```json
{
  "data": {
    "version": 13,
    "changedCells": 2,
    "coverage": [],
    "violations": []
  }
}
```


## 5 열거형

| 구분 | 허용값 |
| --- | --- |
| Membership.role | `HEAD_NURSE`, `NURSE` |
| SocialAccount.provider | `GOOGLE`, `KAKAO` (경로 값은 소문자 `google`, `kakao`) |
| OAuth intent | `LOGIN`, `LINK` |
| Nurse.dutyRole | `CHARGE`, `PRECEPTOR`, `NEW`, `GENERAL` |
| Nurse.status | `ACTIVE`, `PREGNANT`, `ON_LEAVE`, `RETIRED` |
| Schedule.status | `DRAFT`, `GENERATING`, `CONFIRMED`, `ARCHIVED` |
| dutyCode | `D`, `E`, `N`, `O`, `AL`, `ED`, `null` |
| Rule.severity | `HARD`, `SOFT` |
| Request.type | `ANNUAL_LEAVE`, `PREFERRED_OFF`, `PREFERRED_SHIFT` |
| Request.status | `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED` |
| GenerationJob.status | `QUEUED`, `RUNNING`, `SUCCEEDED`, `STOPPED`, `NO_SOLUTION`, `FAILED` |

## 6 표준 오류 코드

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | 필드 형식이나 필수값 오류 |
| 400 | `OAUTH_STATE_INVALID` | 소셜 로그인 state가 없거나 만료되었거나 다름 |
| 400 | `OAUTH_ACCESS_DENIED` | 사용자가 제공자 동의 화면에서 취소함 |
| 401 | `UNAUTHENTICATED` | 세션 없음 또는 만료 |
| 401 | `OAUTH_SIGNUP_TICKET_INVALID` | 소셜 가입 티켓이 없거나 만료됨 |
| 403 | `FORBIDDEN` | 로그인했으나 역할 권한 부족 |
| 404 | `RESOURCE_NOT_FOUND` | 리소스 없음, 다른 병동 리소스, 조회 범위 밖 |
| 404 | `OAUTH_PROVIDER_NOT_SUPPORTED` | 지원하지 않거나 설정되지 않은 소셜 로그인 제공자 |
| 409 | `VERSION_CONFLICT` | 낙관적 잠금 버전 불일치 |
| 409 | `INVALID_RESOURCE_STATE` | 현재 상태에서 요청한 전이 불가 |
| 409 | `SOCIAL_ACCOUNT_ALREADY_LINKED` | 해당 소셜 계정이 이미 다른 계정에 연결됨 |
| 409 | `SOCIAL_PROVIDER_ALREADY_LINKED` | 현재 계정에 같은 제공자가 이미 연결됨 |
| 409 | `LAST_LOGIN_METHOD` | 비밀번호 없는 계정의 마지막 소셜 계정 해제 시도 |
| 413 | `FILE_TOO_LARGE` | 업로드 최대 크기 초과 |
| 422 | `BUSINESS_RULE_VIOLATION` | 입력 형식은 맞지만 업무 규칙 위반 |
| 423 | `SCHEDULE_LOCKED` | 다른 사용자가 편집 잠금 보유 |
| 429 | `RATE_LIMITED` | 로그인 또는 가입 코드 시도 제한 초과 |
| 500 | `INTERNAL_ERROR` | 예상하지 못한 서버 오류 |
| 502 | `OAUTH_PROVIDER_ERROR` | 제공자의 토큰 교환 또는 사용자 조회 실패 |

## 7 감사 로그 기록 대상

회원가입을 제외한 로그인과 로그아웃, 소셜 로그인과 소셜 가입, 소셜 계정 연결과 해제, 병동 개설, 가입 코드 발급, 가입 승인과 반려, 권한 이관, 간호사 등록과 변경과 퇴사, 규칙 변경, 근무표 생성과 셀 변경과 확정과 확정 취소, 자동 생성, 엑셀 입출력, 신청 승인과 반려, 편집 잠금 강제 인수를 기록한다. 비밀번호, 토큰, 인가 코드, state, 전체 업로드 파일과 민감한 자유 입력은 감사 로그에 저장하지 않는다.

## 8 API 수용 기준

- 기능명세서의 P1 기능이 하나 이상의 API와 연결되어야 한다.
- 일반 간호사는 API를 직접 호출해도 초안 근무표와 타인의 민감 필드를 받을 수 없어야 한다.
- 다른 병동의 식별자를 사용한 요청은 데이터 존재 여부를 노출하지 않아야 한다.
- 하드 위반이 있는 근무표의 확정 요청은 항상 실패해야 한다.
- 자동 생성 완료, 중단, 실패 후 근무표 상태와 잠금이 복구되어야 한다.
- 가입 승인, 권한 이관, 일괄 셀 변경과 확정은 부분 저장 없이 원자적으로 처리되어야 한다.
- 모든 오류 응답은 추적 가능한 `traceId`와 안정적인 오류 코드를 제공해야 한다.
- 소셜 로그인은 state 검증 없이 완료될 수 없고, 제공자 토큰과 인가 코드가 프론트엔드나 로그에 노출되지 않아야 한다.
