# 병동 근무표 서비스 API 명세서 v0.3 (엔드포인트 목록)

> 노션(명세서 > api2 > 엔드포인트 (2) DB)을 옮긴 문서입니다 (2026-10-20). [기능명세서 v0.3](기능명세서_v0.3.md)의 기능 ID와 함께 봐 주세요.
> 노션 DB는 `인증 / 유저 / 검색` 3개 카테고리로만 나뉘어 있어서, 여기서는 기능 영역별로 다시 묶었어요. 각 행의 비고는 노션 `기타` 칸을 그대로 옮긴 거예요.
> **사용** 칸의 `수간호사`는 노션의 `관리자`, `간호사`는 `유저`입니다.

## 공통

- 기본 경로는 `/api`입니다. [API명세서 v1.0](API명세서_v1.0.md)의 `/api/v1`, `/wards/me` 방식과 다르니 어느 쪽으로 맞출지 정해야 해요.
- 병동 범위 API는 경로에 `{wardId}`를 받습니다. 기능명세서 v0.3 SEC-03에 따라 서버는 요청자의 소속 병동과 `wardId`가 같은지 매번 확인하고, 다르면 `404`를 반환합니다.
- 역할(`role`)은 요청 본문으로 받지 않습니다. 서버가 세션에서 판정합니다 (AUTH-01, SEC-03 원칙 3).

## 1. 인증 · 계정

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 회원가입 | POST | `/api/auth/signup` | name, email, password, termsAgreed | 간호사 | 소속 없는 계정을 생성합니다. 권한 역할은 병동 개설, 가입 승인, 권한 이관으로만 정해집니다. | v1.1 (AUTH-01): role 입력 제거. 요청에 role이 있어도 무시 (D-24). 노션에 같은 URL 행이 2개 있어 하나로 합침 |
| 로그인 | POST | `/api/auth/login` | email, password | 간호사 | 사용자 로그인 및 인증 토큰 발급 |  |
| 소셜 로그인 | POST | `/api/auth/social-login` | provider, authorizationCode | 간호사 | 카카오 또는 구글 외부 인증을 통해 가입·로그인합니다. 제공자는 확정 필요입니다. |  |
| 로그아웃 | POST | `/api/auth/logout` | accessToken | 간호사, 수간호사 | 현재 로그인한 사용자의 JWT 인증을 종료하고 refreshToken을 폐기합니다. | JWT 기반 인증 |
| JWT 토큰 갱신 | POST | `/api/auth/refresh` | refreshToken | 간호사, 수간호사 | 만료된 액세스 토큰을 갱신하기 위해 refreshToken을 검증하고 새로운 accessToken을 발급합니다. | JWT 기반 인증 |
| 내 정보 조회 | GET | `/api/auth/me` |  | 간호사 | 현재 로그인한 사용자의 정보를 조회합니다. | Authorization 필요 |

## 2. 병동 · 가입

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 병동 생성 | POST | `/api/wards` | name, hospitalName, requiredStaff | 수간호사 | 관리자가 새로운 병동을 생성합니다. |  |
| 병동 코드 발급 | POST | `/api/wards/code` | wardName | 수간호사 | 수간호사가 병동 가입용 6~8자 영숫자 코드를 발급합니다. |  |
| 병동 코드 재발급 | POST | `/api/wards/code/refresh` | wardId | 수간호사 | 기존 병동 코드를 비활성화하고 새 코드를 발급합니다. |  |
| 가입 승인 목록 조회 | GET | `/api/wards/{wardId}/join-requests` | wardId | 수간호사 | 병동 코드로 가입 신청한 승인 대기 사용자를 조회합니다. |  |
| 가입 승인 | PATCH | `/api/wards/{wardId}/join-requests/{requestId}` | wardId, requestId, action, linkMemberId(선택), rejectionReason | 수간호사 | 가입 신청을 승인하거나 반려합니다. | v1.1 (AUTH-06): linkMemberId가 있으면 미리 등록된 간호사 기록에 계정 연결, 없으면 NURSE로 새 소속 생성. 요청의 role 값은 무시 (D-24) |
| 수간호사 권한 이관 | POST | `/api/wards/{wardId}/head-nurse-transfer` | wardId, targetMemberId, mode | 수간호사 | 다른 간호사에게 수간호사 권한을 추가 부여하거나 이관합니다. 이관 시 본인은 간호사로 강등됩니다. | v0.3 신규 (AUTH-07). mode: grant(추가 부여) / transfer(이관) |

