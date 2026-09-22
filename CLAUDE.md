# 프로젝트 개요

간호사가 근무표를 작성하는 과정에서 보기 어려운 엑셀표와 달력을 하나하나 보며 불편하게 작성을 하는 것을 해결하기 위한 솔루션을 제공하는 서비스

- **Frontend**: React
- **Backend**: Spring
- **Database**: PostgreSQL
- 그 외 스택(언어 버전, 빌드 도구, 테스트 프레임워크, 배포 환경 등)은 아직 미정. 확정되는 대로 이 문서를 갱신할 것.

# 역할
- 김효주 - 인프라 구축, 프론트
- 이석찬 - 프론트
- 김민서 - 백, QA
- 최수혁 - 백, 발표

# 디렉토리 구조

Backend(3-Layered)와 Frontend(React)를 분리한 구조를 기본으로 한다. 언어/빌드 도구가 정해지면 세부 경로는 조정한다.

```
backend/
  presentation/   # Controller - 요청/응답 처리
  business/       # Service - 비즈니스 로직, 근무표 생성/검증 규칙
  persistence/    # Repository - DB 접근
  domain/         # 계층에 종속되지 않는 핵심 도메인 모델/규칙 (헥사고날 전환 시 이 쪽으로 이동)

frontend/
  components/     # 재사용 UI 컴포넌트
  pages/          # 화면 단위
  api/            # 백엔드 API 호출
```

# 아키텍처

- **현재**: 3-Layered Architecture (Presentation - Business/Service - Persistence)
- **향후 계획**: Hexagonal Architecture(포트&어댑터)로 마이그레이션 예정

3계층 구조로 개발하되, 이후 헥사고날 전환이 예정되어 있으므로 지금부터 아래 원칙을 지킨다.

- Business/Service 계층은 Controller나 Repository의 구체 구현에 직접 의존하지 않고 인터페이스를 통해서만 통신한다. 나중에 포트(인터페이스)와 어댑터(구현체)로 분리하기 쉽도록 하기 위함이다.
- 근무표 생성 규칙(예: 연속 야간 근무 제한, 최소 휴식일, 인원 배치 제약조건 등) 같은 핵심 도메인 로직은 프레임워크(Spring)나 UI(React)에 종속되지 않는 순수 로직으로 분리한다. 추후 도메인 계층으로 옮기기 쉽게 하기 위함이다.

# 협업 컨벤션

여러 명이 협업하는 프로젝트이므로 아래 규칙을 따른다.

- **협업 도구**: 자체 개발 툴 + Notion 병행, 코드/버전 관리는 GitHub ([https://github.com/Yanggochi](https://github.com/Yanggochi))
- **브랜치 전략**: `main` 보호, 기능 개발은 `feature/*`, 버그 수정은 `fix/*` 브랜치에서 작업 후 PR로 병합
- **커밋 메시지**: Conventional Commits 형식 사용 (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`, `chore:` 등)
- **PR**: 최소 1인 리뷰 승인 후 병합. 구현 중이더라도 리뷰 가능한 단위로 PR을 쪼갠다.

# 코딩 컨벤션

- 계층 간 책임을 명확히 분리한다. Controller는 요청/응답 처리만, Service는 비즈니스 로직, Repository는 데이터 접근만 담당한다.
- 세부 네이밍/포맷 규칙(린터, 포맷터 등)은 스택 세부사항이 정해지면 이 문서에 추가한다.
