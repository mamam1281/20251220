# 온보딩 학습 체크리스트 (Step-by-step)

> 목적: 이 문서 순서대로 실행하면 **로컬에서 서비스가 뜨고**, 최소 스모크 테스트까지 완료됩니다.
> 
> 기본 가정: Windows PowerShell, 작업 경로는 저장소 루트.

---

## 0. 현재 상태 확인 (필수)

- [x] 저장소 루트 확인
  - 기대: `docker-compose.yml`, `alembic.ini`, `app/`, `src/`가 보임
- [x] 환경 파일 존재 확인
  - 기대: `.env`(로컬/도커 실행용, gitignore), `.env.example`, `.env.development`, `.env.frontend`(또는 `.env.frontend.example`) 존재
  - 참고: `.env`가 없으면 `docker compose`의 `env_file: .env`에서 실패할 수 있으니 `.env.example` 또는 `.env.local`을 복사해 생성

---

## 1. 로컬 사전점검 (필수)

- [x] Docker 설치 및 동작 확인 (Docker 트랙 선택 시 필수)
  - 명령: `docker --version`, `docker compose version`
  - 기대: 버전 출력, compose가 v2 계열
- [x] Node/NPM 확인
  - 명령: `node --version`, `npm --version`
  - 기대: Node 18+, npm 10+ 권장
- [x] Python 확인
  - 명령: `python --version`
  - 기대: Python 3.11+

---

## 2. 실행 트랙 선택 (둘 중 하나만)

### 트랙 A) Docker Compose (권장: DB/Redis까지 한번에)

- [x] 컨테이너 빌드/기동
  - 명령: `docker compose up -d --build`
  - 기대: `xmas-db`, `xmas-backend`, `xmas-frontend`, `xmas-nginx` 등이 Up
- [x] 마이그레이션 적용 (중요: 자동 아님)
  - 명령: `docker compose exec backend alembic upgrade head`
  - 기대: 에러 없이 완료
  - 참고: Compose 기본 DB 이름은 `MYSQL_DATABASE`(기본값 `xmas_event`)이며, 백엔드는 `DATABASE_URL`로 이 DB를 바라봄

- 접속 포트(기본):
  - Backend: `http://localhost:8000/`
  - Frontend: `http://localhost:3000/`
  - Nginx: `http://localhost/`
  - DB: `localhost:3307`

### 트랙 B) 네이티브 (로컬에서 직접 실행)

- [ ] (선택) MySQL 준비 (로컬 설치 또는 도커로 mysql만 실행)
- [ ] Python 가상환경 및 의존성 설치
  - 명령:
    - `python -m venv venv`
    - `./venv/Scripts/Activate.ps1`
    - `pip install -r requirements.txt`
- [ ] 마이그레이션 적용
  - 명령: `alembic upgrade head`
- [ ] 백엔드 실행
  - 명령: `uvicorn app.main:app --reload --host 0.0.0.0 --port 8000`
- [ ] 프론트 의존성 설치/실행
  - 명령:
    - `npm install`
    - `npm run dev -- --host --port 5173`

---

## 3. 스모크 테스트 (필수)

- [x] Backend 헬스 확인
  - URL: `http://localhost:8000/` 또는 `http://localhost:8000/health`
- [x] Swagger 확인
  - URL: `http://localhost:8000/docs`
- [x] 토큰 발급(유저 자동 생성)
  - 예시:
    - `curl -X POST http://localhost:8000/api/auth/token -H "Content-Type: application/json" -d "{\"external_id\":\"test-qa-999\"}"`

### (추가) 도메인 플로우 스모크: 3개 게임 + 시즌패스

- [x] (권장) 로컬 QA에서는 Backend `TEST_MODE=true`로 토큰 자동 보충 활성화
  - 파일: `.env`
  - 운영에서는 반드시 `TEST_MODE=false` (또는 미설정)
- [x] 같은 JWT로 아래 순서대로 호출해 "끝까지" 통과하는지 확인
  1) `GET/POST /api/roulette/status|play`
  2) `GET/POST /api/dice/status|play`
  3) `GET/POST /api/lottery/status|play`
  4) `GET /api/season-pass/status`
  5) `POST /api/season-pass/stamp`

### (추가) 유저플로우 스모크: 외부입금(External Ranking) → 시즌패스 영향 (필수)

- [ ] (사전) JWT 발급 (유저 플로우 기준)
  - 라우트: `POST /api/auth/token`
  - 예시:
    - `curl -X POST http://localhost:8000/api/auth/token -H "Content-Type: application/json" -d "{\"external_id\":\"ext-deposit-qa-001\"}"`
  - 기대: 응답에서 `access_token` 확보