## 3. 간호사

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 간호사 등록 | POST | `/api/wards/{wardId}/members` | wardId, name, nurseType, dutyRole, careerMonths, skillLevel, skillGroup, joinedAt, affiliationStart | 수간호사 | 병동에 간호사를 등록합니다. 간호사 유형(수간호사 / 간호사)을 하나만 선택하며, 역할과 권한을 따로 입력받지 않습니다. 듀티 속성(dutyRole)은 배정 속성으로 함께 입력합니다. | v0.3 수정 (NUR-01): 간호사 유형 1개로 통합. position·team·shiftType 제거 (COM-02) |
| 간호사 목록 조회 | GET | `/api/wards/{wardId}/nurses` | wardId, includeRetired, sort, role, status, skillGroup | 수간호사, 간호사 | 병동 소속 간호사 목록을 조회합니다. 일반 간호사는 이름·역할만 반환합니다. | v0.3 수정 (SCH-15): skillGroup으로 그룹 표시 가능 |
| 병동 구성원 조회 | GET | `/api/wards/{wardId}/members` | wardId | 수간호사, 간호사 | 병동에 등록된 간호사 목록을 조회합니다. |  |
| 간호사 정보 수정 | PATCH | `/api/nurses/{memberId}` | memberId, name, nurseType, dutyRole, status, joinedAt, skillLevel, skillGroup | 수간호사 | 간호사 정보를 수정하고 역할·상태 변경 시 기존 배정을 재검증합니다. | v0.3: 역할 변경은 nurseType으로 처리하며 권한은 서버가 결정 |
| 간호사 소속 기간 수정 | PATCH | `/api/nurses/{memberId}/affiliation` | memberId, startDate, endDate | 수간호사 | 간호사의 소속 시작일과 종료일을 관리합니다. |  |
| 간호사 퇴사 처리 | PATCH | `/api/nurses/{memberId}/retire` | memberId, endDate | 수간호사 | 간호사를 퇴사 처리합니다(soft delete). 레코드는 삭제하지 않습니다. | v0.3 신규 (NUR-09). 마지막 수간호사는 권한 이관 전 퇴사 불가 |
| 숙련 그룹 지정 | PATCH | `/api/nurses/{memberId}/skill-group` | memberId, skillGroup | 수간호사 | 간호사의 숙련 그룹을 지정합니다. 변경하면 배정을 재검증합니다. | v0.3 신규 (NUR-11). 수간호사만 지정 가능 |
| 프리셉터 지정 | PUT | `/api/nurses/{memberId}/preceptor` | memberId, preceptorId | 수간호사 | 신입 간호사에 프리셉터를 지정합니다. preceptorId를 비우면 해제합니다. | v0.3 신규 (NUR-12). 프리셉터 소프트 규칙으로 검증 |
| 간호사 비고 조회 | GET | `/api/nurses/{memberId}/notes` | memberId | 수간호사 | 간호사 비고를 조회합니다. 수간호사만 호출할 수 있습니다. | v0.3 신규 (NUR-13) |
| 간호사 비고 추가 | POST | `/api/nurses/{memberId}/notes` | memberId, content | 수간호사 | 간호사 비고를 누적으로 추가합니다. 기존 비고를 덮어쓰지 않으며 작성자와 작성 시각을 함께 기록합니다. | v1.1 (NUR-09): PUT 덮어쓰기에서 POST 누적으로 변경. 최대 길이 🔶 D-22 |
| 간호사 비고 삭제 | DELETE | `/api/nurses/{memberId}/notes/{noteId}` | memberId, noteId | 수간호사 | 비고 한 건을 삭제하고 감사 로그에 기록합니다. | v1.1 신규 (NUR-09) |
| 근무 히스토리 조회 | GET | `/api/nurses/{memberId}/history` | memberId, from, to, shift | 수간호사, 간호사 | 확정된 근무표 기준으로 간호사의 배정 이력을 조회합니다. 일반 간호사는 본인 것만 조회할 수 있습니다. | v0.3 신규 (NUR-13) |

