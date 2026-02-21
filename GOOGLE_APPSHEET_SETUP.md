# Google(AppSheet + Google Workspace) 적용 가이드

이 문서는 현재 저장소의 `index.html` 시안을 기준으로, 실제 운영 가능한 Google 기반 고객데이터관리 앱으로 옮기는 최소 단계를 정리합니다.

## 1) 전체 아키텍처(서버비 최소)

- 데이터 저장: **Google Sheets** (초기), 또는 규모 확장 시 BigQuery 전환
- 앱 UI/권한: **AppSheet**
- 자동화: **AppSheet Automation + Google Apps Script**
- 파일 저장: **Google Drive (공유 드라이브 권장)**
- 사용자 인증: **Google Workspace 계정 로그인**

## 2) 스프레드시트 기본 테이블 구성

아래 시트를 하나의 스프레드시트에 탭으로 생성합니다.

### `customers`
- `customer_id` (Text, Key, 예: CUST-0001)
- `name` (Name)
- `phone_masked` (Text)
- `owner_email` (Email)
- `status` (Enum: 상담중/결제대기/유지고객/재접촉필요)
- `created_at` (DateTime)
- `updated_at` (DateTime)

### `activities`
- `activity_id` (Text, Key)
- `customer_id` (Ref -> customers)
- `owner_email` (Email)
- `activity_at` (DateTime)
- `memo` (LongText)

### `tasks`
- `task_id` (Text, Key)
- `title` (Text)
- `priority` (Enum: 중요/진행/완료)
- `assignee_email` (Email)
- `due_date` (Date)
- `is_done` (Yes/No)

### `users`
- `user_email` (Email, Key)
- `role` (Enum: admin/agent/viewer)
- `is_active` (Yes/No)

## 3) AppSheet 앱 생성

1. [AppSheet](https://www.appsheet.com/)에서 **Make a new app** 선택
2. 데이터 소스로 위 스프레드시트 선택
3. 테이블 연결 후 아래 핵심 설정 적용

### 컬럼 설정 핵심
- `customer_id`, `activity_id`, `task_id`는 `Initial value`에 고유 ID 식 설정
  - 예: `CONCATENATE("CUST-", UNIQUEID())`
- `updated_at`은 앱 저장 시 자동 갱신되도록 `ChangeTimestamp` 활용
- `owner_email`, `assignee_email`은 이메일 타입 지정

### 뷰 매핑(현재 시안 대응)
- `대시보드`: Dashboard view
  - 총 고객 수: `COUNT(customers[customer_id])`
  - 오늘 신규 등록: `COUNT(SELECT(customers[customer_id], TODAY() = DATE([created_at])))`
  - 미응답 문의: 상태 기반 슬라이스 집계
  - 권한 요청: users에서 대기 상태 조건 집계
- `고객 목록`: Table/Deck view (customers)
- `상담 기록`: Table view (activities)
- `권한 관리`: Table view (users, admin만 수정 허용)

## 4) 권한/보안(필수)

### 로그인 및 사용자 제한
- **Require user signin** 활성화
- **Allowed domains**를 회사 도메인으로 제한

### 역할 기반 접근 제어
`users` 테이블의 `role` 값으로 슬라이스/액션 가시성을 분기합니다.

- 관리자(admin): 전체 읽기/쓰기, 권한관리 가능
- 상담원(agent): 담당 고객/업무만 쓰기
- 조회(viewer): 읽기 전용

예시 식:
- 현재 사용자 역할 조회:
  - `LOOKUP(USEREMAIL(), "users", "user_email", "role")`
- 관리자 여부:
  - `LOOKUP(USEREMAIL(), "users", "user_email", "role") = "admin"`

### 민감정보 처리
- 원본 전화번호 대신 `phone_masked` 저장/표시
- 상세 개인정보 컬럼은 `Show_If`로 관리자만 노출

## 5) 자동화

- 신규 고객 등록 시 담당자에게 Gmail/Chat 알림
- `status`가 "재접촉필요"로 바뀌면 `tasks` 자동 생성
- 매일 09:00 미완료 업무 리마인드 발송

## 6) 운영 체크리스트

- 공유 드라이브 소유권(개인 계정 종속 방지)
- 퇴사자 계정 비활성화 프로세스
- 주 1회 백업(스프레드시트 사본 자동 생성)
- 월 1회 접근권한 점검

## 7) 확장 시점

아래 신호가 오면 Sheets에서 DB로 이전을 검토합니다.

- 행 수/동시사용자 증가로 속도 저하
- 복잡한 조인/집계 증가
- 감사로그, 세밀 권한, 장기 보관 요구 증가

권장 전환 경로:
- `Google Sheets -> BigQuery` 또는
- `AppSheet + Cloud SQL(PostgreSQL)`

---

원하면 다음 단계로, 이 문서를 기준으로 실제 **샘플 시트 템플릿(컬럼 헤더 포함 CSV)** 파일까지 저장소에 만들어 바로 Import 할 수 있게 제공할 수 있습니다.
