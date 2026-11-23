# Agent Challenge 2026 Leaderboard

Vue 3와 정적 JSON으로 운영하는 공개 해커톤 리더보드입니다. 별도의 API 서버, 데이터베이스, 로그인 없이 GitHub Pages에 배포할 수 있으며, 운영자는 `public/data/*.json`과 GitHub Release만 관리하면 됩니다.

## 주요 기능

- 전체 참가 팀, 완료 결과, 시나리오와 태스크 현황 요약
- 시나리오-태스크별 리더보드와 동적 지표 컬럼
- 팀별 최고 완료 점수 기준 순위 및 동점 경쟁 순위
- 팀별 응시 현황과 회차별 상세 측정 결과
- 시나리오·태스크 검색 및 GitHub Release ZIP 다운로드
- `active`/`coming-soon` 상태에 따른 공개 범위 제어
- JSON 구조, 날짜, 참조 무결성 검증과 잘못된 행 제외 경고
- GitHub Actions를 통한 GitHub Pages 자동 배포

## 화면 구성

| 경로                                 | 설명                               |
| ------------------------------------ | ---------------------------------- |
| `/#/overview`                        | 참가 현황과 시나리오별 태스크 요약 |
| `/#/leaderboard/:scenarioId/:taskId` | 태스크별 공식 순위와 세부 지표     |
| `/#/attempts/:scenarioId/:taskId`    | 팀별 대표 회차와 전체 회차 상세    |
| `/#/data`                            | 공개 태스크 검색 및 자료 다운로드  |
| `/#/submit`                          | 참가자 제출 방식 안내              |

GitHub Pages에서 직접 접속하거나 새로고침해도 404가 발생하지 않도록 해시 라우팅을 사용합니다.

## 기술 스택

- Vue 3, TypeScript, Vite
- Pinia, Vue Router
- Tailwind CSS 4, Pretendard
- GitHub Pages, GitHub Actions, GitHub Releases

## 프로젝트 구조

```text
.
├─ .github/workflows/deploy-pages.yml  # Pages 빌드·배포
├─ frontend/
│  ├─ public/data/                     # 운영 데이터(JSON)
│  │  ├─ teams.json
│  │  ├─ tasks.json
│  │  ├─ leaderboard.json
│  │  └─ attempts.json
│  └─ src/
│     ├─ components/                   # 레이아웃·리더보드·공통 UI
│     ├─ router/                       # 해시 라우팅
│     ├─ stores/challenge.ts           # 로딩·검증·순위 계산
│     ├─ types/challenge.ts            # 데이터 타입
│     └─ views/                        # 공개 화면
└─ LICENSE
```

## 로컬 실행

Node.js 22 LTS를 권장합니다.

```bash
cd frontend
npm ci
npm run dev
```

개발 서버는 기본적으로 `http://localhost:4173`에서 실행됩니다.

```bash
npm run typecheck     # TypeScript/Vue 타입 검사
npm run build         # 타입 검사 후 프로덕션 빌드
npm run preview       # dist 로컬 미리보기
npm run format        # Prettier 적용
npm run format:check
```

### 환경 변수

| 변수                             | 용도                               | 예시               |
| -------------------------------- | ---------------------------------- | ------------------ |
| `VITE_BASE_PATH`                 | 사이트가 배포될 Pages 하위 경로    | `/LeaderBoard/`    |
| `VITE_GITHUB_RELEASE_REPOSITORY` | 자료를 배포하는 `owner/repository` | `acme/LeaderBoard` |
| `VITE_GITHUB_RELEASE_TAG`        | 고정 릴리즈 태그 또는 최신 릴리즈  | `v1.0.0`, `latest` |

`VITE_BASE_PATH`는 `/`로 시작하고 끝나야 합니다. 사용자·조직 루트 Pages는 `/`, 프로젝트 Pages는 `/<repository-name>/`을 사용합니다.

PowerShell에서 Pages 경로와 Release 연동을 포함해 빌드하려면 다음과 같이 설정합니다.