## 4. 규칙 · 휴일 · OFF 목표

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 규칙 목록 조회 | GET | `/api/wards/{wardId}/rules` | wardId | 수간호사 | 병동의 법정 기본 규칙과 병원 커스텀 규칙을 활성 여부와 함께 조회합니다. | v0.3 신규 (RULE-06·RULE-07) |
| 규칙 추가 | POST | `/api/wards/{wardId}/rules` | wardId, type, target, value | 수간호사 | 병원 커스텀 규칙을 추가합니다. 유형(하드/소프트), 대상, 값을 입력합니다. | v0.3 신규 (RULE-06). 변경은 감사 로그에 기록 |
| 규칙 수정 | PATCH | `/api/wards/{wardId}/rules/{ruleId}` | wardId, ruleId, value | 수간호사 | 규칙의 값을 수정합니다. 법정 기본 규칙은 값만 수정할 수 있습니다. | v0.3 신규 (RULE-06). 다음 자동 생성부터 적용 |
| 규칙 켜기·끄기 | PATCH | `/api/wards/{wardId}/rules/{ruleId}/active` | wardId, ruleId, active | 수간호사 | 규칙을 삭제하지 않고 활성 여부만 변경합니다. | v0.3 신규 (RULE-06) |
| 규칙 삭제 | DELETE | `/api/wards/{wardId}/rules/{ruleId}` | wardId, ruleId | 수간호사 | 병원 커스텀 규칙을 삭제합니다. 법정 기본 규칙은 삭제할 수 없습니다. | v0.3 신규 (RULE-06). 법정 기본 규칙 삭제 요청은 거부 |
| 규칙 프리셋 적용 | POST | `/api/wards/{wardId}/rules/preset` | wardId, preset | 수간호사 | 일반 병동 기본값 또는 최소 규칙 프리셋을 적용합니다. |  |
| 법정 기본 규칙 목록 | GET | `/api/legal-rules` | - | 수간호사 | 법정 기본 규칙과 근거 조항을 조회합니다. | v1.1 신규 (RULE-06) |
| 법정 규칙을 병동 규칙으로 복사 | POST | `/api/wards/{wardId}/rules/from-legal/{legalRuleId}` | wardId, legalRuleId | 수간호사 | 선택한 법정 기본 규칙을 병동 커스텀 규칙(CUSTOM)으로 복사합니다. 원본은 변경하지 않습니다. | v1.1 신규 (RULE-06) |
| 공휴일 조회 | GET | `/api/holidays` | year, month | 수간호사, 간호사 | 법정 공휴일 목록을 조회합니다. 시스템이 제공하며 병동에서 수정할 수 없습니다. | v1.1 (RULE-03). 병동 추가 휴일은 `/api/wards/{wardId}/holidays/{year}` 사용 |
| 병동 휴일 조회 | GET | `/api/wards/{wardId}/holidays/{year}` | wardId, year | 수간호사, 간호사 | 법정 공휴일과 병동 추가 휴일을 합친 목록을 조회합니다. 항목마다 source(LEGAL 또는 WARD)를 포함합니다. | v1.1 신규 (RULE-03, SCH-12) |
| 병동 추가 휴일 저장 | PUT | `/api/wards/{wardId}/holidays/{year}` | wardId, year, dates | 수간호사 | 병동 추가 휴일 목록을 저장합니다. 다른 병동에는 영향이 없습니다. | v1.1 신규 (RULE-03). 법정 공휴일과 중복 날짜는 422 |
| OFF 목표 계산 | GET | `/api/wards/{wardId}/off-target` | wardId, year, month | 수간호사 | 공휴일 수를 기준으로 월별 1인당 OFF 목표를 계산합니다. |  |
| OFF 목표 설정 | PUT | `/api/wards/{wardId}/off-target` | wardId, year, month, targetOffDays | 수간호사 | 월별 간호사당 목표 OFF 일수를 설정합니다. 변경 시 해당 월을 재검증합니다. | v1.1 신규 (RULE-04). 월 일수를 넘으면 422 |

