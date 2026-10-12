# API 명세서 v0.1 (초안)

> 노션 API 명세서 DB를 옮긴 초안입니다. [기능명세서 v0.2](./기능명세서_v0.2.md)의 **10. API 명세서 담당자를 위한 대조 요약**에서 지적된 문제(signup의 `role` 입력, Undo/Redo 서버 API, 토큰 body 전달 등)가 **아직 반영되지 않은 상태**입니다.


## 인증

| 기능 | Method | URL | 파라미터 | 사용자 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 로그인 | `POST` | `/api/auth/login` | email, password | 유저 | 사용자 로그인 및 인증 토큰 발급 |  |
| 로그아웃 | `POST` | `/api/auth/logout` | accessToken | 유저, 관리자 | 현재 로그인한 사용자의 JWT 인증을 종료하고 refreshToken을 폐기합니다. | JWT 기반 인증 |
| 내 정보 조회 | `GET` | `/api/auth/me` |  | 유저 | 현재 로그인한 사용자의 정보를 조회합니다. | Authorization 필요 |
| JWT 토큰 갱신 | `POST` | `/api/auth/refresh` | refreshToken | 유저, 관리자 | 만료된 액세스 토큰을 갱신하기 위해 refreshToken을 검증하고 새로운 JWT accessToken을 발급합니다. | JWT 기반 인증 |
| 회원가입 | `POST` | `/api/auth/signup` | email, password, name, role | 유저 | 간호사 또는 관리자 계정을 생성합니다. |  |
| 회원가입 | `POST` | `/api/auth/signup` | name, email, password, termsAgreed | 유저 | 신규 사용자가 계정을 생성하고 병동 소속이 없는 계정으로 등록합니다. |  |
| 소셜 로그인 | `POST` | `/api/auth/social-login` | provider, authorizationCode | 유저 | 카카오 또는 구글 외부 인증을 통해 가입·로그인합니다. 제공자는 확정 필요입니다. |  |
| 병동 코드 발급 | `POST` | `/api/wards/code` | wardName | 관리자 | 수간호사가 병동 가입용 6~8자 영숫자 코드를 발급합니다. |  |
| 병동 코드 재발급 | `POST` | `/api/wards/code/refresh` | wardId | 관리자 | 기존 병동 코드를 비활성화하고 새 코드를 발급합니다. |  |
| 가입 승인 목록 조회 | `GET` | `/api/wards/{wardId}/join-requests` | wardId | 관리자 | 병동 코드로 가입 신청한 승인 대기 사용자를 조회합니다. |  |
| 가입 승인 | `PATCH` | `/api/wards/{wardId}/join-requests/{requestId}` | wardId, requestId, action | 관리자 | 가입 신청을 승인하거나 반려합니다. |  |

## 유저

| 기능 | Method | URL | 파라미터 | 사용자 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 근무 교환 신청 | `POST` | `/api/exchanges` | scheduleId, targetMemberId | 유저 | 다른 간호사와 특정 근무를 교환 신청합니다. |  |
| 간호사 정보 수정 | `PATCH` | `/api/nurses/{memberId}` | memberId, name, role, status, joinedAt, skillLevel | 관리자 | 간호사 정보를 수정하고 역할·상태 변경 시 기존 배정을 재검증합니다. |  |
| 간호사 소속 기간 수정 | `PATCH` | `/api/nurses/{memberId}/affiliation` | memberId, startDate, endDate | 관리자 | 간호사의 소속 시작일과 종료일을 관리합니다. |  |
| 희망 근무 등록 | `POST` | `/api/requests` | date, preferredShift, reason | 유저 | 간호사가 특정 날짜의 희망 근무 또는 휴무를 등록합니다. |  |
| 연차 신청 | `POST` | `/api/requests/annual-leave` | startDate, endDate, reason | 유저 | 간호사가 단일일 또는 기간으로 연차를 신청합니다. 사유는 개인사정/경조/교육/기타 드롭다운 코드만 허용합니다. | 확정된 근무표 이후 신청 마감일 정책 확인 필요 |
| 희망 오프 신청 | `POST` | `/api/requests/off` | month, dates, reason | 유저 | 간호사가 희망 오프를 신청합니다. 승인 시 자동 생성의 소프트 제약으로 반영합니다. |  |
| 신청 승인·반려 | `PATCH` | `/api/requests/{requestId}/decision` | requestId, action, rejectionReason | 관리자 | 수간호사가 대기 중인 연차·희망 오프 신청을 승인하거나 반려합니다. | 연차 승인 시 하드 제약, 희망 오프 승인 시 소프트 제약 |
| 엑셀 가져오기 | `POST` | `/api/schedules/import` | file, wardId, year, month | 관리자 | xlsx 근무표를 파싱하고 이름·듀티 코드를 매핑한 뒤 규칙 검증을 실행합니다. |  |
| 근무표 수정 | `PATCH` | `/api/schedules/{scheduleId}` | scheduleId, memberId, date, shift | 관리자 | 생성된 근무표의 특정 근무를 수정합니다. |  |
| Undo/Redo | `POST` | `/api/schedules/{scheduleId}/history` | scheduleId, action | 관리자 | 현재 편집 세션의 셀 변경을 되돌리거나 다시 실행합니다. |  |
| 부분 고정 후 재생성 | `POST` | `/api/schedules/{scheduleId}/regenerate` | scheduleId, lockedCells | 관리자 | 고정한 셀을 하드 제약으로 유지한 상태에서 나머지 근무표를 재생성합니다. |  |
| 근무 알림 설정 | `PATCH` | `/api/users/me/notifications` | workSchedule, requestResult, reminder | 유저 | 근무표 확정, 신청 결과, 근무 전날 알림의 on/off를 설정합니다. |  |
| 병동 생성 | `POST` | `/api/wards` | name, hospitalName, requiredStaff | 관리자 | 관리자가 새로운 병동을 생성합니다. |  |
| 간호사 등록 | `POST` | `/api/wards/{wardId}/members` | wardId, name, position, careerYear, team, shiftType | 관리자 | 병동에 간호사를 등록합니다. |  |
| 병동 구성원 조회 | `GET` | `/api/wards/{wardId}/members` | wardId | 관리자, 유저 | 병동에 등록된 간호사 목록을 조회합니다. |  |
| 간호사 목록 조회 | `GET` | `/api/wards/{wardId}/nurses` | wardId, includeRetired, sort, role, status | 관리자, 유저 | 병동 소속 간호사 목록을 조회합니다. 일반 간호사는 이름·역할만 반환합니다. |  |
| 규칙 프리셋 적용 | `POST` | `/api/wards/{wardId}/rules/preset` | wardId, preset | 관리자 | 일반 병동 기본값 또는 최소 규칙 프리셋을 적용합니다. |  |
| 근무표 생성 | `POST` | `/api/wards/{wardId}/schedules` | wardId, year, month | 관리자 | 병동의 월별 근무표를 생성합니다. | 희망근무와 병동별 필요 인원을 반영 |
| 월 근무표 생성 | `POST` | `/api/wards/{wardId}/schedules/blank` | wardId, year, month | 관리자 | 대상 월의 빈 근무표를 생성하고 승인된 연차를 OFF로 선반영합니다. |  |

