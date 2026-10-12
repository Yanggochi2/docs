> 작성: 김민서 · 2026-09-28

# 간호사 근무표(듀티표) 관리 서비스 — 개발 리서치

> 작성일: 2026-09-28 · 대상: 소프트웨어마이스터고 학교 프로젝트 / 대회 출품작<br>범위: ① 도메인 규칙 ② 경쟁 서비스 ③ 스케줄링 알고리즘 ④ 기술 스택·아키텍처

---

## 0. 요약 — 이 프로젝트를 한 문단으로

간호사 근무표는 겉보기엔 달력 앱이지만, 실제로는 **NP-hard 조합최적화 문제 + 매트릭스형 스프레드시트 UI + 역할 기반 승인 워크플로**가 합쳐진 물건입니다. 수간호사 한 명이 매달 20~40명 × 30일 = 600~1,200칸을 손으로 채우면서 "나이트 다음날 데이 금지", "연속 근무 5일 이하", "듀티별 최소 인원", "희망 오프 반영", "나이트 공평 배분"을 동시에 맞춰야 합니다.

**결론부터:**

- 자동 편성 엔진은 **Google OR-Tools CP-SAT (Python)** 를 쓰세요. 25명 × 30일 문제를 **약 10초에 최적해**로 풉니다(이 문서에서 직접 실행 검증함).
- 국내 경쟁 서비스는 이미 "자동 생성" 자체로는 차별화가 끝났습니다. **빈틈은 확정 이후의 운영** — 듀티 체인지(교환) 매칭, 공정성 설명(explainability), 결원 대응, 캘린더 구독입니다.
- UI의 승부처는 캘린더 라이브러리가 아니라 **직접 만든 매트릭스 그리드**입니다.

> ⚠️ 조사 환경 제약: WebSearch가 차단되어 Google 검색 + 개별 사이트 직접 조회로 교차 확인했습니다. `law.go.kr`, `korea.kr` 등 일부 1차 출처는 robots.txt로 자동 수집이 불가해 **검색 스니펫·2차 출처 기반**입니다. 법령 조문과 시범사업 지침 수치는 **제출 전 원문 대조를 권장**합니다.

---

# 1. 도메인 지식 — 간호사 근무 규칙

## 1.1 3교대(D/E/N) 기본 구조

| 근무 | 코드 | 표준 시간대 | 비고 |
|---|---|---|---|
| 데이 | **D** | 07:00 ~ 15:00 | 실제 출근은 06:30~06:40 (인수인계 선행) |
| 이브닝 | **E** | 15:00 ~ 23:00 |  |
| 나이트 | **N** | 23:00 ~ 익일 07:00 | 근로기준법상 야간(22~06시)과 구간이 **어긋남** |
| 오프 | **O / OFF** | — | 휴무 |

병원마다 D를 06:00~14:00 또는 08:00~16:00으로 쓰는 변형이 있고, 2교대(12시간) 병동도 존재합니다. **듀티 코드와 시간대는 하드코딩하지 말고 병동별 마스터 테이블로 설계**하세요.

**인수인계 문제**: 각 근무 20~30분 전 출근해 인계를 받는 것이 관행이라, 임상간호사 일평균 실근무가 **9.44시간**이라는 연구 결과가 있습니다(8시간 배정 대비 초과). 근무 간 최소 휴식 간격을 계산할 때 `8h + 인수인계 30분`으로 잡는 게 현실적입니다.

### 현장 듀티 코드 용어집

| 코드 | 의미 |
|---|---|
| D / E / N | 데이 / 이브닝 / 나이트 |
| O, OFF | 휴무 |
| **나오프 (N-Off)** | 나이트 종료 후 당일 휴식. 나이트 다음날 최소 1 OFF가 기본 |
| **나이트 킵** | 특정 인원을 일정 기간 나이트 고정 배치. 제도화된 형태가 **야간전담간호사** |
| 연차 / 반차 / 병가 / 경조 / 보건 | 각종 휴가 |
| Edu, 교육 | 교육 참석일(근무 인정) |
| CH | 듀티 체인지(교환) 표시 |

> **모델링**: 코드를 `근무형(D/E/N)` / `비근무형(OFF/연차/병가/교육)` 두 축으로 분리하세요. 비근무형은 인력 카운트에서 빠지지만 근로시간·연차 잔여에는 영향을 줍니다.

## 1.2 금기 근무 패턴 — 이게 핵심 제약입니다

| 속칭 | 패턴 | 문제 |
|---|---|---|
| **나데** | N → D | 실질 휴식 20시간 미만. 현장에서 "가장 피하는 듀티" |
| **이데** | E → D | 23시 퇴근 → 다음날 06:40 출근. 수면 4~5시간 |
| **나이** | N → E | 나이트 후 이브닝 |
| 더블 | 하루에 D+E 또는 E+N | 16시간 연속 |

**연속 근무 · 휴식 규칙**

- 연속 야간근무 **3일 이내**
- 연속 근무일 **5일 이내**
- **야간근무 2일 이상 연속 시 48시간 이상 휴식 보장** (복지부 시범사업 규칙)
- 월 야간근무 15일 이내 (실제로는 7~8일로 제한하는 병원이 대부분)
- 근무 간 최소 11시간 간격 (병원간호사회 야간근무 가이드라인 권고)

> **구현 팁**: 금기 전이를 `(전일 듀티 × 당일 듀티) 허용 행렬`로 정의하면 하드 제약이 아주 단순해집니다.

## 1.3 정책 · 법제도

### 보건복지부 「간호사 교대제 개선 시범사업」

이 서비스의 **규칙 프리셋을 그대로 가져올 수 있는 가장 좋은 출처**입니다.

**1차 (2022.4.30 ~ 2025.4.30)**

- 핵심 인력 배치 3종: **야간전담간호사 · 대체간호사 · 지원간호사**
- 계획 근무표를 **3개월 단위로 사전 작성**
- 야간전담간호사 연속 근무기간 **3개월 이하**(개인 동의 전제)
- 성과: 신규간호사 이직률 **15.7% → 10.6%**

**2차 (2025.9.1 ~)**

- 전국 **94개 의료기관** 참여
- 일반병동 근무표는 **2개월 단위**로 계획·운영 (1차의 3개월에서 변경)
- 대체간호사팀 운영체계 명문화
- 쟁점: 야간전담 10% 이상 배치 기준과 병동당 지원간호사 1명 기준이 **2차에서 삭제**되어 병원간호사회가 반발

> 정책의 핵심 KPI가 **"예측 가능성"** 입니다. 시스템에 **확정 공지일 필드 + 근무표 버전 관리 + 확정 후 변경 횟수 지표**를 넣으면 시범사업 참여 병원에 그대로 세일즈 포인트가 됩니다.

### 근로기준법 관련 조항

> ⚠️ law.go.kr robots 차단으로 원문 미확인. 제출 전 대조 필요.

| 조항 | 내용 | 듀티표 함의 |
|---|---|---|
| 제50조 | 1주 40시간, 1일 8시간 | 월 근무일수 상한으로 환산 |
| 제53조 | 연장근로 1주 12시간 한도 | 주 52시간 상한 |
| 제54조 | 8시간당 휴게 1시간 | 8h 듀티 = 실근무 7.5h + 휴게 0.5h |
| **제55조** | 1주 평균 1회 이상 유급 주휴일 | **7일 중 최소 1 OFF = 하드 제약** |
| **제56조** | **22:00~06:00** 야간근로 통상임금 **50% 이상 가산** | N 듀티(23~07시) 중 **22~06시와 겹치는 7시간**이 가산 대상 |
| 제60조 | 연차 15일(1년 80% 출근 기준) | 연차 계획을 근무표에 사전 반영해야 인력 계산이 맞음 |
| **제70조** | **임신 중 여성 야간·휴일근로 원칙 금지**, 산후 1년 미만은 동의+인가 필요 | **임신/산후 1년 미만 → N 배정 하드 금지** |

### 간호관리료 차등제 (간호등급)

- 병동의 간호인력 확보수준에 따라 **1~7등급**을 매겨 입원료를 가산/감산
- **2024년 1분기부터 산정 기준이 '병상 수' → '환자 수'로 전환**
- **야간전담간호사 관리료**: 차등제 등급이 상급종합 3등급 / 종합병원 4등급 / 병원 5등급 이상 + 야간전담간호사 2인 이상
- **야간간호료**(2019 신설): 야간(22~06시) 간호 제공. 야간근무 인원 산정은 8시간 근무 시 1인, 4~8시간 미만 시 0.5인

> **킬러 피처 아이디어**: 간호등급은 근무표의 *제약*이 아니라 *결과 지표*입니다. 근무표 확정 시 `병동별 일평균 간호사 수 / 평균 환자 수`를 집계해 **예상 간호등급을 실시간 표시**하면, 병원 입장에서는 수가 시뮬레이터가 됩니다.

### 간호법

- 「간호법 시행령」 **2025년 6월 21일 시행**
- **한계**: 간호법에는 **간호사 대 환자 비율(법적 배치기준)이 명시되지 않았습니다.** 현장에서 "교대제 개선의 핵심은 법적 인력 기준"이라는 요구가 계속되는 이유입니다.
- → **간호사:환자 비율은 병원이 설정하는 정책값(configurable)** 으로 두되, 시범사업/차등제 기준을 프리셋으로 제공하세요.

## 1.4 근무표 작성 실무

**절차**