## 5. 근무표

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 월 근무표 생성 (빈 표) | POST | `/api/wards/{wardId}/schedules/blank` | wardId, year, month | 수간호사 | 대상 월의 빈 근무표를 생성하고 승인된 연차를 OFF로 선반영합니다. |  |
| 근무표 생성 | POST | `/api/wards/{wardId}/schedules` | wardId, year, month | 수간호사 | 병동의 월별 근무표를 생성합니다. | 희망근무와 병동별 필요 인원을 반영. v0.3 수정 (GEN-01): 활성 규칙을 DB(RULE 테이블)에서 읽어 적용 |
| 근무표 조회 | GET | `/api/wards/{wardId}/schedules` | wardId, year, month | 수간호사, 간호사 | 병동의 월별 근무표를 조회합니다. |  |
| 병동 전체 근무표 조회 | GET | `/api/wards/{wardId}/schedules/confirmed` | wardId, year, month | 수간호사, 간호사 | 확정된 병동 전체 근무표를 조회하며 타인의 숙련도·사유·상태는 제거합니다. |  |
| 과거 근무표 조회 | GET | `/api/wards/{wardId}/schedules/history` | wardId, year, month | 수간호사, 간호사 | 확정된 이전 달 근무표를 당시 소속 인원 기준으로 조회합니다. | v0.3 확인: 과거 근무표는 NUR-09 퇴사 처리(soft delete) 기록을 기준으로 당시 소속 인원을 표시 |
| 근무표 수정 | PATCH | `/api/schedules/{scheduleId}` | scheduleId, memberId, date, shift | 수간호사 | 자동 생성 결과를 포함해 DRAFT 근무표의 셀을 수동으로 편집합니다. 편집 직후 규칙을 재검증합니다. | v0.3 수정 (SCH-17): 자동 생성 후 수동 보정 기능. 확정 후에는 편집 불가 |
| Undo/Redo | POST | `/api/schedules/{scheduleId}/history` | scheduleId, action | 수간호사 | 현재 편집 세션의 셀 변경을 되돌리거나 다시 실행합니다. |  |
| 근무표 확정 | POST | `/api/schedules/{scheduleId}/confirm` | scheduleId | 수간호사 | 하드 위반이 0건일 때 근무표를 확정(CONFIRMED)하고 편집을 잠급니다. 소속 간호사에게 알림을 보냅니다. | v0.3 신규 (SCH-12). 위반이 있으면 차단하고 위반 목록을 반환 |
| 근무표 확정 취소 | POST | `/api/schedules/{scheduleId}/unconfirm` | scheduleId, reason | 수간호사 | 확정을 취소해 근무표를 DRAFT로 되돌립니다. 취소 사유는 필수입니다. | v0.3 신규 (SCH-12). 취소 즉시 일반 간호사 화면에서 제외 |
| 날짜별 차지 지정 | POST | `/api/schedules/{scheduleId}/charge-overrides` | scheduleId, date, memberId | 수간호사 | 특정 날짜의 차지 간호사를 지정합니다. 지정된 간호사는 해당 날짜에 D 배정이어야 합니다. | v0.3 신규 (SCH-16) |
| 날짜별 차지 해제 | DELETE | `/api/schedules/{scheduleId}/charge-overrides/{date}` | scheduleId, date | 수간호사 | 지정된 날짜의 차지 지정을 해제합니다. | v0.3 신규 (SCH-16) |
| 엑셀 내보내기 | GET | `/api/schedules/{scheduleId}/export` | scheduleId | 수간호사 | 근무표와 커버리지 및 개인 통계를 xlsx로 생성하고 감사 로그에 기록합니다. |  |
| 엑셀 가져오기 | POST | `/api/schedules/import` | file, wardId, year, month | 수간호사 | xlsx 근무표를 파싱하고 이름·듀티 코드를 매핑한 뒤 규칙 검증을 실행합니다. |  |

## 6. 자동 생성

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 자동 생성 실행 | POST | `/api/schedules/{scheduleId}/generate` | scheduleId | 수간호사 | DRAFT 근무표에 규칙·승인된 연차·희망 오프를 반영해 자동 생성을 시작합니다. 생성 중에는 상태가 GENERATING으로 잠깁니다. | v0.3 신규 (GEN-01). 제한 시간 🔶 60초 권장 |
| 자동 생성 진행률 조회 | GET | `/api/schedules/{scheduleId}/generation/progress` | scheduleId | 수간호사 | 자동 생성 단계, 경과 시간, 현재 최선 결과 지표를 조회합니다. |  |
| 자동 생성 중단 | POST | `/api/schedules/{scheduleId}/generate/stop` | scheduleId | 수간호사 | 진행 중인 자동 생성을 중단하고, 그 시점의 최선 해를 DRAFT 근무표에 반영합니다. | v0.3 신규 (GEN-01·GEN-02 중단 버튼) |
| 부분 고정 후 재생성 | POST | `/api/schedules/{scheduleId}/regenerate` | scheduleId, lockedCells | 수간호사 | 고정한 셀을 하드 제약으로 유지한 상태에서 나머지 근무표를 재생성합니다. |  |