```powershell
$env:VITE_BASE_PATH='/LeaderBoard/'
$env:VITE_GITHUB_RELEASE_REPOSITORY='owner/LeaderBoard'
$env:VITE_GITHUB_RELEASE_TAG='latest'
npm run build
Remove-Item Env:VITE_BASE_PATH
Remove-Item Env:VITE_GITHUB_RELEASE_REPOSITORY
Remove-Item Env:VITE_GITHUB_RELEASE_TAG
```

Release 관련 변수가 없으면 Data 화면의 릴리즈 바로가기와 ZIP 다운로드 기능은 표시되지 않습니다.

## 운영 데이터 관리

모든 JSON 파일은 `frontend/public/data/`에 있습니다. 앱은 처음 열 때 네 파일을 동시에 불러오며, 화면의 **새로고침** 버튼을 누르면 캐시를 우회해 다시 요청합니다.

### 1. 팀 등록: `teams.json`

`teamId`는 결과 파일에서 참조하므로 등록 후 변경하지 않는 식별자로 사용합니다. 팀명은 `teamName`만 수정하면 모든 화면에 반영됩니다.

```json
{
  "teams": [{ "teamId": "team-003", "teamName": "새 팀" }]
}
```

### 2. 시나리오와 태스크 등록: `tasks.json`

```json
{
  "id": "scenario-01",
  "name": "기본 협업",
  "description": "여러 로봇이 함께 과제를 수행합니다.",
  "tone": "blue",
  "tasks": [
    {
      "id": "task-01",
      "name": "지도 제작",
      "status": "active",
      "download": { "asset": "scenario-01-task-01.zip" },
      "leaderboard": {
        "title": "기본 협업 지도 제작",
        "description": "공식 순위를 확인합니다.",
        "note": "COMPLETED 결과 중 팀별 최고 점수만 반영합니다.",
        "columns": [
          { "key": "team", "label": "팀", "format": "text" },
          { "key": "score", "label": "응시 점수", "format": "score" },
          { "key": "exploration", "label": "탐사율", "format": "percent" }
        ],
        "baseline": { "score": 1.0, "exploration": 95 }
      }
    }
  ]
}
```

- `tone`: `blue`, `orange`, `violet` 중 하나입니다.
- `status`: `active`는 화면과 링크를 활성화하고, `coming-soon`은 준비 중으로 표시합니다.
- `download.asset`: GitHub Release에 업로드할 파일명과 정확히 일치해야 합니다.
- `leaderboard.columns`: 표시할 열과 순서를 정의합니다. `team`과 `score`는 각각 팀명과 최종 점수를 사용하고, 나머지 키는 결과의 `metrics`에서 찾습니다.
- `format`: `number`, `score`, `percent`, `seconds`, `megabytes`, `status`, `text`를 지원합니다.
- `baseline`: 선택 사항이며 리더보드 하단에 기준 정책 행을 표시합니다.

### 3. 리더보드 결과 등록: `leaderboard.json`

```json
{
  "updatedAt": "2026-11-23T14:30:00+09:00",
  "results": [
    {
      "submissionId": "team-003-s01-t01-v001",
      "teamId": "team-003",
      "scenarioId": "scenario-01",
      "taskId": "task-01",
      "score": 1.184,
      "status": "COMPLETED",
      "submittedAt": "2026-11-23T13:30:00+09:00",
      "gradedAt": "2026-11-23T14:10:00+09:00",
      "metrics": {
        "exploration": 97,
        "consistency": 95,
        "modelSize": 0.42,
        "reproduction": "재현됨"
      }
    }
  ]
}
```