1. 병동 듀티별 필요 인원 확정 (예: D 6명 / E 5명 / N 4명)
2. 고정 항목 선반영 (연차·경조·병가·교육·출산휴가, 야간전담/고정근무자)
3. 희망 OFF 접수 (마감일 지정) 및 반영
4. 금기 패턴 회피하며 로테이션 배치
5. 주말·공휴일·나이트 형평성 점검
6. 공지(확정) → 이후 듀티 체인지 처리

**희망 오프**: 월 1~2개 한도가 일반적. 경합 시 연차/경력 우선, 전월 미반영자 우선 등 암묵 규칙.<br>→ **소프트 제약 + 우선순위 가중치**로 모델링하되, **"연속 미반영 횟수"를 누적해 가중치를 자동 상승**시키면 체감 공정성이 크게 올라갑니다.

**듀티 체인지**: 확정 후 1:1 맞교환 + 수간호사 승인이 관행.<br>→ 교환은 "두 사람의 두 날짜 듀티 swap"이므로 **swap 후 전체 제약 재검증 API**가 필요합니다.

## 1.5 인력 구성 제약

- **프리셉터-프리셉티 동일 듀티 고정**: 신규 간호사는 교육기간(1~3개월) 동안 프리셉터와 같은 듀티 동반 배치
- **신규 나이트 투입 유예**: 독립 전 N 배정 금지, 첫 나이트는 시니어 동반
- **차지 간호사 상시 배치**: 모든 듀티(특히 N)에 차지 자격자 최소 1명 — 나이트는 인원이 적어 차지 부재 시 근무표 자체가 성립 안 됨
- **신규 동시 배치 상한**: 한 듀티에 신규 2명 이상 겹치지 않게
- **교육전담간호사**: D 고정(비교대)이 일반적

→ 간호사 엔티티에 `경력월수`, `역할(차지/프리셉터/프리셉티/야간전담/교육전담/일반)`, `보유 역량 태그[]`, `배치 가능 듀티[]`, `임신·육아단축 여부` 필드가 필요합니다.

## 1.6 제약조건 종합 — 설계에 바로 쓰는 형태

### 하드 제약 (위반 시 근무표 무효)

| # | 제약 | 근거 |
|---|---|---|
| H1 | 1인 1일 1듀티 | 관행 |
| H2 | N 직후 D 금지, E 직후 D 금지 | 현장 관행 |
| H3 | 연속 야간근무 ≤ 3일 | 시범사업·가이드라인 |
| H4 | 야간 2일 이상 연속 시 이후 48시간 이상 휴식 | 시범사업 지침 |
| H5 | 연속 근무일 ≤ 5일 | 현장 관행 |
| H6 | 주 평균 1회 이상 유급 휴일 | 근로기준법 §55 |
| H7 | 주 근로시간 ≤ 52시간 | 근로기준법 §50·§53 |
| H8 | 임신 중/산후 1년 미만 → N·휴일근무 금지 | 근로기준법 §70 |
| H9 | 듀티별 최소 인원 충족 | 병동 정책·간호등급 |
| H10 | 듀티별 차지 자격자 ≥ 1명 | 실무 |
| H11 | 프리셉티는 프리셉터와 동일 듀티 | 실무 |
| H12 | 확정 휴가는 근무 배정 불가 | — |
| H13 | 야간전담 연속 배정 ≤ 3개월 | 시범사업 지침 |

### 소프트 제약 (가중치 튜닝)

| # | 제약 |
|---|---|
| S1 | 월 야간근무 ≤ 15일 (권장 7~8일) |
| S2 | 희망 OFF 반영 (미반영 누적 시 가중치↑) |
| S3 | 순방향 로테이션(D→E→N) 선호 |
| S4 | N 블록 후 OFF 2일 이상 |
| S5 | 개인별 나이트/주말/공휴일 수 편차 최소화 |
| S6 | 단독 근무일(고립 D/E/N) 최소화 |
| S7 | 듀티별 평균 경력·스킬 커버리지 확보 |
| S8 | 야간전담 ≥ 10%, 병동당 지원간호사 1명 |
| S9 | 근무 간 간격 ≥ 11시간 |

---

# 2. 경쟁 서비스 벤치마킹

## 2.1 국내 — 간호 듀티 전용

| 서비스 | 타깃 | 자동 생성 | 가격 | 특징 |
|---|---|---|---|---|
| **하루듀티** (haruduty.com) | 수간호사 + 병동 | ● 30초 | **무료** | 나이트·주말 누적치 기반 공정 배분, 금지 패턴 차단, 모바일 캘린더 연동, 2026.8 앱 정식 출시 |
| **듀티메이트** (dutymate.net) | 수간호사 + 간호사 | ● 평균 10초 | 미공개 ("무료로 시작") | D·E·N·M·O, 병동별 규칙 커스터마이징, 모바일 신청 |
| **DutyMate** (dutymate.kr) | 병원 조직(SaaS) | ● 초안 생성 | **₩99,000~199,000/월** | 미충족 조건 플래그, **피로/위험 점검**, PWA, 권한 분리 |
| **듀티메이커** (dutymaker.com) | 병원 | ● | 미공개 | 나이트 배분 자동화, 숙련도 반영. **서울성모병원 정식 공급계약** |
| **듀티플래너** (dutyplanners.com) | 수간호사 + 기관 | ● 1분 | **₩49,900/월** (30일 무료) | **30개 이상 제약 규칙**, 엑셀 양방향, **통계 대시보드(공정성 점수·원티드 반영률)** |
| 개인 듀티앱군 (하이듀티, 듀팅, 쉬프트잇 등) | **개인 간호사** | ○ | 무료+광고 | 근무 달력, **근무별 알람**, 통계 리포트, 백업, 그룹 공유 |

일반 근태 SaaS인 **시프티(shiftee.io)**, **잡캔(jobcankr.com)** 은 스케줄 "편성" 도구일 뿐 자동 최적화 솔버가 없고, 병동 3교대 특화 기능도 없습니다.

## 2.2 해외

| 서비스 | 타깃 | 자동 생성 | 가격 |
|---|---|---|---|
| **NurseGrid** | 개인 간호사 (65만+ 사용자, 리뷰 11.7만개 4.9점) | X (개인 캘린더) | 무료 |
| **NurseGrid Manager** | 2~250명 팀 | X — 셀프 스케줄링 + 승인 중심 | **\$5/user/월** |
| **QGenda** | 병원 엔터프라이즈 | ● AI 예측 스케줄링 | 미공개 |
| **Shiftboard** (2025 UKG 인수) | 24/7 규제산업 | ● | 미공개 |
| **Smartlinx** | 시니어케어 | ● census·중증도 기반 자동 조정, **callout 관리** | 미공개 |
| **Deputy** | 일반 (39만+ 사업장) | ● AI 수요예측 | 유료 |
| **When I Work** | 일반 | ● 전 플랜 포함 | \$2.5 / \$5 / \$8 per user/월 |
| **Timefold** | 개발자 — 솔버 API | ● | 미공개 |

> Timefold는 경쟁 서비스가 아니라 **자동 생성 엔진 후보**로 보는 게 맞습니다(4장 참고).

## 2.3 MVP 필수 기능 (없으면 경쟁 탈락)

1. 병동/팀 생성 + 직원 초대 + **권한 분리**(관리자 / 간호사)
2. 근무 유형 정의 (D/E/N/O, 시간·강도)
3. 병동별 규칙 설정 — 최소 인원, 연속근무 상한, 금지 패턴, 최소 휴식
4. 간호사 요청 접수 (연차·오프·원티드) — 카톡/수기 대체
5. **자동 생성 + 관리자 수정·확정 워크플로** — 업계 표준은 "100% 자동 확정"이 아니라 **"초안" 프레이밍**
6. 확정 근무표 모바일 배포 + 알림
7. 개인 월간/오늘·다음 근무 조회
8. 엑셀 import/export

## 2.4 차별화 가능한 빈틈 (우선순위 순)

**① 듀티 체인지(간호사 간 교환) 워크플로 — 가장 큰 공백**<br>해외(NurseGrid Manager, QGenda, Smartlinx)는 shift swap / giveaway / open shift 픽업 + 관리자 승인이 핵심 기능입니다. 국내는 하루듀티·듀티메이트·듀티메이커 모두 "요청 제출"까지만 있고 **간호사끼리의 맞교환 매칭이 없습니다.** 실제 병동에서는 확정 후 체인지가 상시 발생하는데 전부 카톡·수기로 돌아갑니다.<br>→ 여기에 **"교환해도 규칙 위반이 안 되는 상대만 자동 추천"** 을 붙이면 국내 최초입니다.

**② 공정성 지표 시각화 + 설명 가능성(explainability)**<br>듀티플래너만 공정성 점수를 제공하고 나머지는 "공정하게 배분한다"고 주장만 합니다. 간호사가 납득하지 못하면 수간호사가 결국 수기로 고칩니다.<br>→ **"왜 내가 이번 달 나이트 6개인가"에 답하는 개인별 누적 대시보드**(나이트·주말·연휴 누적, 팀 내 백분위)

**③ 결원(call-out) 대응** — Smartlinx의 callout management(결원 시 자격 갖춘 대기 인력 자동 탐색 + 알림)에 해당하는 기능이 국내 전무

**④ 캘린더 구독** — iCal/Google Calendar 구독 URL(확정 시 자동 반영). 개인 간호사 앱군의 최대 니즈인데 생성 서비스들이 안 함

**⑤ 국내 규제 기반 규칙 프리셋** — 교대제 개선 시범사업 지침 + 야간간호료 산정 기준을 규칙 프리셋 + 위반 경고로 내장 → 병원 입장에서 **수가 리스크 관리 도구**가 됨