## 7. 검증 · 통계 · 분석

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 근무표 검증 | POST | `/api/schedules/{scheduleId}/validate` | scheduleId | 수간호사 | 근무표의 인력 부족, 연속 근무, 휴식 부족 등의 문제를 검사합니다. | 핵심 기능 |
| 커버리지 조회 | GET | `/api/schedules/{scheduleId}/coverage` | scheduleId | 수간호사 | 날짜별 D/E/N 실제 배정 인원과 필요 인원 대비 상태를 조회합니다. |  |
| 개인별 통계 조회 | GET | `/api/schedules/{scheduleId}/statistics` | scheduleId | 수간호사, 간호사 | D/E/N/OFF, OFF 목표 대비, 주말·공휴일 근무 수를 조회합니다. |  |
| 근무 부담 분석 | GET | `/api/schedules/{scheduleId}/analysis` | scheduleId | 수간호사, 간호사 | 근무표를 기반으로 야간근무, 연속근무, 주말근무 등의 부담을 분석합니다. | 핵심 기능 |
| 간호사별 부담 분석 | GET | `/api/members/{memberId}/workload` | memberId, startDate, endDate | 수간호사, 간호사 | 개별 간호사의 근무량과 야간근무 편중을 분석합니다. |  |
| 근무표 개선안 조회 | GET | `/api/schedules/{scheduleId}/recommendations` | scheduleId | 수간호사 | 검증 결과를 바탕으로 문제가 있는 근무의 개선 방향을 제공합니다. | 핵심 기능 |
| 대시보드 조회 | GET | `/api/wards/{wardId}/dashboard` | wardId, year, month | 수간호사 | 병동의 근무표 상태와 부담 분석 결과를 한 번에 조회합니다. | 핵심 기능 |

## 8. 신청

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 연차 신청 | POST | `/api/requests/annual-leave` | startDate, endDate, reason | 간호사 | 간호사가 단일일 또는 기간으로 연차를 신청합니다. 사유는 개인사정/경조/교육/기타 드롭다운 코드만 허용합니다. | 확정된 근무표 이후 신청 마감일 정책 확인 필요 |
| 희망 오프 신청 | POST | `/api/requests/off` | month, dates, reason | 간호사 | 간호사가 희망 오프를 신청합니다. 승인 절차 없이 접수 즉시 자동 생성의 소프트 제약으로 반영합니다. 희망 오프 마감일 이후에는 차단합니다. | v0.3 수정 (REQ-02): 승인 제거. 마감일은 REQ-08 기준 |
| 희망 근무 등록 | POST | `/api/requests` | date, preferredShift, reason | 간호사 | 간호사가 특정 날짜의 희망 근무 또는 휴무를 등록합니다. |  |
| 연차 신청 승인·반려 | PATCH | `/api/requests/{requestId}/decision` | requestId, action, rejectionReason | 수간호사 | 수간호사가 대기 중인 연차 신청을 승인하거나 반려합니다. 희망 오프와 희망 근무는 승인 대상이 아닙니다. | v1.1 (REQ-02): 희망 오프·희망 근무 대상이면 422 REQUEST_NOT_APPROVABLE. 연차 승인 시 AL 자동 입력과 재검증 |
| 신청 목록 조회 (본인) | GET | `/api/requests/me` | year, month, type, status | 간호사 | 본인의 연차·희망 오프 신청 내역을 조회합니다. | v0.3 신규 (REQ-07) |
| 신청 목록 조회 (관리) | GET | `/api/wards/{wardId}/requests` | wardId, status, memberId, year, month, type | 수간호사 | 병동의 신청 목록을 조회합니다. 희망 오프는 승인 상태가 없으므로 목록에서 조회만 합니다. | v0.3 신규 (REQ-07) |
| 희망 오프 마감일 설정 | PUT | `/api/wards/{wardId}/request-deadlines/{year}/{month}` | wardId, year, month, deadline | 수간호사 | 월별 희망 오프 신청 마감일을 설정합니다. 마감 이후에는 신청을 차단합니다. | v0.3 신규 (REQ-08). 미설정 시 기본값 🔶 확정 필요 |
| 마감 후 대리 신청 등록 | POST | `/api/wards/{wardId}/requests/on-behalf` | wardId, memberId, type, targetDates, reasonCode | 수간호사 | 희망 오프 마감 이후 수간호사가 간호사 대신 신청을 등록합니다. 감사 로그에 기록합니다. | v1.1 신규 (REQ-05, D-25) |

## 9. 근무 교환

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 근무 교환 추천 | GET | `/api/exchanges/recommendations` | scheduleId | 간호사 | 현재 근무를 교환할 수 있는 간호사 후보를 추천합니다. |  |
| 근무 교환 신청 | POST | `/api/exchanges` | scheduleId, targetMemberId | 간호사 | 다른 간호사와 특정 근무를 교환 신청합니다. |  |