- [ ] (유저) 오늘 랭킹 조회(외부랭킹 포함) 라우트 확인
  - 라우트: `GET /api/ranking/today?top=10`
  - 예시(Authorization 필요):
    - `curl http://localhost:8000/api/ranking/today?top=10 -H "Authorization: Bearer <JWT>"`
  - 기대: 응답의 `external_entries`에 외부랭킹 리스트가 포함될 수 있음

- [ ] (관리자) 외부입금/이용횟수(외부 랭킹 데이터) 업서트
  - 라우트: `POST /admin/api/external-ranking`
  - 의미: `deposit_amount`, `play_count`는 “일일 누적 총액/총횟수”로 넣는 값이며, 서버가 이전 값 대비 증분(delta)만 계산해 시즌패스 XP에 반영
  - 예시(테스트 유저 id=1):
    - `curl -X POST http://localhost:8000/admin/api/external-ranking -H "Content-Type: application/json" -d "[{\"user_id\":1,\"deposit_amount\":100000,\"play_count\":0,\"memo\":\"qa\"}]"`
    - 같은 날 재업서트는 “누적치”로 전송: `deposit_amount: 200000` 처럼 증가된 총액을 전송

- [ ] 같은 유저 JWT로 시즌패스/랭킹 재조회 후 영향 확인
  - 시즌패스 라우트: `GET /api/season-pass/status`
    - `curl http://localhost:8000/api/season-pass/status -H "Authorization: Bearer <JWT>"`
  - 랭킹 라우트: `GET /api/ranking/today?top=10`
    - `curl http://localhost:8000/api/ranking/today?top=10 -H "Authorization: Bearer <JWT>"`
  - 기대:
    - 시즌패스: 누적 예치가 10만원 단위로 늘어날 때마다 `progress.current_xp`가 20씩 증가(기본값 기준)
    - 랭킹: `external_entries` 정렬(예치금 desc → 이용횟수 desc → user_id asc) 반영
  - 기본 정책(환경변수):
    - `EXTERNAL_RANKING_DEPOSIT_STEP_AMOUNT=100000` (단위)
    - `EXTERNAL_RANKING_DEPOSIT_XP_PER_STEP=20` (단위당 지급 XP)
    - `EXTERNAL_RANKING_DEPOSIT_MAX_STEPS_PER_DAY=50` (0이면 무제한)
    - `EXTERNAL_RANKING_DEPOSIT_COOLDOWN_MINUTES=0`
  - 기대: 누적 예치가 10만원 단위로 늘어날 때마다 시즌패스 `progress.current_xp`가 20씩 증가
- [ ] (주간) 외부랭킹 TOP10 스탬프 지급 확인
  - 조건: 예치/횟수 정렬 기준 TOP10에 포함
  - 기대: `season_pass_stamp_log`에 `source_feature_type=EXTERNAL_RANKING_TOP10` 1회 기록(같은 주에는 중복 지급 없음)

### (추가) 시즌패스 레벨 프로그래스 바 확인 (필수)

- [ ] 상단 시즌패스 바 진행률이 “현재 구간(이전 임계값~다음 임계값)” 기준으로 증가하는지 확인
  - 확인 위치: 메인 레이아웃 상단 `시즌 패스 진행도` 바
  - 기대: XP가 다음 임계값에 가까워질수록 0→100%로 증가하고, 레벨업 시 다시 0% 근처로 리셋

---

## 4. 핵심 도메인 학습(추천 순서)

- [ ] 시즌패스 흐름 이해
  - 문서: `docs/03_api/03_api_season_pass_v1.0.md`
  - 포인트: stamp/claim, `NO_ACTIVE_SEASON`, auto-claim
- [ ] 게임 토큰(지갑/원장) 정책 이해
  - 문서: `docs/ONBOARDING.md`
  - 포인트: `TEST_MODE=true` 시 자동 보충(QA 전용)

---

## 5. 테스트/검증 (선택)

- [ ] 백엔드 테스트
  - 명령: `pytest -q`
- [ ] 프론트 타입체크
  - 명령: `npx tsc --noEmit`
- [ ] 프론트 테스트
  - 명령: `npm run test`

---

## 6. 운영/보안 체크(배포 전 필수)

- [ ] 운영에서 `TEST_MODE=false` 확인
- [ ] Docker(Compose) 기준 DB 선택이 올바른지 확인
  - 기대: backend `DATABASE_URL`이 `.../xmas_event`를 가리킴(기본)
  - 확인(예): `docker compose exec backend printenv | grep DATABASE_URL`
  - 참고: 네이티브 로컬 개발 가이드에서는 `xmas_event_dev`를 사용하므로 혼동 주의
- [ ] DB/Redis 외부 포트 노출 제한 검토
  - 문서: `docs/SECURITY_GUIDE.md`
  - 예: DB 포트를 `127.0.0.1:3307:3306`로 제한하거나 포트 제거
- [ ] HTTPS 적용(도메인 운영 시)
  - 문서: `docs/SECURITY_GUIDE.md`