**⑥ 개인 앱 + 관리자 웹의 통합** — 국내는 둘이 완전히 분리되어 있음. NurseGrid가 해외에서 이 구조로 65만 사용자 확보(개인 앱으로 유저 확보 → Manager로 수익화)

**⑦ 가격 공백** — 국내는 무료 vs ₩49,900~199,000/월로 양극화. 해외 표준인 인당 과금 중간대가 비어 있음

**⑧ 다중 병동 / float pool** — 병동 간 인력 이동 가시성이 국내 전무

## 2.5 기능 비교표

| 기능 | 하루듀티 | 듀티메이트 | DutyMate.kr | 듀티메이커 | 듀티플래너 | 개인앱군 | NurseGrid Mgr | QGenda |
|---|---|---|---|---|---|---|---|---|
| 자동 생성 | ● 30초 | ● 10초 | ● 초안 | ● | ● 1분 | ○ | ○ | ● AI |
| 병동별 규칙 커스텀 | ● | ● | ● | ● | ● 30+ | ○ | ○ | ● |
| 위험/금지 패턴 차단 | ● | △ | ● | ● | ● | ○ | ○ | ● |
| 개인 요청 | ● | ● | ● | ● | ● | ○ | ● | ● |
| **듀티 체인지** | ○ | ○ | △ | ○ | ○ | ○ | **●** | ● |
| **셀프 스케줄링** | ○ | ○ | ○ | ○ | ○ | ○ | **●** | ● |
| **결원 대응** | ○ | ○ | ○ | ○ | ○ | ○ | ● | ● |
| **공정성 시각화** | △ | ○ | △ | ○ | **●** | △ | ○ | ● |
| 캘린더 연동 | ● | ○ | ○ | ○ | ○ | ● | ● | ● |
| 엑셀 import/export | △ | ○ | ○ | ○ | ● 양방향 | ○ | ○ | ● |
| 근무별 알람 | ○ | ○ | ○ | ○ | ○ | **●** | ○ | ○ |

● 지원 / △ 부분·불명확 / ○ 미지원

## 2.6 리뷰 기반 사용자 요구

개인 듀티 앱 사용자들이 반복적으로 요구하는 것:

- 깔끔한 근무 달력 (입력이 몇 초 안에 끝나야 함)
- **근무별 알람** (D/E/N마다 다른 기상 알람 자동 설정) — 개인 앱군의 1순위 기능
- 월별 나이트 수·오프 수 통계
- **안전한 백업** (기기 변경 시 데이터 유실이 큰 불만)
- 그룹 근무표 공유 (동료 듀티 조회 → 약속 잡기)

관리자 측 고통: "엑셀로 오래 고민", "수십 번 수정" — 듀티메이커·듀티플래너가 공통으로 지적하는 기존 워크플로

---

# 3. 스케줄링 알고리즘

## 3.1 Nurse Rostering Problem (NRP)

간호사 × 날짜 × 듀티의 3차원 할당 문제이며 **NP-hard**입니다. 25명 × 30일 × 4듀티만 해도 이론적 경우의 수가 4\^750 ≈ 10\^451 — 완전탐색은 불가능합니다.

**핵심 설계 원칙**: 하드 제약은 **모델의 제약식**으로, 소프트 제약은 **목적함수의 페널티 항**으로 표현합니다.

**표준 벤치마크**