## 검색

| 기능 | Method | URL | 파라미터 | 사용자 | 설명 | 비고 |
|---|---|---|---|---|---|---|
| 근무 교환 추천 | `GET` | `/api/exchanges/recommendations` | scheduleId | 유저 | 현재 근무를 교환할 수 있는 간호사 후보를 추천합니다. |  |
| 간호사별 부담 분석 | `GET` | `/api/members/{memberId}/workload` | memberId, startDate, endDate | 관리자, 유저 | 개별 간호사의 근무량과 야간근무 편중을 분석합니다. |  |
| 근무 부담 분석 | `GET` | `/api/schedules/{scheduleId}/analysis` | scheduleId | 관리자, 유저 | 근무표를 기반으로 야간근무, 연속근무, 주말근무 등의 부담을 분석합니다. | 핵심 기능 |
| 커버리지 조회 | `GET` | `/api/schedules/{scheduleId}/coverage` | scheduleId | 관리자 | 날짜별 D/E/N 실제 배정 인원과 필요 인원 대비 상태를 조회합니다. |  |
| 엑셀 내보내기 | `GET` | `/api/schedules/{scheduleId}/export` | scheduleId | 관리자 | 근무표와 커버리지 및 개인 통계를 xlsx로 생성하고 감사 로그에 기록합니다. |  |
| 자동 생성 진행률 조회 | `GET` | `/api/schedules/{scheduleId}/generation/progress` | scheduleId | 관리자 | 자동 생성 단계, 경과 시간, 현재 최선 결과 지표를 조회합니다. |  |
| 근무표 개선안 조회 | `GET` | `/api/schedules/{scheduleId}/recommendations` | scheduleId | 관리자 | 검증 결과를 바탕으로 문제가 있는 근무의 개선 방향을 제공합니다. | 핵심 기능 |
| 개인별 통계 조회 | `GET` | `/api/schedules/{scheduleId}/statistics` | scheduleId | 관리자, 유저 | D/E/N/OFF, OFF 목표 대비, 주말·공휴일 근무 수를 조회합니다. |  |
| 근무표 검증 | `POST` | `/api/schedules/{scheduleId}/validate` | scheduleId | 관리자 | 근무표의 인력 부족, 연속 근무, 휴식 부족 등의 문제를 검사합니다. | 핵심 기능 |
| 감사 로그 조회 | `GET` | `/api/wards/{wardId}/audit-logs` | wardId, startDate, endDate, actorId, action | 관리자 | 본인 병동의 감사 로그를 기간·행위자·행위 종류로 조회합니다. |  |
| 대시보드 조회 | `GET` | `/api/wards/{wardId}/dashboard` | wardId, year, month | 관리자 | 병동의 근무표 상태와 부담 분석 결과를 한 번에 조회합니다. | 핵심 기능 |
| OFF 목표 계산 | `GET` | `/api/wards/{wardId}/off-target` | wardId, year, month | 관리자 | 공휴일 수를 기준으로 월별 1인당 OFF 목표를 계산합니다. |  |
| 근무표 조회 | `GET` | `/api/wards/{wardId}/schedules` | wardId, year, month | 관리자, 유저 | 병동의 월별 근무표를 조회합니다. |  |
| 병동 전체 근무표 조회 | `GET` | `/api/wards/{wardId}/schedules/confirmed` | wardId, year, month | 관리자, 유저 | 확정된 병동 전체 근무표를 조회하며 타인의 숙련도·사유·상태는 제거합니다. |  |
| 과거 근무표 조회 | `GET` | `/api/wards/{wardId}/schedules/history` | wardId, year, month | 관리자, 유저 | 확정된 이전 달 근무표를 당시 소속 인원 기준으로 조회합니다. |  |