- `status`는 `COMPLETED`, `FAILED`, `PENDING` 중 하나입니다.
- 순위에는 `COMPLETED` 결과만 포함되며 팀마다 가장 높은 `score` 하나를 사용합니다.
- 점수 내림차순으로 정렬하고, 동점은 같은 순위를 부여한 뒤 다음 순위를 건너뛰는 경쟁 순위를 적용합니다.
- `metrics`의 키는 해당 태스크의 `leaderboard.columns[].key`와 맞춰야 합니다.
- 잘못된 숫자·날짜·상태 또는 존재하지 않는 팀/태스크 참조는 그 결과만 제외하고 화면에 경고합니다.

### 4. 응시 상세 등록: `attempts.json`

```json
{
  "updatedAt": "2026-11-23T14:30:00+09:00",
  "attempts": [
    {
      "attemptId": "team-003-s01-t01-r01",
      "teamId": "team-003",
      "scenarioId": "scenario-01",
      "taskId": "task-01",
      "round": 1,
      "completed": true,
      "submittedAt": "2026-11-23T13:30:00+09:00",
      "explorationRate": 97,
      "consistencyRate": 95,
      "contacts": 1,
      "taskScore": 0.445,
      "communicationSavings": 55,
      "finalScore": 1.065,
      "measurementStatus": "정상"
    }
  ]
}
```

측정값은 숫자 또는 `null`을 사용할 수 있습니다. 팀 목록은 이름순, 각 팀의 회차는 `round` 오름차순으로 표시됩니다. 요약 표에는 첫 회차가 보이며 팀 행을 선택하면 등록된 전체 회차가 펼쳐집니다.

## GitHub Release 자료 배포

1. `tasks.json`의 각 태스크에 `download.asset` 파일명을 지정합니다.
2. GitHub 저장소에서 Release를 만들고 동일한 이름의 ZIP 파일을 자산으로 업로드합니다.
3. Actions Variables에 `VITE_GITHUB_RELEASE_REPOSITORY`와 `VITE_GITHUB_RELEASE_TAG`를 설정합니다.
4. 사이트를 다시 배포합니다.

사용자가 다운로드를 누르면 GitHub Releases API로 자산 존재 여부를 먼저 확인합니다. 파일이 있으면 다운로드하고, 없거나 API 확인에 실패하면 안내 모달과 릴리즈 페이지 링크를 표시합니다. 비공개 저장소나 인증이 필요한 자산을 브라우저에 노출하는 용도로 사용하면 안 됩니다.

## GitHub Pages 배포

1. 저장소 **Settings → Pages → Source**에서 **GitHub Actions**를 선택합니다.
2. **Settings → Secrets and variables → Actions → Variables**에 필요한 환경 변수를 등록합니다.
3. `main` 브랜치에 `frontend/**` 또는 배포 workflow 변경을 push합니다.
4. **Actions → Deploy GitHub Pages**에서 빌드와 배포 결과를 확인합니다.

workflow는 Node.js 22에서 `npm ci`, 타입 검사, 프로덕션 빌드, Pages artifact 업로드와 배포를 순서대로 수행합니다. 기본 브랜치가 `main`이 아니라면 `.github/workflows/deploy-pages.yml`의 `branches`도 변경해야 합니다.

```text
JSON/코드 수정 → commit 및 push → GitHub Actions → GitHub Pages 반영
```

## 운영 시 주의사항

- `teamId`, `scenarioId`, `taskId`는 모든 JSON에서 정확히 일치해야 합니다.
- 날짜는 시간대가 포함된 ISO 8601 형식을 권장합니다.
- JSON은 주석과 마지막 쉼표를 지원하지 않습니다.
- GitHub Pages에 토큰, 비밀번호, PAT, 실제 `.env` 또는 비공개 저장소 자격 증명을 넣지 마세요.
- 현재 사이트는 참가자 파일을 직접 받거나 검증·채점하지 않습니다. 제출 방식은 별도 운영 정책이 확정된 뒤 Submit 화면에서 안내합니다.

## License

The leaderboard application source code is licensed under the MIT License.

Participant submissions and third-party materials are not covered by the application's MIT License. Their use is governed by the applicable submission terms and the license declarations included with each submission. The Pretendard dependency retains its own license.
