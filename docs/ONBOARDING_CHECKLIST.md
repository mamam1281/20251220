# 온보딩 학습 체크리스트 (Step-by-step)

> 목적: 이 문서 순서대로 실행하면 **로컬에서 서비스가 뜨고**, 최소 스모크 테스트까지 완료됩니다.
> 
> 기본 가정: Windows PowerShell, 작업 경로는 저장소 루트.

---

## 0. 현재 상태 확인 (필수)

- [ ] 저장소 루트 확인
  - 기대: `docker-compose.yml`, `alembic.ini`, `app/`, `src/`가 보임
- [ ] 환경 파일 존재 확인
  - 기대: `.env`, `.env.development`, `.env.frontend.local` 존재

---

## 1. 로컬 사전점검 (필수)

- [ ] Docker 설치 및 동작 확인 (Docker 트랙 선택 시 필수)
  - 명령: `docker --version`, `docker compose version`
  - 기대: 버전 출력, compose가 v2 계열
- [ ] Node/NPM 확인
  - 명령: `node --version`, `npm --version`
  - 기대: Node 18+, npm 10+ 권장
- [ ] Python 확인
  - 명령: `python --version`
  - 기대: Python 3.11+

---

## 2. 실행 트랙 선택 (둘 중 하나만)

### 트랙 A) Docker Compose (권장: DB/Redis까지 한번에)

- [ ] 컨테이너 빌드/기동
  - 명령: `docker compose up -d --build`
  - 기대: `xmas-db`, `xmas-backend`, `xmas-frontend`, `xmas-nginx` 등이 Up
- [ ] 마이그레이션 적용 (중요: 자동 아님)
  - 명령: `docker compose exec backend alembic upgrade head`
  - 기대: 에러 없이 완료

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

- [ ] Backend 헬스 확인
  - URL: `http://localhost:8000/` 또는 `http://localhost:8000/health`
- [ ] Swagger 확인
  - URL: `http://localhost:8000/docs`
- [ ] 토큰 발급(유저 자동 생성)
  - 예시:
    - `curl -X POST http://localhost:8000/api/auth/token -H "Content-Type: application/json" -d "{\"external_id\":\"test-qa-999\"}"`

### (추가) 도메인 플로우 스모크: 3개 게임 + 시즌패스

- [ ] (권장) 로컬 QA에서는 Backend `TEST_MODE=true`로 토큰 자동 보충 활성화
  - 파일: `.env`
  - 운영에서는 반드시 `TEST_MODE=false` (또는 미설정)
- [ ] 같은 JWT로 아래 순서대로 호출해 "끝까지" 통과하는지 확인
  1) `GET/POST /api/roulette/status|play`
  2) `GET/POST /api/dice/status|play`
  3) `GET/POST /api/lottery/status|play`
  4) `GET /api/season-pass/status`
  5) `POST /api/season-pass/stamp`

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
- [ ] DB/Redis 외부 포트 노출 제한 검토
  - 문서: `docs/SECURITY_GUIDE.md`
  - 예: DB 포트를 `127.0.0.1:3307:3306`로 제한하거나 포트 제거
- [ ] HTTPS 적용(도메인 운영 시)
  - 문서: `docs/SECURITY_GUIDE.md`