## 10. 알림 · 감사 로그

| 기능 | Method | URL | 파라미터 | 사용 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 근무 알림 설정 | PATCH | `/api/users/me/notifications` | workSchedule, requestResult, reminder | 간호사 | 근무표 확정, 신청 결과, 근무 전날 알림의 on/off를 설정합니다. |  |
| 감사 로그 조회 | GET | `/api/wards/{wardId}/audit-logs` | wardId, startDate, endDate, actorId, action | 수간호사 | 본인 병동의 감사 로그를 기간·행위자·행위 종류로 조회합니다. |  |

## 11. 정리가 필요한 부분

노션 DB를 옮기면서 [기능명세서 v0.3](기능명세서_v0.3.md)과 맞지 않거나 비어 있는 곳을 모았어요. 노션 DB는 고치지 않았습니다.

### 11-1. 노션 템플릿에서 남은 행

노션 API 템플릿의 예시 행으로 보여서 위 표에서 뺐어요. 확인 후 노션에서 지우면 됩니다. 기능명세서 v0.3 10절에서도 "출결 검색 잔재"를 정리 대상으로 적어 두었어요.

| 기능 | Method | URL | 내용 |
|---|---|---|---|
| 로그인 하기 | POST | `/auth/login` | `/api/auth/login`과 중복 |
| 모든 유저 목록 | GET | `/users` | 전역 사용자 목록. SEC-03 원칙 5 위반 |
| 유저 조회 | GET | `/users/{user_id}` | 설명이 "학생을 조회"로 되어 있음 |
| 출결 검색 | GET | `/search` | 출결 기능은 이 서비스에 없음 |
| new Endpoint | - | - | 내용 없는 빈 행 |

### 11-2. 기능명세서 v0.3과 다른 부분

| API | 내용 | 관련 기능 |
|---|---|---|
| 로그아웃, JWT 토큰 갱신 | 파라미터로 accessToken·refreshToken을 받음. 기능명세서는 토큰을 쿠키로만 주고받고 본문·쿼리로 받지 않음 | AUTH-05 |
| Undo/Redo | 서버 API로 정의됨. 기능명세서는 클라이언트 메모리 스택으로 처리하고 서버 API로 만들지 않는다고 명시 | SCH-07 |
| 병동 코드 발급 | 파라미터가 wardName. 기능명세서는 병동이 이미 있으므로 병동명을 입력받지 않음 | AUTH-04 |
| 간호사별 부담 분석 | 타인 memberId로 조회 가능. 본인 리소스는 `me`를 쓰도록 되어 있음 | SEC-03 원칙 6, COM-01 |
| 월 근무표 생성 (빈 표) | 승인된 연차를 OFF로 선반영한다고 되어 있음. 기능명세서는 `AL`로 선반영 | SCH-01, COM-03 |
| 엑셀 가져오기 | 단일 요청으로 정의됨. 기능명세서는 미리보기 → 확정 2단계 | SCH-11 |
| 근무 교환 신청 | 대상 날짜·듀티 없음. 기능명세서에 교환 기능 정의 자체가 없음 | 10절 |
| 간호사 목록 조회 / 병동 구성원 조회 | 같은 목록을 `/nurses`와 `/members` 두 경로로 조회. 기능명세서 10절의 "nurses/members 혼재" | NUR-03 |
| 근무표 생성 / 월 근무표 생성 (빈 표) | 생성 API가 2개. 자동 생성 실행(`/generate`)과의 역할 구분 필요 | SCH-01, GEN-01 |
| 경로 형식 | 간호사 API는 `/api/nurses/{memberId}`, 등록은 `/api/wards/{wardId}/members`로 섞여 있음 | - |

### 11-3. 기능은 있는데 API가 없는 것

| 기능 | 기능 ID |
|---|---|
| 병동 코드 입력 · 가입 신청 | AUTH-03 |
| 신청 본인 취소 (확정 전까지) | REQ-02 |
| 셀 일괄 편집 (범위·붙여넣기) | SCH-02~04 |
| 규칙 위반 목록 조회 (위반 표시 패널) | SCH-06 |
| 동시 편집 잠금 획득·해제·강제 해제 | SCH-13 |
| 생성 실패 시 충돌 원인·완화 제안 조회와 적용 | GEN-06 |
| 알림 목록 조회 | REQ-06 |

[← 명세서 목록](README.md)