- **Curtois & Qu 인스턴스** (http://www.schedulingbenchmarks.org/nrp/) — 24개 인스턴스, 최소 2주×8명 ~ 최대 52주×150명×32듀티. 대부분 최적해 증명됨
- **INRC-I (2010) / INRC-II (2014-15)** — 국제 간호사 근무표 경진대회. INRC-II는 주 단위 순차 편성이라 더 현실적

> 처음부터 벤치마크를 풀려 하지 마세요. 우리 병원 요구사항을 직접 정의하고 푸는 게 훨씬 현실적이고 발표에서도 설득력 있습니다. 벤치마크는 나중에 1~2개만 돌려 "표준 문제에도 적용 가능"을 보이는 검증용이면 충분합니다.

## 3.2 Google OR-Tools CP-SAT ★ 주력 추천

설치는 `pip install ortools` — 이게 전부입니다. 별도 솔버 설치, 라이선스, JVM 불필요.<br>저장소: https://github.com/google/or-tools (★14,112, Apache 2.0)

### 실무 제약을 CP-SAT로 표현하는 4가지 패턴

**(a) 슬라이딩 윈도우 — 가장 쉽고 실용적 (이것부터 익히세요)**

```python
# 연속 6일 창 안에 근무가 최대 5일 => 연속근무 <= 5
for d in range(NUM_DAYS - 5):
    model.add(sum(work[n, d+k] for k in range(6)) <= 5)

# 연속 나이트 <= 3
for d in range(NUM_DAYS - 3):
    model.add(sum(x[n, d+k, N] for k in range(4)) <= 3)
```

**(b) `add_forbidden_assignments` — 짧은 윈도우(2~3일)에만**

```python
# N(2) 다음날 D(0)/E(1) 금지
model.add_forbidden_assignments([duty[n][d], duty[n][d+1]], [(2, 0), (2, 1)])
```

5일 윈도우면 4\^5=1024개 튜플이라 관리 불가. 짧은 구간에만 쓰세요.

**(c) `add_automaton` — 정규 언어로 패턴 제약 (강력)**<br>"나이트는 반드시 2~3일 연속으로만 선다" 같이 튜플로는 표현이 끔찍한 제약을 상태 기계 몇 줄로 끝냅니다.

```python
# 상태: 0=N아님, 1=N 1일째, 2=N 2일째, 3=N 3일째
tr = []
for nonN in (D, E, O):
    tr += [(0, nonN, 0), (2, nonN, 0), (3, nonN, 0)]  # 상태1에 비N 전이 없음 => 단독 나이트 금지
tr += [(0, N, 1), (1, N, 2), (2, N, 3)]                # 상태3에 N 전이 없음 => 4연속 금지
model.add_automaton(duty, 0, [0, 2, 3], tr)
```

실행 검증 결과: `OPTIMAL`, 14일 패턴 `ONNOOOOOONNNOO` — 나이트가 정확히 2·3연속 덩어리로만 나타나고 단독 나이트 없음.

**(d) 소프트 제약 — 페널티 변수 + Minimize**

```python
miss = model.new_bool_var(f"missOff_n{n}_d{d}")
model.add(miss == 1).only_enforce_if(x[n, d, O].negated())
model.add(miss == 0).only_enforce_if(x[n, d, O])
obj_terms.append((miss, 4))   # 가중치 4

model.minimize(sum(var * coef for var, coef in obj_terms))
```

`only_enforce_if`(reification)가 핵심입니다. 이것만 이해하면 어떤 소프트 제약이든 만들 수 있습니다.

**(e) 공정성(편차 최소화)**

```python
# 방법 1 — max-min 스프레드 최소화 (추천)
model.add_min_equality(n_min, night_counts)
model.add_max_equality(n_max, night_counts)
model.add(spread == n_max - n_min)
obj_terms.append((spread, 10))
```

방법 1은 "가장 억울한 사람"을 줄이고(minimax), 평균편차합 최소화는 전체 평균 불공정을 줄입니다. **간호사 근무표는 방법 1이 체감 만족도가 높습니다** — 한 명만 나이트 8개 서는 상황이 제일 불만이 큽니다.

### 공식 예제 `shift_scheduling_sat.py` 해부

https://github.com/google/or-tools/blob/stable/examples/python/shift_scheduling_sat.py — **OR-Tools 예제 중 NRP에 가장 가까운 실전 코드**입니다. 꼭 읽어보세요.

핵심 헬퍼 3개:

- `negated_bounded_span(works, start, length)` — 구간을 부정하고 **경계 앞뒤를 붙여** 극대(maximal) 구간만 잡히게 함
- `add_soft_sequence_constraint(...)` — **연속 구간 길이**에 하드 한계 + 소프트 페널티 동시 적용 (예: "나이트는 2~3일 연속이 이상적, 1일이나 4일이면 페널티, 5일 이상 절대 금지")
- `add_soft_sum_constraint(...)` — **총 개수(합계)** 버전. `add_max_equality(excess, [delta, 0])` 이 "음수는 0으로 자르기" 관용구

제약을 **선언적 튜플 테이블**로 정의하는 게 이 예제의 진짜 교훈입니다:

```python
# (듀티, hard_min, soft_min, min_penalty, soft_max, hard_max, max_penalty)
shift_constraints = [
    (0, 1, 1, 0, 2, 2, 0),      # OFF: 최소 1일, 최대 2일
    (3, 1, 2, 20, 3, 4, 5),     # 나이트: 목표 2~3, 하드 상한 4
]
# (이전듀티, 다음듀티, 비용) — 비용 0이면 완전 금지
penalized_transitions = [
    (2, 3, 4),   # 오후 -> 나이트 비용 4
    (3, 1, 0),   # 나이트 -> 아침 금지  ← "N 다음날 D 금지"
]
```

전이 제약 구현의 관용구 — **금지 리터럴 리스트에 페널티 변수 하나를 append 하면 소프트가 된다**:

```python
transition = [~work[e, prev_shift, d], ~work[e, next_shift, d + 1]]
if cost == 0:
    model.add_bool_or(transition)          # 하드 금지
else:
    trans_var = model.new_bool_var(...)
    transition.append(trans_var)
    model.add_bool_or(transition)          # 소프트
    obj_bool_coeffs.append(cost)
```

### 성능 — 직접 측정한 벤치마크

`ortools 9.15.6755`, Python 3.11, 8 workers, 제한 60초:

| 규모 | 상태 | 소요 시간 | 목적값 | 변수 수 |
|---|---|---|---|---|
| 15명 × 28일 | **OPTIMAL** | 1.11초 | 4 | 1,782 |
| **25명 × 30일** | **OPTIMAL** | **6.9~10.4초** | 5 | 3,118 |
| 40명 × 30일 | **OPTIMAL** | 27.3초 | 10 | 4,933 |
| 60명 × 31일 | FEASIBLE (시간초과) | 60.0초 | 10 | 7,596 |
| 100명 × 31일 | **OPTIMAL** | 4.6초 | 4 | 12,596 |

**비직관적이지만 중요한 인사이트 3가지:**

1. **25명 × 30일은 10초 안에 최적해가 나옵니다.** 웹 앱에서 "생성" 버튼 누르고 기다릴 수 있는 수준.
2. **변수 수가 많다고 느려지지 않습니다.** 100명 케이스가 60명보다 **13배 빨랐습니다.** 인원이 여유로우면 제약이 헐거워져 해 공간이 넓어집니다. → **난이도는 규모가 아니라 "제약의 팽팽함(tightness)"이 결정합니다.** 발표에서 "N명까지 됩니다"보다 이 분석이 훨씬 수준 높습니다.
3. 최적해를 못 찾아도 `FEASIBLE`이면 **이미 모든 하드 제약을 만족하는 쓸 수 있는 근무표**입니다. `max_time_in_seconds`를 30~60초로 걸고 FEASIBLE도 수용하는 설계가 정답.

### 실행 검증된 예제 코드

아래 코드는 이 리서치 중 **실제로 실행해 동작을 확인**했습니다. (동봉 파일 `nurse_roster.py`)

```python
"""
간호사 근무표 자동 생성 (Google OR-Tools CP-SAT)
듀티: D(주간) / E(저녁) / N(야간) / O(off)

[하드] H1. 1일 1듀티  H2. 듀티별 일일 최소 인원
       H3. N 다음날 D/E 금지  H4. 연속 근무일<=5  H5. 연속 나이트<=3
[소프트] S1. 개인별 나이트 수 편차 최소화  S2. 희망 OFF 반영  S3. 과잉 인력 억제
"""
import time
from ortools.sat.python import cp_model

NUM_NURSES, NUM_DAYS = 25, 30
SHIFTS = ["D", "E", "N", "O"]
D, E, N, O = 0, 1, 2, 3
WORK_SHIFTS = [D, E, N]
MIN_STAFF = {D: 6, E: 5, N: 4}
MAX_CONSEC_WORK, MAX_CONSEC_NIGHT = 5, 3

REQUESTED_OFF  = [(n, (n * 7 + 3) % NUM_DAYS) for n in range(NUM_NURSES)]
REQUESTED_OFF += [(n, (n * 5 + 11) % NUM_DAYS) for n in range(0, NUM_NURSES, 2)]

W_OFF_REQUEST, W_NIGHT_FAIR, W_EXCESS = 4, 10, 1
all_nurses, all_days = range(NUM_NURSES), range(NUM_DAYS)

model = cp_model.CpModel()
x = {(n, d, s): model.new_bool_var(f"x_{n}_{d}_{s}")
     for n in all_nurses for d in all_days for s in range(len(SHIFTS))}
obj_terms = []

# H1: 1일 1듀티
for n in all_nurses:
    for d in all_days:
        model.add_exactly_one(x[n, d, s] for s in range(len(SHIFTS)))

# H2: 듀티별 최소 인원 (+ 과잉 페널티)
for d in all_days:
    for s, need in MIN_STAFF.items():
        staff = sum(x[n, d, s] for n in all_nurses)
        model.add(staff >= need)
        excess = model.new_int_var(0, NUM_NURSES, f"ex_{d}_{s}")
        model.add(excess == staff - need)
        obj_terms.append((excess, W_EXCESS))

# H3: N 다음날 D/E 금지
for n in all_nurses:
    for d in range(NUM_DAYS - 1):
        model.add_implication(x[n, d, N], x[n, d + 1, D].negated())
        model.add_implication(x[n, d, N], x[n, d + 1, E].negated())

# H4: 연속 근무일 <= 5 (슬라이딩 윈도우)
work = {(n, d): x[n, d, O].negated() for n in all_nurses for d in all_days}
for n in all_nurses:
    for d in range(NUM_DAYS - MAX_CONSEC_WORK):
        model.add(sum(work[n, d + k] for k in range(MAX_CONSEC_WORK + 1)) <= MAX_CONSEC_WORK)

# H5: 연속 나이트 <= 3
for n in all_nurses:
    for d in range(NUM_DAYS - MAX_CONSEC_NIGHT):
        model.add(sum(x[n, d + k, N] for k in range(MAX_CONSEC_NIGHT + 1)) <= MAX_CONSEC_NIGHT)

# S1: 나이트 수 편차 최소화
night_counts = []
for n in all_nurses:
    c = model.new_int_var(0, NUM_DAYS, f"nights_{n}")
    model.add(c == sum(x[n, d, N] for d in all_days))
    night_counts.append(c)
n_min = model.new_int_var(0, NUM_DAYS, "nmin")
n_max = model.new_int_var(0, NUM_DAYS, "nmax")
model.add_min_equality(n_min, night_counts)
model.add_max_equality(n_max, night_counts)
spread = model.new_int_var(0, NUM_DAYS, "spread")
model.add(spread == n_max - n_min)
obj_terms.append((spread, W_NIGHT_FAIR))

# S2: 희망 OFF 소프트 반영
for (n, d) in REQUESTED_OFF:
    miss = model.new_bool_var(f"miss_{n}_{d}")
    model.add(miss == 1).only_enforce_if(x[n, d, O].negated())
    model.add(miss == 0).only_enforce_if(x[n, d, O])
    obj_terms.append((miss, W_OFF_REQUEST))

model.minimize(sum(v * c for v, c in obj_terms))

solver = cp_model.CpSolver()
solver.parameters.max_time_in_seconds = 30.0
solver.parameters.num_workers = 8
t0 = time.time(); status = solver.solve(model); elapsed = time.time() - t0

print(f"상태: {solver.status_name(status)} / {elapsed:.2f}초 / 목적값 {solver.objective_value:.0f}")
for n in all_nurses:
    row = "".join(SHIFTS[s] if s != O else "."
                  for d in all_days for s in range(len(SHIFTS)) if solver.value(x[n, d, s]))
    print(f"N{n:02d} {row} (N={solver.value(night_counts[n])})")
```

**실행 결과**

```text
상태     : OPTIMAL
소요 시간: 10.40초  (best bound=5 == objective=5, 최적성 증명됨)

간호사 012345678901234567890123456789
N00    DN...ED.NN..E.N..E.N.E.DD.DEDE  (N=5)
N01    DN.ED.DEEN.E.N.DEE.EEDE.NN..EE  (N=5)
N02    N..DD.DNN.N.EDDDN...D.DE.DDD..  (N=5)
...
희망 OFF : 38건 중 38건 반영
나이트 수: 최소 5 ~ 최대 5 (편차 0)
독립 검증 스크립트: 하드 제약 위반 0건
```

25명 전원이 정확히 나이트 5개씩 — **편차 0**, 희망 OFF **100% 반영**입니다.

### 실전 확장 아이디어

| 확장 | 구현 방법 |
|---|---|
| 전월 말 근무 이어받기 | 앞에 `carry_over` 며칠을 붙이고 `model.add(x[n,d,s] == 1)`로 고정 |
| 신규/경력 조합 | 듀티마다 `sum(경력자 x) >= 1` |
| 주말 공정성 | 토/일 근무 수에도 `spread` 동일 적용 |
| 수간호사 수동 수정 후 재최적화 | 수정 칸을 `add_hint()` 또는 고정 제약으로 넣고 재실행 |
| **불가능 원인 진단** | `model.add_assumptions()` + `solver.sufficient_assumptions_for_infeasibility()` → **발표 때 아주 좋은 어필 포인트** |

## 3.3 대안 엔진: Timefold Solver (구 OptaPlanner)

- https://github.com/TimefoldAI/timefold-solver (★1,796, Apache 2.0) / 퀵스타트에 **Employee Scheduling 예제 포함**
- 2024년 6월 **공식 Python API 출시** (`pip install timefold-solver`) — 단, 내부는 JVM 기반이라 **Java 런타임 필요**
- 애노테이션 기반 도메인 모델링 (`@PlanningEntity`, `@PlanningVariable`) + Constraint Streams API + **HardSoftScore 내장**
- ⚠️ 오픈소스판 외에 Commercial edition이 있고 **멀티스레드 솔빙·제약 프로파일링은 상용 전용**
- ⚠️ 전신 OptaPlanner 저장소는 **archived**(개발 중단)

| 항목 | OR-Tools CP-SAT | Timefold |
|---|---|---|
| 설치 | `pip install ortools` — 끝 | pip + **JVM 필요** |
| 알고리즘 | SAT+LP 기반, **최적성 증명 가능** | 메타휴리스틱, 증명 불가 |
| 소규모(~50명) | **매우 빠름, 최적해 보장** | 준수하나 증명 X |
| 초대규모(수백명) | 최적 증명이 오래 걸릴 수 있음 | **강점** |
| 실시간 재계획 | 재모델링 필요 | **강점** (내장) |
| 한국어 자료 | 상대적으로 많음 | 적음 |

> **학교 프로젝트 결론: OR-Tools.** Timefold는 "대안 검토했음"으로 비교 섹션에 언급하는 정도가 적절합니다.

## 3.4 메타휴리스틱 (GA / SA / Tabu)

1990~2010년대 NRP 연구의 주류였습니다. 직접 구현하면 교육적 가치가 크지만, **결과 품질만 따지면 CP-SAT에 밀립니다.**

| 알고리즘 | 구현 난이도 | NRP 적합도 | 평가 |
|---|---|---|---|
| **Simulated Annealing** | ★☆☆ (100줄 내외) | 중 | 페널티 합산 + `random.random() < exp(-Δ/T)` 만 구현하면 됨. **직접 구현 과제로 최적** |
| **Tabu Search** | ★★☆ | **높음** (문헌 주류) | tabu list + aspiration criteria가 까다로움. 파라미터 튜닝이 성패 좌우 |
| **Genetic Algorithm** | ★★★ | 중하 | **NRP에 GA는 궁합이 나쁩니다.** 두 근무표를 교차하면 거의 항상 하드 제약이 깨져 복구(repair) 로직이 본체보다 복잡해짐 |

**이웃 연산(move) 설계가 성능을 결정합니다**: Swap(두 간호사 같은 날 교환 — 인원 수 자동 보존, 안전), Change(한 칸 변경 — 인원 제약이 깨질 수 있음), Block swap(연속 구간 통째 교환)

> **권장**: 주력은 CP-SAT. 다만 **SA를 200줄쯤 직접 구현해 CP-SAT와 나란히 비교**하면 "왜 전문 솔버를 쓰는가"에 대한 실증적 답변 + 구현 역량을 동시에 보여줄 수 있어 **비교 실험 섹션으로 최고의 소재**입니다.

## 3.5 LLM 활용 — 되는 것과 안 되는 것

**❌ 절대 하지 말 것**: LLM에게 직접 "근무표를 짜줘"라고 하기. 그럴듯하지만 제약을 위반한 표가 나옵니다. arXiv 2026의 *"The Heuristic Trap in LLM-Generated Combinatorial Solvers"* 가 지적하듯, LLM은 정확한 모델 대신 "적당히 되는 것처럼 보이는" 휴리스틱을 생성하는 경향이 있습니다.

**✅ 실용적인 3가지 용법**

1. **제약 추출** — 병원 근무 규정 문서나 과거 근무표 엑셀을 주고 "여기 적용된 규칙을 목록으로 뽑아줘". 모델링 단계 시간을 크게 줄여줍니다. 최종 제약식은 사람이 확정.
2. **INFEASIBLE 원인 설명 (가장 추천)** — CP-SAT의 `sufficient_assumptions_for_infeasibility()` 결과는 변수 이름 나열이라 사람이 못 읽습니다. 이걸 LLM에게 풀어달라고 하는 건 **명확한 가치가 있고 구현도 쉽습니다.**
	> "12월 24~25일에 D 최소 6명이 필요한데 희망 OFF 신청자가 9명이라 동시 충족 불가입니다. (1) 최소 인원을 5명으로 완화 (2) 희망 OFF 2건 반려 중 선택이 필요합니다."

3. **후처리 설명 생성** — "왜 김간호사가 이번 달 나이트 5개인지" 설명 생성 → 수용성 향상

**올바른 아키텍처**

```text
자연어 요구사항 → [LLM: 제약 추출] → 사람 검토 → 제약 정의(JSON/DSL)
                                                      ↓
                                          [CP-SAT: 실제 최적화]  ← 정확성 보장
                                                      ↓
                              근무표 → [LLM: 설명/원인 해설] → 사용자
```

**LLM은 입출력 껍데기, 최적화는 솔버.** 이 아키텍처를 명확히 제시하면 트렌디함과 신뢰성을 동시에 얻습니다.

## 3.6 참고 오픈소스

| 저장소 | ★ | 언어 | 특징 |
|---|---|---|---|
| [d-krupke/cpsat-primer](https://github.com/d-krupke/cpsat-primer) | **823** | Jupyter | **최우선 추천.** 공식 문서보다 훨씬 친절한 CP-SAT 종합 가이드 |
| [weiran-aitech/shift_schedule](https://github.com/weiran-aitech/shift_schedule) | 48 | Python | CP로 직원 교대 + 간호사 로스터링 모델링 |
| [markvincevarga/shift-optimizer](https://github.com/markvincevarga/shift-optimizer) | 15 | Python | 개인 선호도 반영 NSP 솔버 — 희망 OFF 로직 참고 |
| [j3soon/nurse-scheduling](https://github.com/j3soon/nurse-scheduling) | 14 | TypeScript | **웹 앱 형태** — UI/UX 설계 참고 |
| [TimefoldAI/timefold-quickstarts](https://github.com/TimefoldAI/timefold-quickstarts) | 585 | Java | Employee Scheduling 퀵스타트 |
| [lab-core/NurseScheduler](https://github.com/lab-core/NurseScheduler) | 35 | C++ | Branch-and-Price. 학술적 정통 접근 |
| [dwave-examples/nurse-scheduling](https://github.com/dwave-examples/nurse-scheduling) | 57 | Python | 양자 어닐링. 실용성 낮지만 "미래 기술" 소재 |

## 3.7 접근법 추천도 (학교 프로젝트 기준)

| 접근법 | 구현 난이도 | 결과 품질 | 발표 어필 | 추천도 |
|---|---|---|---|---|
| **OR-Tools CP-SAT (Python)** | ★★☆ | ★★★ | ★★★ | **★★★★★ 주력 채택** |
| **LLM + CP-SAT 하이브리드** | ★★☆ | ★★★ | ★★★ | **★★★★☆ 차별화 요소로 추가** |
| Simulated Annealing 직접 구현 | ★☆☆ | ★★☆ | ★★★ | **★★★★☆ 비교 실험용 서브** |
| Timefold Solver | ★★★ | ★★★ | ★★☆ | ★★★☆☆ 비교 대상으로만 |
| Tabu Search 직접 구현 | ★★☆ | ★★☆ | ★★☆ | ★★★☆☆ SA를 했다면 생략 가능 |
| 유전 알고리즘 | ★★★ | ★☆☆ | ★★☆ | ★★☆☆☆ NRP엔 궁합 나쁨 |
| LLM 단독 생성 | ★☆☆ | ☆☆☆ | ★☆☆ | **★☆☆☆☆ 금지** |

---

# 4. 기술 스택 · 아키텍처

## 4.1 프론트엔드

| 항목 | Next.js 16 (App Router) | React + Vite |
|---|---|---|
| 최신 | Next.js 16 (2025-10-21, Turbopack 기본, React 19.2) | React 19.x + Vite 7.x |
| 학습 난이도 | 중~상 (Server/Client Components 경계, 캐싱 규칙) | 하 (순수 CSR) |
| 백엔드 포함 | ● Route Handlers / Server Actions | ○ 별도 필요 |
| 판단 기준 | **백엔드를 따로 안 만들 때** | 백엔드를 Spring/NestJS로 분리할 때 |

> Next.js 16은 App Router가 기본이고 캐싱 개념(`Cache Components`, `use cache`)이 15→16에서 또 바뀌었으니 **공식 문서만 보고 따라하세요. 블로그 글은 대부분 구버전 기준이라 오히려 혼란을 줍니다.**

**상태관리 권장 조합: TanStack Query v5(서버 상태) + Zustand v5(클라이언트 UI 상태)**<br>"근무표 데이터는 Query, 지금 드래그 중인 셀 범위는 Zustand" — 이 역할 분리를 발표 자료에 넣으면 아키텍처 이해도로 평가받습니다.

**UI: shadcn/ui + Tailwind CSS 권장.** 근무표 매트릭스는 어차피 직접 구현해야 해서, 라이브러리의 완성된 컴포넌트보다 자유로운 셀 스타일링이 중요합니다. 대회 심사에서 "MUI 기본 테마 그대로"는 감점 요인이 되기 쉽습니다. (MUI X DataGrid Pro/Premium은 유료)

## 4.2 근무표 UI — 이 프로젝트의 진짜 승부처

### 결론: 매트릭스는 직접 구현하세요

1. 이벤트가 시간 구간이 아니라 **"하루 = 한 칸 = 한 글자"** 로 고정 → 타임라인 라이브러리의 픽셀 계산이 불필요
2. 키보드 입력(`D`/`E`/`N`/`O` 키), 드래그 범위 선택, 복사/붙여넣기 같은 **스프레드시트 UX**를 캘린더 라이브러리는 지원 안 함
3. 유료 라이선스(\$480~\$1,100+) 회피
4. 대회에서 "라이브러리 갖다 쓴 것"이 아니라 **"직접 만든 것"** 으로 평가받음

### 그래도 필요한 비교표

| 라이브러리 | 라이선스 | 가격 | 리소스 타임라인 뷰 | 무료 사용 |
|---|---|---|---|---|
| **FullCalendar** | Standard MIT / Premium 상용 | Premium **\$480/년~** | **Premium 전용** | ⚠️ 리소스 뷰 무료 불가. 단 **비상업·오픈소스 라이선스 신청 가능** → 학교 프로젝트는 신청해볼 만함 |
| **Schedule-X** | 오픈소스 부분 MIT | Premium 가격 미확인 | **Premium** | 무료는 month/week/day/agenda 4개 뷰만 |
| **react-big-calendar** | MIT | 무료 | △ 일/주 뷰 열 분할. 월간 매트릭스 불가 | ● v1.20.0, React 19 지원, 유지보수 활발 |
| **Toast UI Calendar** | MIT | 무료 | ○ | ⚠️ **2026-09-02 GitHub 아카이브 = 유지보수 중단 → 신규 채택 비권장** |
| **Bryntum Scheduler** | 상용 | **\$680~** (Pro \$1,100~) | ● 최강 | ○ 트라이얼만 |
| **DHTMLX Scheduler** | Standard **GPL v2** / Pro 상용 | 미공개 | Pro 이상 | ⚠️ 무료판 GPL v2 → **내 소스도 GPL 공개 의무**. 대회 출품에 리스크 |
| **Syncfusion** | 상용 / Community 무료 | 개인·소규모 무료 | ● | ● 학생은 Community License 가능성 높음 |

### 직접 구현 가이드

**DOM 구조 — CSS Grid + sticky** (`

| 규모 | 셀 수 | 처방 |
|---|---|---|
| 40명 × 31일 | ~1,240 | **가상화 불필요.** 그냥 렌더링해도 60fps |
| 200명 × 31일 | ~6,200 | 경계선. `React.memo` + key 안정화로 충분 |
| 1,000명 × 31일 | ~31,000 | `@tanstack/react-virtual`로 **행(row)만** 가상화 |

가상화보다 먼저 할 것:

1. 셀 컴포넌트 `React.memo` + **props는 원시값만** (객체를 그대로 넘기면 memo가 무력화됨)
2. **셀렉션 상태를 셀에 내리지 말 것.** 선택 영역은 부모에 두고 CSS 클래스나 오버레이 div 하나로 표시 — 드래그할 때마다 1,200개 셀이 리렌더되는 게 최대 병목
3. 드래그 중 상태 업데이트는 `requestAnimationFrame` 단위로 throttle, `mouseup`에서만 커밋
4. Zustand의 selector 구독으로 셀 단위 필요한 값만 구독

**셀 편집 UX**

```text
클릭 → 셀 선택 / Shift+클릭 → 범위 / 드래그 → 사각형 범위
D/E/N/O 키 → 선택 셀 일괄 입력 / Delete → 비우기
Ctrl+C·V → 범위 복사·붙여넣기(TSV) / Ctrl+Z → Undo 스택(명령 패턴)
```

- **낙관적 업데이트**: TanStack Query `useMutation`의 `onMutate`에서 캐시 먼저 변경, 실패 시 `onError`에서 롤백
- **배치 저장**: 셀마다 API 호출 금지. 디바운스 2초 또는 "저장" 버튼으로 변경분 배열을 한 번에 PATCH
- **실시간 검증 피드백**: 규칙 위반 셀에 빨간 밑줄 + 툴팁("N 다음날 D 금지"). 검증 로직을 **프론트/백엔드 공유 모듈**(`packages/rules`)로 빼면 어필 포인트
- **접근성**: `role="grid"`, `role="gridcell"`, `aria-selected`, 방향키 이동 → 심사 가점

## 4.3 백엔드

|  | Next.js Route Handlers | NestJS 11 | Spring Boot 3.x | FastAPI |
|---|---|---|---|---|
| 언어 | TypeScript | TypeScript | Java 21 LTS | Python |
| 학습 난이도 | 하 | 중 (DI, 모듈) | 중~상 | 하~중 |
| 구조화 | 약함 | **강함** | **강함** | 중간 |
| 특징 | 프론트와 단일 코드베이스 | 아키텍처 학습 | **국내 취업 수요 1위** | 솔버와 언어 통일 |

### 솔버 마이크로서비스 분리 ⭐ 대회 핵심

**왜 분리하나**: OR-Tools는 Python 라이브러리라 Node/Java 백엔드에 직접 못 넣고, 편성 연산이 수 초~수 분이라 HTTP 요청-응답 타임아웃을 초과합니다.

```text
[Next.js / NestJS 앱]
   │ POST /api/schedules/:id/generate
   ▼
[PostgreSQL]  ← job 레코드 생성 (status=PENDING)
   │ HTTP POST (간호사·규칙·요청 JSON)
   ▼
[FastAPI 솔버 서비스]  → 즉시 202 Accepted + job_id 반환
   │ enqueue
   ▼
[Worker (OR-Tools CP-SAT)] ── 진행률 콜백 ──▶ [메인 앱 Webhook]
   │ 완료
   ▼
결과 JSON → assignment 테이블 저장 → 프론트에 SSE/폴링 알림
```

**큐 선택**

| 방식 | 적합도 |
|---|---|
| FastAPI `BackgroundTasks` | ⭐ 인프라 0. 프로세스 죽으면 작업 소실 |
| **RQ (Redis Queue)** | ⭐ **권장.** Redis만 있으면 됨. Celery보다 훨씬 단순 |
| Celery + Redis/RabbitMQ | 기능 풍부하지만 이 규모엔 과함 |

> 시간이 없으면 `BackgroundTasks`로 시작하고 발표 때 "확장 시 RQ로 전환 가능한 구조"라고 설명하세요.

## 4.4 DB 설계 (PostgreSQL)

```text
hospital 1──* ward 1──* nurse_ward_assignment *──1 users
                │                                    │
             schedule ──1──* assignment *──1── shift_type
                │                 │
                │                 └──* swap_request
                ├──* shift_request
                └──* constraint_rule
audit_log — 모든 변경 이력 (polymorphic)
```

### 주요 테이블

**`users`** ⚠️ Postgres에서 `user`는 예약어 → `users` 또는 `app_userid`, `hospital_id`, `email(citext UNIQUE)`, `password_hash`, `name`, `employee_no`, `phone`, `hire_date`, `is_senior(boolean — 차지 간호사, 솔버 제약용)`, `status(ACTIVE/LEAVE/RESIGNED)`, `deleted_at(soft delete)`

- `UNIQUE (hospital_id, employee_no)`, `INDEX (hospital_id, status)`

**`role` + `user_role`** — 수간호사는 "3병동의 HEAD_NURSE"처럼 **스코프가 병동 단위**라 `ward_id`를 함께 둡니다.

```text
role: id, code enum('ADMIN','HEAD_NURSE','NURSE')
user_role: user_id, role_id, ward_id (PK 3개 조합)
```

**`shift_type`** — 병동별 커스터마이즈 가능<br>`ward_id(NULL이면 공통)`, `code('D','E','N','OFF','EDU','ANN')`, `name`, `start_time/end_time`, `duration_minutes(야간수당 계산용)`, `color`, `is_work`, `sort_order`

**`schedule`** — 월 단위 근무표<br>`ward_id`, `year`, `month`, `status(DRAFT/GENERATING/REVIEW/PUBLISHED/ARCHIVED)`, `published_at`, `solver_job_id`, `solver_score`, **`version(낙관적 락)`**

- `UNIQUE (ward_id, year, month)`

**`assignment`** — 근무 배정 (가장 큰 테이블, `bigserial` PK 권장)<br>`schedule_id`, `user_id`, `work_date(date)`, `shift_type_id`, `source(SOLVER/MANUAL/SWAP)`, `locked(재편성 시 고정)`, `note`

```sql
-- 한 사람이 하루에 한 근무만 (핵심 무결성)
CREATE UNIQUE INDEX uq_assignment_user_date
  ON assignment (schedule_id, user_id, work_date);
-- 그리드 조회 최적화
CREATE INDEX idx_assignment_schedule ON assignment (schedule_id, user_id, work_date);
-- "내 다음 근무는?" 개인 조회
CREATE INDEX idx_assignment_user_date ON assignment (user_id, work_date DESC);
-- 날짜별 인원 집계(커버리지 확인)
CREATE INDEX idx_assignment_date_shift ON assignment (work_date, shift_type_id);
```

> 규모 감각: 40명 × 31일 = 1,240행/월/병동. 10병동 × 3년 ≈ 45만 행. **파티셔닝 불필요.**

**`shift_request`** — 희망 오프/근무<br>`schedule_id`, `user_id`, `request_date`, `shift_type_id`, `priority(WISH/STRONG/MANDATORY)`, `reason`, `status`, `reviewed_by/at`

- ⚠️ `reason`에 **건강 사유 등 민감정보가 유입될 수 있습니다** → 자유 텍스트 대신 코드값 선택으로 제한하세요 (6.8 참고)

**`swap_request`** — 듀티 체인지<br>`requester_id`, `target_id`, `requester_assignment_id`, `target_assignment_id(NULL이면 단순 양도)`, `status(PENDING/ACCEPTED/APPROVED/... )`, `expires_at`

- **2단계 승인**: 상대 수락 → 수간호사 최종 승인
- ⚠️ 교환 실행은 **반드시 트랜잭션**: 두 assignment의 shift_type_id 스왑 + swap_request 상태 변경 + audit_log 기록을 하나로

**`constraint_rule`** — 병동별 규칙<br>`ward_id`, `rule_type(MIN_STAFF/MAX_CONSECUTIVE_WORK/MAX_CONSECUTIVE_NIGHT/FORBIDDEN_SEQUENCE/MIN_REST_HOURS/...)`, **`params jsonb`**, `is_hard`, `weight`, `enabled`, `effective_from/to`

> `jsonb`로 파라미터를 유연하게 두는 게 핵심. 규칙 종류가 계속 늘어나는데 컬럼을 매번 추가할 수 없습니다.

**`audit_log`** — 감사 로그 (법적 요구 + 어필 요소)<br>`actor_id`, `action`, `entity_type/entity_id`, `before/after (jsonb)`, `ip_address`, `user_agent`, `created_at`

- ⚠️ **append-only**: Postgres 레벨에서 `REVOKE UPDATE, DELETE ON audit_log`
- 개인정보 접근기록은 **최소 1년** 보관 의무

### 설계 주의사항 3가지

1. **날짜 타입**: `work_date`는 `date` (timestamptz 아님 — 근무일은 타임존 개념이 없음). 반면 `created_at` 류는 `timestamptz` + Asia/Seoul 렌더링
2. **동시 편집**: `schedule.version`으로 낙관적 락 (`UPDATE ... WHERE version = ?` → 0행이면 409)
3. **PUBLISHED 후 변경 금지**: 확정 근무표의 assignment 직접 수정을 막고 **반드시 swap_request를 통해서만** 바뀌게

### ORM: Prisma vs Drizzle

|  | Prisma | Drizzle |
|---|---|---|
| 학습 난이도 | **하** (문서·자료 압도적) | 중 (SQL을 알아야 편함) |
| 복잡한 쿼리 | 어려움 (`$queryRaw` 탈출) | **쉬움** |
| 런타임 | 무거움(엔진 바이너리) | 가벼움, 엣지 OK |
| 추천 | **처음이면 Prisma** | SQL 실력 어필하려면 Drizzle |

```text
model Assignment {
  id           BigInt    @id @default(autoincrement())
  scheduleId   String    @map("schedule_id") @db.Uuid
  schedule     Schedule  @relation(fields: [scheduleId], references: [id], onDelete: Cascade)
  userId       String    @map("user_id") @db.Uuid
  workDate     DateTime  @map("work_date") @db.Date
  shiftTypeId  String    @map("shift_type_id") @db.Uuid
  source       AssignmentSource @default(MANUAL)
  locked       Boolean   @default(false)

  @@unique([scheduleId, userId, workDate], name: "uq_assignment_user_date")
  @@index([scheduleId, userId, workDate])
  @@map("assignment")
}
```

## 4.5 인증 · 권한 (RBAC)

| 역할 | 권한 |
|---|---|
| **ADMIN** | 병원/병동/계정 관리, 전체 감사 로그 |
| **HEAD_NURSE** | **자기 병동만**: 근무표 생성·편집·확정, 요청 승인, 스왑 최종 승인, 규칙 설정 |
| **NURSE** | 본인 + 소속 병동 근무표 조회, 본인 요청 등록, 스왑 요청/수락 |

핵심은 **역할 + 스코프(병동)** 입니다. "HEAD_NURSE"만으로는 부족하고 "3병동의 HEAD_NURSE"여야 합니다. 이걸 놓치면 A병동 수간호사가 B병동 근무표를 수정할 수 있게 됩니다.

```typescript
// packages/auth/permissions.ts — 프론트·백 공유
export function can(user: SessionUser, action: Action, wardId?: string): boolean {
  if (user.roles.some(r => r.code === 'ADMIN')) return true;
  const inWard = user.roles.filter(r => r.wardId === wardId);
  switch (action) {
    case 'schedule:read':  return inWard.length > 0;
    case 'schedule:write':
    case 'schedule:publish':
    case 'swap:approve':   return inWard.some(r => r.code === 'HEAD_NURSE');
    default: return false;
  }
}
```

> ⚠️ **가장 흔한 취약점**: 프론트에서 버튼만 숨기고 API에서 재검증을 안 하는 것. **모든 API 핸들러 첫 줄에서 `can()` 호출.** 심사위원이 가장 먼저 찌르는 부분입니다.

**Auth.js (NextAuth) v5**: `npm install next-auth@beta` + `npx auth secret`. `jwt` 콜백에서 역할을 토큰에 심고 `session` 콜백에서 노출합니다. JWT는 빠르지만 역할 변경이 즉시 반영 안 되니(토큰 만료까지), 계정 정지를 즉시 반영해야 하면 `strategy: "database"`.

**Supabase Auth (대안)**: **RLS(Row Level Security)** 가 킬러 피처 — DB 레벨에서 권한을 강제해 API를 빼먹어도 안전합니다.

```sql
CREATE POLICY assignment_select ON assignment FOR SELECT
USING (EXISTS (
  SELECT 1 FROM schedule s
  JOIN nurse_ward_assignment nwa ON nwa.ward_id = s.ward_id
  WHERE s.id = assignment.schedule_id
    AND nwa.user_id = auth.uid() AND nwa.ended_on IS NULL
));
```

> "DB 레벨 RLS로 다층 방어를 구현했다"는 강력한 보안 어필 포인트입니다.

## 4.6 실시간 · 알림

| 방식 | 복잡도 | Vercel 호환 | 적합도 |
|---|---|---|---|
| **폴링** | 최저 | ● | ⭐ TanStack Query `refetchInterval: 30000` 한 줄. **MVP엔 충분** |
| **SSE** | 낮음 | △ (타임아웃 주의) | ⭐ **권장.** 알림은 단방향이라 딱 맞음. 자동 재연결 내장 |
| WebSocket | 중~상 | ○ (별도 서버) | 여러 명이 **동시 편집**할 때만 |
| Supabase Realtime | 낮음 | ● | ⭐ Supabase 쓰면 DB 변경 자동 브로드캐스트 |

> 권장 경로: 폴링 → SSE → (동시 편집이 필요해지면) WebSocket. 발표에서 **"왜 WebSocket이 아니라 SSE인가"** 를 설명할 수 있으면 오히려 가점입니다.

**웹 푸시**: Web Push API(VAPID, 무료, 표준) 권장. ⚠️ **iOS Safari는 PWA를 홈 화면에 추가해야만 동작**(16.4+) — 간호사 대부분이 아이폰이면 치명적이니 감안하세요.

**카카오 알림톡 — 학생은 사실상 불가**: **사업자등록번호**가 있어야 발신프로필을 만들 수 있습니다.<br>→ 현실적 대안: `NotificationChannel` 인터페이스로 추상화해두고 실제 데모는 Web Push + 이메일(Resend 무료 티어)로. 발표에서 "사업자 등록 후 어댑터만 교체하면 즉시 연동 가능"이라고 설명하세요. 카카오를 꼭 넣고 싶으면 **카카오 로그인(OAuth)** 은 개인 개발자도 가능합니다.

## 4.7 배포 (학생 예산)

| 플랫폼 | 무료 | 최소 비용 | 비고 |
|---|---|---|---|
| **Vercel** | Hobby (비상업 한정) | \$20/월 | Next.js 1분 배포. **서버리스라 장시간 연산 불가** |
| **Supabase** | DB 500MB, MAU 5만, 프로젝트 2개 | Pro \$25/월 | ⚠️ **1주 미사용 시 일시정지** → 대회 데모 전 반드시 깨워두기 |
| **Neon** | 0.5GB, 100 CU-시간 | 종량제 | DB 브랜칭이 강점 |
| **Railway** | 30일 \$5 크레딧 | **\$5/월** | ⭐ **파이썬 솔버 + Redis + Postgres 한 번에.** 솔버 배포 최적 |
| **Fly.io** | 체험만 | ~\$5/월 | **auto-stop/auto-start**로 트래픽 없을 때 자동 정지 |
| **Render** | 무료 웹서비스 | \$7/월~ | 무료는 15분 미사용 시 슬립(첫 요청 30초+) |
| **AWS** | 프리티어 12개월 | 과금 폭탄 위험 | ⚠️ 학생 비추천. 쓰려면 **Billing Alert 필수** |

**권장 구성 (월 \$0~5)**

```text
Vercel (무료) ── Next.js 프론트 + API
   │                    │
   ▼                    ▼
Supabase (무료)     Railway ($5/월)
  Postgres            ├ FastAPI (OR-Tools)
  Auth                ├ RQ Worker
  Realtime            └ Redis
```

**솔버 배포 주의점**

- `ortools` 패키지는 **100MB+** 라 Vercel 서버리스 함수 용량 제한(250MB)에 걸리기 쉽습니다 → **Vercel에 솔버를 올리지 마세요**
- 메모리 **최소 512MB, 권장 1GB**
- 환경변수 `SOLVER_API_KEY`로 메인 앱만 호출 가능하게 — ⚠️ 솔버 엔드포인트를 공개해두면 누구나 CPU를 태울 수 있습니다

```docker
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 4.8 개인정보 · 보안

> ⚠️ law.go.kr robots 차단으로 조문 원문 미확인. 제출 전 https://www.law.go.kr , https://www.pipc.go.kr 에서 직접 확인하세요.

| 데이터 | 분류 | 주의 |
|---|---|---|
| 이름, 사번, 연락처 | 일반 개인정보 | 수집·이용 동의 필요 (§15) |
| 근무 일정·이력 | 일반 개인정보 | 근로 관계상 필요 범위 내 |
| **희망 오프 사유** ("병원 진료", "가족 간병") | ⚠️ **건강정보 = 민감정보로 변질 가능** (§23) | **자유 텍스트 금지. "개인사정/연차/교육" 코드값 선택으로 제한** ← 가장 현실적인 회피책 |
| 주민등록번호 | 고유식별정보 (§24-2) | ❌ **절대 수집 금지.** 사번으로 충분 |
| 환자 정보 | 민감정보 + 의료법 | ❌ **스코프를 "근무표"로만 한정하세요** |

**5원칙**: 최소 수집(§16) / 목적 외 이용 금지(§18) / 안전조치 의무(§29) / **접근 기록 최소 1년 보관** / 보유기간 경과 시 파기

### 체크리스트

**🔴 반드시**

- [ ] 비밀번호 **argon2id 또는 bcrypt(cost ≥ 12)**. 평문/MD5/SHA256 단독 금지
- [ ] `.env`를 `.gitignore`에. 이미 커밋했다면 **키 전부 재발급** + 히스토리 정리
- [ ] 모든 API에서 **서버 사이드 권한 검증** (프론트 버튼 숨김은 보안이 아님)
- [ ] ORM 파라미터 바인딩 (문자열 연결 쿼리 금지)
- [ ] 세션 쿠키 `HttpOnly`, `Secure`, `SameSite=Lax`
- [ ] 데모는 **가명 데이터**. 실습 중 얻은 진짜 근무표를 절대 올리지 마세요

**🟡 하면 좋음 (어필 포인트)**

- [ ] `audit_log`로 모든 근무표 변경·조회 기록
- [ ] 로그인 실패 5회 잠금 + 레이트 리밋
- [ ] 권한 없는 사용자에겐 이름 마스킹 ("김OO")
- [ ] 전화번호 컬럼 단위 암호화 (`pgcrypto` 또는 AES-256-GCM)
- [ ] **개인정보 처리방침 페이지** 작성 → "법적 검토까지 했다"는 인상
- [ ] 보안 헤더 (CSP, X-Frame-Options, HSTS), `npm audit` / Dependabot

## 4.9 추천 스택 3안

### 🅐 가장 빠르게 (2~4주, 1~2인)

```text
Next.js 16 + shadcn/ui + TanStack Query/Zustand
매트릭스: CSS Grid 직접 구현
백엔드: Next.js Route Handlers (단일 저장소)
DB: Supabase Postgres + Prisma / 인증: Supabase Auth
자동편성: ❌ 없음 or 간단한 휴리스틱
알림: 앱 내 알림 + 폴링
배포: Vercel + Supabase → 월 $0
```

장점: 최속 완주, 비용 0 / 단점: 자동 편성이 없어 "그냥 CRUD" 인상

### 🅑 기술적으로 어필 ⭐ 대회 출품 추천 (6~10주, 2~3인)

```text
[프론트] Next.js 16 + shadcn/ui + TanStack Query/Zustand
          매트릭스 직접 구현 + react-virtual + 드래그/키보드/Undo
          PWA + Web Push
[백엔드] NestJS 11 + Prisma + PostgreSQL
          Auth.js/Passport-JWT + RBAC 가드
          SSE 알림 + audit_log 인터셉터
[솔버]   FastAPI + OR-Tools CP-SAT
          RQ(Redis) 비동기 + 진행률 콜백
          편성 품질 리포트(희망 반영률, 위반 건수, 공정성 지표)
[인프라] 모노레포(Turborepo) — packages/rules 프론트·백 공유
          Docker Compose + GitHub Actions CI
          Vercel + Railway + Supabase → 월 $0~5
```

> **발표 시나리오**: "수간호사가 근무표 한 장을 짜는 데 평균 4~8시간 걸립니다. 저희는 OR-Tools CP-SAT로 **10초**에, 희망 오프 반영률 **100%**, 나이트 편차 **0**으로 해결했습니다." — **숫자가 있는 문제 정의**가 심사에서 가장 강합니다.

### 🅒 실제 병원 도입까지 (3~6개월, 팀)

```text
Spring Boot 3.x (Java 21) + Spring Security + JPA/QueryDSL + Flyway
멀티테넌시(hospital_id 격리) + Envers/AOP 감사 로그
PostgreSQL Primary+Replica, pgcrypto 컬럼 암호화, RLS, PITR 백업
Docker + K8s, AWS 서울 리전(데이터 국내 보관), Sentry + Grafana
개인정보 처리방침/동의 관리, 접속기록 1년+ 보관
```

> 학생 기간엔 완주 불가. **구현은 🅑안으로 하고 이건 "확장 로드맵"으로 발표 자료에만** 넣으세요. 오히려 도메인 이해도가 깊다는 인상을 줍니다.

---

# 5. 착수 순서 (🅑안 기준)

| 주차 | 할 일 | 산출물 |
|---|---|---|
| 1 | 도메인 리서치(간호사 인터뷰 또는 듀티표 실물 확보), ERD 확정 | ERD, 화면 기획서 |
| 2 | DB 스키마 + 마이그레이션, 시드 데이터(가명), 인증/RBAC | 로그인 + 역할 분기 |
| 3~4 | **매트릭스 그리드 UI** (조회 → 셀 편집 → 드래그 → Undo) | 수동 편성 가능 |
| 5 | 희망 오프 요청 / 승인 플로우 | CRUD 완성 |
| 6~7 | **OR-Tools 솔버** (FastAPI + CP-SAT), 제약 모델링, 연동 | 자동 편성 동작 |
| 8 | 스왑 요청(2단계 승인), 알림(SSE + Web Push) | 핵심 기능 완료 |
| 9 | 통계 대시보드(공정성, 희망 반영률), audit_log 뷰어 | 어필 요소 |
| 10 | 배포, 성능 튜닝, 발표 자료, 데모 리허설 | 제출 |

> ⚠️ **가장 흔한 실패 패턴**: 솔버를 6주차가 아니라 1주차에 붙잡고 있다가 UI를 못 만드는 것.<br>**수동 편성이 되는 완성품을 먼저 만들고, 솔버는 그 위에 얹으세요.** 솔버가 안 되어도 제출 가능해야 합니다.

---

# 6. 후속 확인 권장 (이번 조사에서 원문 미확보)

1. **대한병원협회 「제2차 간호사 교대제 개선 시범사업 지침」 원문 PDF** — https://www.kha.or.kr 공지사항 / 서울시간호사회 http://www.seoulnurse.or.kr (97p, 2025.9.16). **근무규칙의 가장 정확한 1차 출처**
2. **병원간호사회 「간호사 야간근무 가이드라인」** — https://khna.or.kr (연속야간·최소휴식 권고치 원문)
3. **근로기준법·간호법·개인정보보호법 조문** — https://www.law.go.kr (robots 차단으로 자동 수집 불가, 수동 대조 필요)
4. 한국보건사회연구원 박경옥 외(2022) 「임상간호사의 교대제 개선을 위한 예측 가능한 패턴형 근무제」 — **패턴형 근무제 설계가 자동 생성 알고리즘의 템플릿 후보**로 직접 활용 가능
5. Schedule-X Premium 가격 (pricing 페이지 502), DHTMLX 상용 가격 (미공개)

---

# 부록: 주요 출처

**도메인 · 정책**

- 다우오피스HR 간호사 근무표 가이드 — https://hr.daouoffice.com/blog/nurse-duty-schedule
- 심평원 「간호사 교대제 개선 시범사업 수가화 방안」(윤난희, 2025) — https://repository.hira.or.kr
- 대한병원협회 시범사업 지침 — https://www.kha.or.kr
- 서울시간호사회 제2차 시범사업 지침 — http://www.seoulnurse.or.kr
- 병원간호사회 보도자료 — https://khna.or.kr/home/notice/news.php
- 정책브리핑(2025.8.29) — https://www.korea.kr
- 심평원 간호관리료 차등제 / 야간전담간호사 관리료 — https://www.hira.or.kr
- 한국노동연구원 고용 영향 분석(2025.3) — https://www.kli.re.kr
- 간호사신문 — https://www.nursenews.co.kr
- 주간경향 「'쉬어도 쉬는 게 아닌' 간호사 근무표」 — https://weekly.khan.co.kr

**경쟁 서비스**

- https://haruduty.com · https://dutymate.net · https://dutymate.kr · https://dutymaker.com · https://dutyplanners.com · https://dutyflow.co.kr · https://dutying.net
- https://shiftee.io/ko · https://jobcankr.com
- https://www.nursegrid.com · https://www.qgenda.com/solutions/nurse-staff/ · https://www.ukg.com/shiftboard · https://www.smartlinx.com/products/scheduling · https://www.deputy.com · https://wheniwork.com/pricing

**알고리즘**

- OR-Tools Employee Scheduling — https://developers.google.com/optimization/scheduling/employee_scheduling
- `shift_scheduling_sat.py` — https://github.com/google/or-tools/blob/stable/examples/python/shift_scheduling_sat.py
- CP-SAT Python API — https://developers.google.com/optimization/reference/python/sat/python/cp_model
- CP-SAT Primer — https://github.com/d-krupke/cpsat-primer
- NRP 벤치마크 — http://www.schedulingbenchmarks.org/nrp/ · https://www.schedulingbenchmarks.org/papers
- Timefold — https://docs.timefold.ai · https://timefold.ai/blog/new-open-source-solver-python

**기술 스택**

- https://nextjs.org/blog/next-16 · https://tanstack.com/query/latest · https://tanstack.com/virtual/latest · https://zustand.docs.pmnd.rs/ · https://ui.shadcn.com/
- https://fullcalendar.io/pricing · https://schedule-x.dev/docs/calendar · https://github.com/jquense/react-big-calendar/releases · https://github.com/nhn/tui.calendar · https://bryntum.com/store/ · https://www.syncfusion.com/sales/communitylicense
- https://fastapi.tiangolo.com/tutorial/background-tasks/ · https://python-rq.org/ · https://docs.nestjs.com/ · https://spring.io/projects/spring-boot
- https://www.prisma.io/docs · https://orm.drizzle.team/docs/overview · https://www.postgresql.org/docs/current/indexes.html
- https://authjs.dev/getting-started/installation · https://supabase.com/docs/guides/database/postgres/row-level-security
- https://supabase.com/pricing · https://neon.com/pricing · https://railway.com/pricing · https://docs.fly.io/about/pricing · https://vercel.com/pricing
- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events · https://developer.mozilla.org/en-US/docs/Web/API/Push_API
- https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html · https://www.w3.org/WAI/ARIA/apg/patterns/grid/
- https://www.pipc.go.kr/ · https://www.law.go.kr/
