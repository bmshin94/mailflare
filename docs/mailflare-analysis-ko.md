# Mailflare 전수조사 분석 정리 (한국어)

> 이 문서는 Mailflare 저장소를 전수조사한 결과와, 설치·활용·수익화에 대한
> 논의를 정리한 기록입니다.

- **원본 저장소:** https://github.com/hieunc229/mailflare
- **이 저장소 (포크):** https://github.com/bmshin94/mailflare
- **공식 사이트:** https://mailflare.co/
- **원클릭 배포:** https://deploy.workers.cloudflare.com/?url=https://github.com/hieunc229/mailflare
- **라이선스:** AGPL-3.0
- **작성일:** 2026-09-27

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [쉬운 설명 (비유 버전)](#2-쉬운-설명-비유-버전)
3. [설치 및 사용법](#3-설치-및-사용법)
4. [정체 정리: 플러그인 / 스킬 / MCP?](#4-정체-정리-플러그인--스킬--mcp)
5. [API 토큰 정리](#5-api-토큰-정리)
6. [GitHub에서 유명한 이유](#6-github에서-유명한-이유)
7. [로컬 AI 에이전트 구축 활용성](#7-로컬-ai-에이전트-구축-활용성)
8. [React / PHP 구현 가능성](#8-react--php-구현-가능성)
9. [수익화 아이디어](#9-수익화-아이디어)
10. [핵심 주의사항 (Gotchas)](#10-핵심-주의사항-gotchas)

---

## 1. 프로젝트 개요

### 한 줄 요약

Mailflare는 내 도메인(`@example.com`)으로 쓰는 **셀프호스팅 웹메일 서비스**다.
Cloudflare 인프라 위에서 사실상 무료(발송만 월 $5)로 돌아가는 완성형 제품이다.

### 규모

| 항목 | 수치 |
|---|---|
| 총 코드량 | 약 38,700줄 (TS/TSX) |
| API 엔드포인트 | 88개 |
| DB 테이블 | 29개 |
| DB 마이그레이션 | 31개 |
| `src/lib` 모듈 | 167개 파일 |
| 커밋 / 기여자 | 115 커밋 / 13명 |
| 개발 기간 | 2026-06-19 ~ 2026-09-18 (약 3개월) |
| GitHub 지표 | ⭐ 3.6k / 🍴 499 |

### 기술 스택

```
Next.js 16 (App Router) + React 19 + TypeScript
  └ Tailwind v4 + shadcn/Radix UI
  └ TanStack Query
Cloudflare Workers (OpenNext로 변환 배포)
  ├ D1        → SQLite DB (메일 메타데이터, 계정, 규칙)
  ├ R2        → 원본 MIME, 첨부파일, 백업
  ├ Queues    → 비동기 메일 처리 (inbound / outbound)
  ├ Durable Object → 실시간 WebSocket 알림
  ├ Email Routing  → 메일 수신
  ├ send_email 바인딩 → 메일 발송
  └ Cron      → 매일 02:00 UTC 자동 백업
Drizzle ORM
```

### 폴더 구조

```
mailflare/
├── worker.ts                실제 진입점. Next.js가 표현할 수 없는 핸들러 담당
│                              ├ fetch()     → /api/realtime WebSocket 업그레이드
│                              ├ email()     → Cloudflare 메일 수신 핸들러
│                              ├ queue()     → 큐 소비자 (inbound/outbound/웹훅 재시도)
│                              └ scheduled() → 매일 DB 백업 크론
├── wrangler.jsonc           Cloudflare 바인딩 선언 (D1/R2/Queue/DO/크론/레이트리밋)
├── src/
│   ├── app/
│   │   ├── (dashboard)/     inbox, sent, drafts, spam, trash, starred,
│   │   │                    snoozed, archived, folders, calendar, compose,
│   │   │                    rules, import-export
│   │   ├── (admin)/         domains, mailboxes, accounts, api-keys, webhooks,
│   │   │                    audit-logs, backups, branding, licenses,
│   │   │                    routing, activity
│   │   ├── (auth)/          login, register, setup, onboarding,
│   │   │                    forgot-password, reset-password
│   │   ├── (settings)/      account, auto-reply, inbox, rules, import/export
│   │   ├── api/             88개 REST 엔드포인트
│   │   │   └── v1/          외부 공개 API (send, messages) — API 키 인증
│   │   ├── jmap/            JMAP 서버 (표준 메일 프로토콜)
│   │   └── .well-known/jmap JMAP 자동 발견
│   ├── lib/                 비즈니스 로직 본체 (167개 파일)
│   │   ├── email/           수신/발송/스레딩/라우팅/웹훅/첨부/주소 파싱
│   │   ├── auth/            세션, 쿠키, TOTP(2FA), 비밀번호 재설정
│   │   ├── api/             API 키 인증 + 스코프 권한
│   │   ├── domains/         Cloudflare 존 프로비저닝 / DNS / 정리
│   │   ├── mailboxes/       메일박스 공유·권한 (access.ts)
│   │   ├── spam/            자체 스팸 필터 (베이지안 학습, 외부 전송 없음)
│   │   ├── search/          FTS5 전문검색 + Gmail 문법 파서
│   │   ├── jmap/            프레임워크 독립 JMAP 구현체
│   │   ├── licenses/        Paymug 라이선스 검증 (Pro/Team 브랜딩 게이트)
│   │   ├── backups/         JSON 백업/복원
│   │   ├── import/          IMAP 마이그레이션
│   │   ├── calendar/        캘린더 이벤트
│   │   ├── realtime/        Durable Object 허브
│   │   └── setup/migration.ts  스키마 전체를 inline SQL로 중복 보유 (주의!)
│   ├── db/schema/index.ts   29개 테이블 단일 파일 정의
│   └── components/          UI 컴포넌트 (compose 에디터, 메시지 뷰어 등)
├── server/                  Docker 셀프호스팅용 Node 런타임
│   └── runtime/             D1↔better-sqlite3, R2↔로컬파일, Queue↔인메모리,
│                            DO↔WebSocket 허브, SMTP 리스너(포트 25), nodemailer
├── deploy/cloudflare-email-relay/  MX는 CF에 두고 내 서버로 릴레이하는 미니 Worker
├── drizzle/migrations/      31개 SQL 마이그레이션
├── docs/                    deployment / api / self-hosting / spam / troubleshooting
├── tests/                   .mjs 스크립트 2개 (테스트 러너는 없음)
└── .github/workflows/deploy-update.yml  자체 업데이트 워크플로
```

### 핵심 아키텍처

#### 1) 메일 수신 파이프라인

```
발신자
  → Cloudflare Email Routing (MX)
  → worker.ts email() 핸들러
       ├ 도메인 라우팅 규칙 판정 (reject / forward / store)
       │   ※ reject·forward는 여기서만 가능 (setReject/forward API 제약)
       ├ 계정 전달 설정 적용 (루프 방지 헤더)
       ├ 원본 MIME을 R2에 저장 (유실 방지)
       └ INBOUND_QUEUE에 enqueue (인라인 파싱 금지)
  → queue() 소비자
       → processInboundMessage()
            ├ 주소 해석 (메일박스 / 별칭 / 캐치올)
            ├ postal-mime으로 MIME 파싱
            ├ 스팸 점수 계산 (70점 이상 → 스팸함)
            ├ messages + attachments INSERT
            ├ 연락처 upsert
            ├ 스레딩 (In-Reply-To / References 매칭)
            ├ 웹훅 발송
            └ Durable Object로 실시간 알림 push
```

#### 2) 이중 런타임

동일 코드가 Cloudflare Workers와 Docker/Node 양쪽에서 동작한다.
`getEnv()` 하나로 추상화하고, Node 모드에서는 `server/runtime/env.ts`가
D1·R2·Queue·DO를 같은 인터페이스로 구현해 주입한다. 애플리케이션 코드는
어느 런타임인지 알 필요가 없다.

#### 3) 두 가지 인증 표면

- **세션 쿠키**(`ep_session`) — 웹 대시보드용
- **API 키 베어러 토큰** — `/api/v1/*`, JMAP용 (스코프 권한 있음)

메일박스 권한은 유저 role과 별개로 `mailbox_access` 공유 테이블로 관리된다.
덕분에 "팀 공용 support@ 메일박스" 같은 구성이 가능하다.

#### 4) 라우팅 규칙의 두 스코프

| 스코프 | 평가 시점 | 가능한 액션 |
|---|---|---|
| `domain` | 주소 해석 중 (`resolveInboundAddress`) | reject / forward / store |
| `mailbox` | 배달 후 (`resolveInboxRuleDestination`) | 폴더 지정 / 스팸·휴지통 이동 |

`domain` 스코프는 reject → 정확한 메일박스·별칭 → 캐치올 폴백의 3단계로
평가된다. 이 단계 분리가 `*` 캐치올이 실제 메일박스를 가리는 것을 막는다.

### 어떨 때 쓰는가

| 상황 | 적합도 |
|---|---|
| 내 도메인 메일을 Google Workspace 대신 쓰고 싶다 | 매우 적합 |
| 팀 공용 support@ / hello@ 메일박스가 필요하다 | 매우 적합 |
| 내 서비스에서 프로그래밍으로 메일 발송/수신 | 매우 적합 |
| 메일 데이터 주권 / 프라이버시 규정 대응 | 매우 적합 |
| 대량 마케팅 메일 발송 | 부적합 (SES 등 사용) |

### 나에게 주는 이득

1. **비용 절감** — 수신 무료, 발송만 Workers 유료 월 $5. 인원당 과금 없음.
2. **최신 스택 레퍼런스** — Next.js 16 + Workers + Drizzle + OpenNext의
   실제 프로덕션 예제.
3. **바로 붙일 메일 인프라** — `POST /api/v1/send` 한 번으로 인증메일·문의 알림 처리.
4. **재사용 가능한 부품** — 스팸필터, FTS5 검색, JMAP 서버, TOTP 2FA,
   웹훅 재시도 백오프 등이 각각 독립적으로 떼어 쓸 수 있는 완성품.
5. **사업 시드** — AGPL 하에서 서비스형 수익화 가능 (9장 참고).

---

## 2. 쉬운 설명 (비유 버전)

### "메일 회사 창업 세트"

- 기존 방식(Gmail/네이버): 아파트에 월세 내고 사는 것. 규칙도 집주인이 정하고
  내 편지도 집주인 창고에 보관된다.
- Mailflare: 내 땅에 우체국을 직접 짓는 설계도 + 자재 세트. 편지는 내 창고에
  쌓이고 규칙도 내가 정한다.

### 부품별 비유

| 코드 | 우체국 비유 | 설명 |
|---|---|---|
| Cloudflare Email Routing | 배달 트럭 | 세상의 메일을 내 건물까지 가져온다 |
| `worker.ts email()` | 정문 경비원 | 받을까/거절할까/넘길까 즉석 판단 |
| R2 버킷 | 대형 창고 | 원본 편지·첨부 원본 보관 |
| Queue | 처리 대기 바구니 | 경비원은 뜯지 않고 바구니에만 넣는다 |
| `processInboundMessage` | 분류 직원 | 봉투 뜯어 읽고 서류함에 정리 |
| D1 | 서류 캐비닛 | 누가 언제 무슨 제목으로 보냈는지 목록 |
| 스팸 필터 | 광고 걸러내는 직원 | 100점 중 70점 넘으면 스팸함 |
| Durable Object | "택배 왔어요" 벨 | 새 메일 즉시 브라우저 알림 |
| Cron | 야간 백업 직원 | 매일 새벽 2시 캐비닛 전체 복사 |

### 왜 "경비원이 뜯어보지 않는" 게 중요한가

메일 하나 처리에 3초가 걸린다면, 정문에서 붙잡고 있으면 뒤가 밀린다.
그래서 정문에서는 "창고에 던져놓고 바구니에 메모"만 0.1초에 끝내고,
뒤에서 여유롭게 처리한다. 이것이 **큐 기반 비동기 처리** 패턴이며,
실무에서 매우 자주 쓰인다.

### 정문 판단과 내부 판단의 차이

```
domain 스코프  = 정문 경비원
  "이 사람 편지는 안 받아" (reject)
  "이건 다른 주소로 넘겨" (forward)
  → 배달 트럭이 아직 안 떠났을 때만 가능

mailbox 스코프 = 분류 직원
  "이건 영수증 서류함에" / "이건 스팸함으로"
  → 이미 건물 안에 들어온 편지 정리
```

---

## 3. 설치 및 사용법

### 방법 A: Cloudflare 배포 (권장, 약 5분)

1. 원본 저장소의 **Deploy to Cloudflare** 버튼 클릭
   (https://github.com/hieunc229/mailflare)
2. **앱 이름을 반드시 `mailflare`로 지정** (다른 이름이면 동작하지 않음)
3. `CF_TOKEN` 입력 (배포용 토큰과 별개인 런타임 토큰)

   필요 권한:
   ```
   All accounts → Email Sending:Edit, DNS Settings:Edit,
                  Email Routing Addresses:Edit
   All zones    → DNS Settings:Edit, Email Routing Rules:Edit,
                  Zone Settings:Edit, DNS:Edit
   ```
4. 배포 후 `/setup` 접속 → D1 DB 초기화 + 관리자 계정 생성
5. Admin → Domains 에서 도메인 연결 (같은 CF 계정의 CF DNS 도메인)
   → MX/SPF/DMARC DNS가 자동 설정됨
6. 첫 메일박스 주소 생성 후 테스트 메일 발송

> **이름이 `mailflare`여야 하는 이유:** `wrangler.jsonc`의
> `services[].service`, `CF_EMAIL_WORKER_NAME`, 배포된 Worker 이름 3개가
> 모두 일치해야 한다. Cloudflare 서비스 바인딩은 리터럴 문자열만 받고
> 최상위 `name`을 참조할 수 없다. 메일박스 생성 시 이 이름으로 Email Routing
> 규칙을 만들기 때문에, 하나라도 어긋나면 메일이 들어오지 않는다.

### 방법 B: Docker 셀프호스팅

```bash
git clone https://github.com/hieunc229/mailflare && cd mailflare
cp .env.docker.example .env.docker   # 수신/발송 방식 설정
docker compose up -d --build
# → http://your-host:3000/setup
```

주요 환경변수:

| 변수 | 용도 |
|---|---|
| `DATA_DIR=/data` | SQLite + 첨부 + 백업 저장 경로 |
| `SMTP_INBOUND_PORT=25` | 내장 SMTP 수신 (0이면 비활성) |
| `MAIL_HOSTNAME` | MX가 가리키는 호스트명 / SMTP 배너 |
| `SMTP_URL=smtps://user:pass@host:465` | 발송 릴레이 (SES/Postmark/Mailgun 등) |
| `APP_URL` | 리버스 프록시 뒤 공개 URL |
| `INBOUND_WEBHOOK_SECRET` | CF 릴레이 Worker의 `/api/inbound` 활성화 |
| `TURNSTILE_SECRET_KEY` | 로그인·재설정 폼 봇 방어 |

주의사항:
- 포트 25는 다수 클라우드(AWS/GCP 등)가 기본 차단한다. 막혀 있으면
  `deploy/cloudflare-email-relay` Worker로 MX는 Cloudflare에 두고
  내 서버로 릴레이하면 된다.
- 문서는 `.env.docker.example` 복사를 안내하지만 **해당 파일이 저장소에 없다.**
  `docs/self-hosting.md`의 설정 표를 보고 직접 작성해야 한다.

### 방법 C: 로컬 개발

```bash
cp .dev.vars.example .dev.vars   # CF_TOKEN 입력
npm install
npm run db:migrate:local
npm run dev                      # localhost:3000
npm run db:seed                  # 샘플 데이터 (dev 서버 실행 중일 때)
```

### 기능 목록

**일반 사용자**
- 받은편지함 / 보낸편지함 / 임시보관 / 별표 / 스누즈 / 아카이브 / 스팸 / 휴지통
- 커스텀 폴더, 대화형(스레드) 보기 토글
- Gmail 스타일 검색: `from:maya has:attachment after:2026-09-01 -광고`
- 리치텍스트 작성기 (서명, 인용 접기, 전달 시 첨부 자동 복사)
- 자동응답(부재중), 연락처, 발신자 차단
- 실시간 새 메일 알림, 키보드 단축키, 캘린더
- IMAP 가져오기 / 내보내기

**관리자**
- 도메인 연결/DNS 상태, 메일박스 생성 + 별칭
- 계정 관리, 메일박스 공유 권한 위임
- 라우팅 규칙 (store / forward / reject / 폴더 분류)
- API 키 발급(스코프별), 웹훅 + 배달 로그 + 수동 재시도
- 감사 로그, DB 백업/복원, 브랜딩, 검색 인덱스 재구축

---

## 4. 정체 정리: 플러그인 / 스킬 / MCP?

**결론: 셋 다 아니다. 완전한 독립 웹 애플리케이션이다.**

| | 정체 | 실행 주체 | 예시 |
|---|---|---|---|
| 플러그인 | 다른 앱에 끼우는 확장 | 호스트 앱 | VSCode 확장 |
| 스킬 | AI에게 주는 작업 지침서 | Claude 등 AI | `/code-review` |
| MCP 서버 | AI가 도구로 호출하는 프로토콜 서버 | AI 클라이언트 | GitHub MCP |
| **Mailflare** | **SaaS 급 풀스택 웹앱** | **Workers / Docker** | Gmail, 노션 |

근거:
- `package.json`에 `next`, `react`, `wrangler` → 웹앱 프레임워크
- `src/app/`에 88개 REST 라우트 + 대시보드 UI
- `SKILL.md` / `.claude/skills/` 없음 → 스킬 아님
- `@modelcontextprotocol/sdk` 의존성 없음, MCP 툴 정의 없음 → MCP 아님
- 호스트 앱 확장 매니페스트 없음 → 플러그인 아님

다만 **MCP 서버로 감싸기 매우 쉬운 구조**다. `/api/v1/send`,
`/api/v1/messages`, JMAP 서버, 스코프 기반 API 키가 이미 준비돼 있어
얇은 MCP 래퍼만 작성하면 AI가 메일을 읽고 답장할 수 있다.

참고로 이 포크에는 Claude Code용 컨텍스트 문서 `CLAUDE.md`가 있으나,
이는 스킬이 아니라 프로젝트 안내 문서다.

---

## 5. API 토큰 정리

**결론: Cloudflare 배포 시 필수 / Docker 셀프호스팅 시 선택.**

### 1) `CF_TOKEN` — 필수 (Cloudflare 배포)

Cloudflare 배포 버튼이 쓰는 토큰은 앱에 전달되지 **않는다.** 별도로 만들어
`CF_TOKEN`에 넣어야 한다.

Mailflare는 Cloudflare를 호스팅으로만 쓰지 않고 **런타임 의존성**으로 쓴다:

```
도메인 추가 → 앱이 CF API 호출 → Email Routing 활성화 + DNS 생성
                                  + 발송 서브도메인 설정
메일박스 생성 → CF Email Routing 규칙 생성
                (CF_EMAIL_WORKER_NAME 워커를 타겟으로)
도메인 삭제 → 해당 리소스 정리
```

필요 권한:
```
All accounts → Email Sending:Edit, DNS Settings:Edit,
               Email Routing Addresses:Edit
All zones    → DNS Settings:Edit, Email Routing Rules:Edit,
               Zone Settings:Edit, DNS:Edit
```

- `Email Sending:Edit`는 수신 전용이면 생략 가능.
- 토큰 **시크릿 값만** 입력한다. `Bearer` 접두어와 토큰 ID는 넣지 않는다.
- 레거시 대안으로 `CF_EMAIL` + `CF_API_KEY`(Global API Key)도 지원하지만,
  계정 전체 권한이라 위험하므로 `CF_TOKEN`을 권장한다.

### 2) Mailflare 자체 API 키 — 선택 (외부 연동)

앱 안에서 발급 (Admin → API Keys). Cloudflare와 무관하다.

```
용도: /api/v1/send, /api/v1/messages 호출
      JMAP으로 Thunderbird/Apple Mail 연결 (jmap 스코프)
인증: Authorization: Bearer <key>
스코프: read / send / jmap 등 권한 분리
```

Settings → Account → Email apps 에서 JMAP용 키를 만들 수 있다.

### 3) `GITHUB_UPDATE_TOKEN` — 선택

관리자 화면의 "Update Mailflare" 버튼이 GitHub Actions를 dispatch해
업스트림 최신 코드를 머지한다.

```
GITHUB_UPDATE_TOKEN  → fine-grained PAT (Actions:write, Contents:write)
GITHUB_UPDATE_REPO   → owner/repo
GITHUB_UPDATE_REF    → 브랜치 (생략 시 기본 브랜치)
```

GitHub 저장소 시크릿에 `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`도 필요하다.

- 문서는 `update.yml`이라 표기하지만 실제 파일은
  `.github/workflows/deploy-update.yml`이다. 코드가 기준이다.
- 이 버튼은 Docker 셀프호스팅에서는 비활성화된다 (이미지 pull로 업데이트).

### 4) `TURNSTILE_SECRET_KEY` — 선택

로그인/회원가입 폼에 Cloudflare Turnstile 캡차를 적용한다.
`NEXT_PUBLIC_TURNSTILE_SITE_KEY`와 페어로 사용한다.

### Docker 셀프호스팅의 경우

```
CF_TOKEN 없어도 동작한다.
  → provision.ts가 존을 "manual"로 기록
  → Cloudflare 호출은 전부 no-op
  → DNS 페이지가 직접 설정할 레코드 목록만 표시
  → SMTP_URL로 발송, 내장 SMTP로 수신

CF_TOKEN이 있으면 Workers와 동일하게 자동 프로비저닝된다.
```

### 보안 요약

| 토큰 | 위험도 | 관리 |
|---|---|---|
| `CF_TOKEN` | 높음 (DNS/메일 조작 가능) | Worker Secret, 최소 스코프, 주기적 회전 |
| Mailflare API 키 | 중간 | 스코프 최소화, 앱에서 즉시 폐기 가능 |
| `GITHUB_UPDATE_TOKEN` | 중간 | fine-grained, 해당 저장소만 |
| Global API Key | 매우 높음 | 사용 비권장 |

---

## 6. GitHub에서 유명한 이유

3개월 만에 ⭐3.6k / 🍴499를 달성한 배경 분석.

### 1) "월 $5로 Google Workspace 대체" — 경제성

| 서비스 | 5인 팀 월 비용 | 연간 |
|---|---|---|
| Google Workspace | $30 | $360 |
| Microsoft 365 | $30 | $360 |
| Zoho Mail | $5 | $60 |
| **Mailflare** | **$5 (인원 무관)** | **$60** |

결정적 차이는 **인원당 과금이 아니라는 점**이다. 100명이어도 $5다.

### 2) 셀프호스팅 메일의 고통을 실제로 해결

```
기존 셀프호스팅 메일의 어려움
  - Postfix + Dovecot + SpamAssassin + OpenDKIM 설정
  - 포트 25 차단, IP 평판 관리
  - SPF/DKIM/DMARC 설정
  - 스팸함 직행 문제
  - 서버 유지비 + 보안 패치

Mailflare의 해결
  - Cloudflare Email Routing이 MX·수신 담당 (평판 부담 감소)
  - DNS를 앱이 자동 설정
  - 서버 관리 불필요 (서버리스)
  - 버튼 하나로 배포
```

### 3) 원클릭 배포

README 최상단의 "Deploy to Cloudflare" 버튼 하나로 5분 내 내 도메인 메일함이
생긴다. 이 즉각적 보상이 스타 → 실사용 → 입소문 루프를 만든다.
499개 포크가 그 증거다 (배포 시 자기 저장소로 포크되기 때문).

### 4) 기능 완성도가 데모 수준이 아님

```
스레드(대화형) 보기      FTS5 전문검색 (Gmail 문법)
자체 스팸필터 (베이지안)  2FA (TOTP) + 복구코드
웹훅 + 재시도 백오프      감사 로그
메일박스 공유·권한 위임    자동 DB 백업/복원
JMAP 서버 (표준)         IMAP 마이그레이션
첨부/서명/자동응답/스누즈  실시간 WebSocket 알림
캘린더                   API 키 스코프
```

특히 **JMAP 서버를 직접 구현한 점**이 메일 매니아층을 끌었다.

### 5) 최신 스택 레퍼런스 가치

`Next.js 16 App Router + Cloudflare Workers(OpenNext) + D1 + R2 + Queues +
Durable Objects + Drizzle` 조합의 제대로 된 프로덕션 예제가 드물다.
메일을 쓰지 않을 사람도 코드를 보려고 스타를 누른다.

### 6) 타이밍과 트렌드

```
2026년 개발자 커뮤니티 기류
  → 프라이버시/데이터 주권 관심 증가
  → 빅테크 SaaS 비용 피로감
  → r/selfhosted, Hacker News 셀프호스팅 붐
  → Cloudflare 무료 티어 생태계 확장
  → 이 교차점에 정확히 착지
```

### 7) AGPL-3.0 + 오픈코어 모델

완전 오픈소스로 신뢰도를 확보하면서, Pro/Team 라이선스로 브랜딩 기능을 파는
오픈코어 구조다. 지속 개발 신호를 주어 "곧 중단될 프로젝트"라는 우려를 줄인다.
`@hieuSSR`의 X 홍보와 스폰서(Sequenzy) 노출도 유입에 기여했다.

### 8) 단일 개발자의 개발 속도

3개월 / 115커밋 / 38,700줄 / 13명 기여자, 주 개발자 81커밋.
"살아있는 프로젝트"라는 시그널이 강하다.

### 종합

```
(가격 경쟁력) × (진입장벽 0) × (실사용 완성도)
  × (학습 가치) × (프라이버시 트렌드) × (개발 속도)
= 3개월 만에 ⭐3.6k
```

---

## 7. 로컬 AI 에이전트 구축 활용성

**결론: 매우 유용하다. 특히 "에이전트의 입과 귀"로써 가치가 높다.**

에이전트 개발에서 가장 어려운 부분이 외부 세계와 소통하는 채널이고,
이메일은 그 중 가장 범용적인 채널이다. Mailflare는 그것을 통째로 제공한다.

### 직접적으로 도움 되는 것

#### 1) 에이전트 전용 메일 주소 + 프로그래밍 제어

```
agent@내도메인.com 생성 후
  수신: /api/v1/messages?q=is:unread  (폴링)
        또는 웹훅 (푸시, 권장)
  발송: POST /api/v1/send
  검색: FTS5 기반 Gmail 문법
  인증: API 키 스코프 (read/send 분리)
```

#### 2) 웹훅 = 에이전트 트리거

```
메일 도착
  → dispatchWebhooks()가 내 에이전트 엔드포인트로 POST
  → 에이전트가 처리 후 답장 발송

재시도 백오프 + 배달 로그 + 수동 재시도가 이미 구현되어 있어,
에이전트가 다운되어도 메일이 유실되지 않는다.
```

#### 3) MCP 서버로 감싸기 (가장 추천)

```javascript
// 개념 스케치
server.tool("search_email", { query: z.string() }, async ({ query }) => {
  const r = await fetch(`${BASE}/api/v1/messages?q=${encodeURIComponent(query)}`,
    { headers: { Authorization: `Bearer ${KEY}` } });
  return { content: [{ type: "text", text: JSON.stringify(await r.json()) }] };
});

server.tool("send_email",
  { to: z.string(), subject: z.string(), text: z.string() },
  async (args) => {
    await fetch(`${BASE}/api/v1/send`, {
      method: "POST",
      headers: { Authorization: `Bearer ${KEY}`, "Content-Type": "application/json" },
      body: JSON.stringify({ from: "agent@내도메인.com", ...args }),
    });
    return { content: [{ type: "text", text: "sent" }] };
  });
```

Claude Desktop / Claude Code에 연결하면 "내 메일을 읽고 답장하는 AI"가 된다.

#### 4) JMAP 표준 = 범용 커넥터

JMAP은 IMAP의 현대판 표준이다. 서버가 이미 구현돼 있어 JMAP 클라이언트
라이브러리를 쓰는 어떤 도구든 바로 붙는다. 자체 프로토콜을 만들 필요가 없다.

#### 5) 로컬 100% 실행 (프라이버시)

```
Docker 모드 = 내 서버에서 전부 동작
  SQLite + 로컬 파일 + 내장 SMTP
  → 메일 데이터가 외부로 나가지 않는다
  → 로컬 LLM(Ollama 등)과 조합하면 완전 오프라인 AI 메일 비서
```

#### 6) 아키텍처 학습 가치

| 패턴 | 참고 위치 |
|---|---|
| 큐 기반 비동기 워커 | `worker.ts queue()` + `src/lib/email/inbound.ts` |
| 재시도 + 지수 백오프 | `src/lib/email/webhooks.ts` |
| 런타임 추상화 | `getEnv()` + `server/runtime/env.ts` |
| 스코프 기반 API 인증 | `src/lib/api/key-auth.ts` |
| 실시간 푸시 (WebSocket 허브) | `src/lib/realtime/hub.ts` |
| 프레임워크 독립 핸들러 | `src/lib/jmap/handler.ts` |
| 베이지안 분류기 | `src/lib/spam/` |
| FTS5 전문검색 | 마이그레이션 0030 + `src/lib/search/conditions.ts` |

특히 `src/lib/jmap/handler.ts`는 Next.js에 의존하지 않도록 설계되어,
같은 로직을 Worker/Express/Lambda 어디든 마운트할 수 있다. 에이전트 설계 시
그대로 차용할 가치가 있는 패턴이다.

### 한계

| 한계 | 설명 | 우회 |
|---|---|---|
| MCP 내장 없음 | 직접 래퍼 작성 필요 | 위 스케치 수준으로 충분 |
| LLM 기능 없음 | AI 요약/분류 미구현 | 직접 추가 (차별화 기회) |
| 테스트 러너 없음 | `.mjs` 스크립트 2개뿐 | 직접 테스트 작성 |
| 타입 체크 느슨 | `ignoreBuildErrors: true`, `noImplicitAny: false` | `npx tsc --noEmit` 실행 |
| 스키마 이중 관리 | 마이그레이션 추가 시 `src/lib/setup/migration.ts`의 `INITIAL_SCHEMA_SQL`·`MIGRATION_NAMES`도 함께 수정 | 누락 시 신규 설치가 깨진다 |

### 추천 조합

```
[Mailflare (메일 I/O 레이어)]
        ↓ 웹훅 트리거
[내 에이전트 (Node/Python)]
        ↓
[Claude API + MCP 도구들]
        ↓
[액션: 답장 발송 / 폴더 이동 / 캘린더 등록 / Slack 알림]
```

Mailflare는 "이메일 I/O 레이어"로 쓰고, 지능은 위에 얹는 구조가 최적이다.

---

## 8. React / PHP 구현 가능성

### React: 이미 React다

```
현재 스택: Next.js 16 + React 19 + TypeScript
  src/components/ 전부 React 컴포넌트
  src/app/ App Router (React Server Components)
```

React를 알면 바로 수정할 수 있다. 다만 주의점:

- Next.js 16 App Router (Pages Router 아님)
- `next dev`가 `node_modules/next/dist/docs/`에 가이드를 생성하므로 참고
- 탭 인덴트, `@/*` → `src/*` 경로 별칭
- 타입/헬퍼는 `*-types.d.ts` / `*-utils.ts` 형제 파일로 분리 (컨벤션)
- `DialogContent`에 max-height가 없어 긴 폼은
  `max-h-[calc(100vh-4rem)] overflow-y-auto`를 추가해야 한다

### PHP: 가능하지만 아키텍처를 다시 설계해야 한다

핵심 문제는 PHP가 Cloudflare Workers에서 동작하지 않는다는 점이다.

| Mailflare 기능 | Workers | PHP 대체 |
|---|---|---|
| 메일 수신 | Email Routing → `email()` | Postfix + 파이프 스크립트, 또는 CF 릴레이 → PHP 웹훅 |
| 비동기 큐 | CF Queues | Redis + 워커 데몬 / Laravel Horizon |
| 실시간 알림 | Durable Objects | Pusher / Soketi / ReactPHP |
| DB | D1 | MySQL / PostgreSQL |
| 파일 저장 | R2 | S3 / MinIO / 로컬 |
| 스케줄 | CF Cron | 시스템 crontab |
| MIME 파싱 | postal-mime | `php-mime-mail-parser` |
| 발송 | `send_email` 바인딩 | PHPMailer / Symfony Mailer |
| 런타임 | 서버리스 (관리 0) | 서버 필요 (관리 O) |

### 권장 판단

| 시나리오 | 추천 | 이유 |
|---|---|---|
| Mailflare 커스터마이징/개선 | 현재 스택 유지 | 38,700줄이 이미 완성 |
| PHP 사이트에 메일 기능 붙이기 | API로 연동 | `/api/v1/send` 호출로 끝 |
| 완전히 새로 만들고 싶다 | Next.js/Workers 유지 | 서버리스 유지비·관리 이점 |
| 이미 Laravel 인프라 보유 | Laravel 재구현 | 팀 역량이 곧 생산성 |
| PHP로 포팅 (학습 목적) | 비권장 | 노력 대비 이득 적음 |

PHP 연동 예시 (가장 현실적):

```php
<?php
// PHP 사이트의 문의 폼 → Mailflare로 발송
$ch = curl_init('https://mailflare.example.com/api/v1/send');
curl_setopt_array($ch, [
  CURLOPT_POST => true,
  CURLOPT_HTTPHEADER => [
    'Authorization: Bearer ' . getenv('MAILFLARE_API_KEY'),
    'Content-Type: application/json',
  ],
  CURLOPT_POSTFIELDS => json_encode([
    'from'    => 'noreply@example.com',
    'to'      => $_POST['email'],
    'subject' => '문의 접수 완료',
    'text'    => '감사합니다. 곧 답변드리겠습니다.',
  ]),
  CURLOPT_RETURNTRANSFER => true,
]);
$res = curl_exec($ch);
```

PHP로 다시 만들 필요 없이, PHP에서 갖다 쓰는 것이 합리적이다.

---

## 9. 수익화 아이디어

### 법적 전제: AGPL-3.0

```
AGPL-3.0 Section 13 (네트워크 조항)
  "수정한 소프트웨어를 네트워크로 서비스하면,
   그 수정된 소스코드를 사용자에게 제공해야 한다"
```

| 하려는 일 | 가능? | 조건 |
|---|---|---|
| 사내 자체 사용 | 가능 | 조건 없음 |
| 수정 없이 SaaS 운영 | 가능 | 원본 소스 링크 제공 |
| 수정해서 SaaS 운영 | 주의 | 수정 코드도 AGPL로 공개 |
| 설치/운영 **서비스** 판매 | 가능 | 코드 공개 의무 없음 |
| 별도 앱으로 API만 호출 | 가능 | 내 앱은 AGPL 아님 (프로세스 분리) |
| 소스 비공개 SaaS | 불가 | 위반 |

두 가지 전략:

```
전략 A: "서비스를 판다" — 설치·운영·지원·컨설팅으로 수익
전략 B: "별도 프로세스로 분리" — Mailflare는 그대로 두고
         내 앱이 API/웹훅으로만 통신 (가장 안전하고 유연)
```

> 상업화 전 변호사 검토를 권한다. "프로세스 분리"의 경계는 케이스별 해석 차이가
> 있다. 규모 있는 사업이면 원작자(`@hieuSSR`)에게 상업용 듀얼 라이선스를
> 문의하는 것이 가장 깔끔하다.

### 아이디어 1: 관리형 Mailflare 호스팅 (최우선 추천)

컨셉: 설치·운영이 부담스러운 사람을 대신한다.

```
AGPL 완전 안전 (서비스 판매)
인프라 비용은 고객 부담 또는 마진 적용
원클릭 배포가 있어도 CF_TOKEN 권한 설정, DNS, 포트 25,
  업데이트, 백업 확인은 비개발자에게 장벽
이미 ⭐3.6k = 수요 검증됨
```

| 플랜 | 가격 | 내용 |
|---|---|---|
| Setup | ₩150,000 (1회) | 배포 + 도메인 연결 + DNS + 첫 메일박스 |
| Managed | ₩29,000/월 | 위 + 모니터링 + 업데이트 + 백업 확인 + 이메일 지원 |
| Managed Pro | ₩79,000/월 | 위 + 무제한 메일박스 + 우선 대응 + 월간 리포트 |
| Full Hosted | ₩49,000/월 | 내 계정에서 운영 (고객은 로그인만) |

고객 관점: Google Workspace 10명 = 월 ₩84,000 vs Managed Pro ₩79,000
(인원 무제한) → 20명 이상이면 압도적이며, 데이터 소유권도 확보.

시작 순서:
```
1주차: 내 도메인에 직접 설치·운영 경험 축적
2주차: 설치 자동화 스크립트 (Terraform / wrangler CLI)
3주차: 랜딩페이지 + 가격표
4주차: 무료 베타 3팀 → 후기 확보
       (r/selfhosted, 개발자 커뮤니티, 원본 저장소 Discussions)
2개월차: 유료 전환, 자동화 개선
```

핵심 무기: `CF_TOKEN` 권한 7개 설정이 가장 어려운 단계다. 이를 가이드
위저드로 만들면 그 자체가 상품이 된다.

### 아이디어 2: AI 메일 에이전트 레이어 (차별화 최강)

컨셉: Mailflare는 메일 인프라, 그 위에 AI 지능을 판다.

```
AGPL 회피 — 완전 별개 프로세스
   [Mailflare] ←웹훅/API→ [내 AI 서비스 (비공개 가능)]
Mailflare에 AI 기능이 전무 = 순수한 빈 공간
필요한 훅이 이미 존재: 웹훅(트리거) + API(읽기/쓰기) + JMAP(표준)
AI 기능은 프리미엄 가격이 정당화됨
```

제품 구조:

```
┌─────────────────────────────────────┐
│ Mailflare (고객 소유, 그대로 사용)      │
└──────────────┬──────────────────────┘
               │ 새 메일 웹훅
               ▼
┌─────────────────────────────────────┐
│ 내 서비스: 메일 비서                    │
│  · 3줄 요약 + 우선순위 점수             │
│  · 자동 분류 (인보이스/계약/문의/광고)    │
│  · 답장 초안 생성 (내 톤 학습)           │
│  · "다음 주 화요일" → 캘린더 이벤트       │
│  · 첨부 인보이스 → 금액·기한 추출         │
│  · 중요 메일만 Slack/카톡 알림           │
│  · 주간 리포트                         │
└──────────────┬──────────────────────┘
               │ /api/v1/send, 폴더 이동, 캘린더 API
               ▼
        Mailflare에 결과 반영
```

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | ₩0 | 월 50통 요약 |
| Personal | ₩9,900/월 | 무제한 요약 + 분류 + 답장 초안 |
| Pro | ₩29,000/월 | 위 + 인보이스 추출 + 캘린더 + 커스텀 규칙 |
| Team | ₩79,000/월 | 위 + 공용함 자동 배분 + 팀 리포트 + MCP 서버 |

킬러 기능: **MCP 서버 제공.** "Claude Desktop에 연결하면 Claude가 직접 메일을
읽고 답장한다"는 셀링포인트는 2026년 기준 경쟁자가 거의 없다.

기술 난이도는 낮다:
```
웹훅 리스너 (Hono/Express)  → 1일
Claude API 호출 + 프롬프트   → 2일
Mailflare API 연동          → 1일
간단한 대시보드              → 3일
→ MVP 약 1주
```

### 아이디어 3: 수직 특화 패키지

범용 메일 대신 특정 업종 전용 메일 + 워크플로우로 판다.

**A. 프리랜서/1인 사업자 — ₩19,000/월**
```
hello@내이름.com (프로 이미지)
  + 견적 요청 메일 자동 감지 → 템플릿 답장
  + 인보이스 메일 추적 (미수금 알림)
  + 클라이언트별 자동 폴더 분류
  + 계약서 첨부 자동 보관
```

**B. 부동산 중개 — ₩49,000/월**
```
매물 문의 메일 → 자동 파싱 → CRM 등록
  + 관심 지역/예산 추출
  + 매칭 매물 자동 답장
  + 계약 단계별 알림
```

**C. 병원/클리닉 — ₩59,000/월**
```
예약 문의 → 캘린더 연동 자동 확인 메일
  + 진료 리마인더
  + 데이터가 내 서버에 있음 = 의료정보 규정 대응
```

**D. 온라인 셀러 — ₩39,000/월**
```
주문/반품 문의 자동 분류 + 우선순위
  + 반복 문의 자동 답장
  + 매출/문의 통계 리포트
```

수직이 강한 이유: 범용 "메일함"은 경쟁자가 많고 가격 경쟁에 빠진다.
"부동산 중개 메일 자동화"는 경쟁자가 거의 없고 가격 결정력이 높다.
같은 기능이어도 2~3배 가격이 가능하다 (고객이 "내 문제"로 인식).

### 아이디어 4: 개발자 대상 인프라 판매

**4-A. Email API for Indie Devs**
```
컨셉: Resend/Postmark보다 저렴하고, 수신도 되는 메일 API
  Resend: 발송만, 3,000통/월 무료, 이후 $20/월
  내 것:  발송 + 수신 + 저장 + 검색, ₩9,900/월 무제한급
차별점: 수신 메일을 웹훅으로 받고 검색까지 가능
사용 사례: SaaS 알림메일, 문의폼, 이메일 파싱 자동화
```

**4-B. AI 에이전트용 이메일 인프라 (미래성 최고)**
```
컨셉: AI 에이전트에게 이메일 주소를 발급한다
  POST /agents → agent-abc123@agents.내서비스.com
  웹훅으로 수신, API로 발송, 에이전트당 격리 메일박스
  MCP 서버 기본 제공
타겟: AI 에이전트 개발자/스타트업
가격: 에이전트당 ₩3,000/월 또는 통당 과금
근거: 2026년 AI 에이전트 붐 + 경쟁자 희소
```

**4-C. 화이트라벨 (라이선스 주의)**
```
컨셉: 웹호스팅사·MSP가 자사 브랜드로 메일 서비스 제공
주의: Mailflare 자체에 브랜딩 기능이 라이선스로 게이트됨
      (src/lib/licenses/, Paymug 검증)
      → 원작자와 리셀러/듀얼라이선스 협의 필수
수익: MSP당 ₩500,000/월 + 고객당 수수료
```

### 아이디어 5: 콘텐츠·교육 (저위험 시작점)

```
1) 유료 강의 "Cloudflare Workers로 메일 서비스 만들기"
   ₩55,000 × 200명 = 약 ₩11,000,000
2) 유료 뉴스레터 "서버리스 셀프호스팅"
   ₩5,000/월 × 200명 = ₩1,000,000/월
3) 한국어 완전 설치 가이드 (블로그/유튜브)
   → 광고 + 제휴 + 위 서비스 유입 퍼널
   ※ 한국어 자료가 거의 없어 선점 기회
4) 컨설팅 "기업 메일 이전 프로젝트"
   건당 ₩2,000,000~5,000,000
```

### 아이디어 6: 상위 레이어 제품

**6-A. 팀 공용 메일함 → Helpdesk**
```
Mailflare 기존 기능: 메일박스 공유, 권한 위임
추가할 것: 티켓 상태, 담당자 배정, SLA, 매크로, 만족도 조사
경쟁: Zendesk $55/에이전트/월, Front $19/사용자/월
내 것: ₩39,000/월 (인원 무제한)
```

**6-B. 메일 기반 CRM**
```
메일 스레드 = 고객 타임라인
  + 딜 파이프라인 + 자동 후속 알림 + AI 리드 스코어링
경쟁: Streak, Mixmax (Gmail 종속)
차별: 내 도메인 + 내 데이터 소유
```

### 추천 로드맵

```
Phase 1 (0~2개월): 리스크 0 검증
  · 내 도메인에 직접 설치·운영
  · 한국어 설치 가이드 블로그/유튜브 (SEO 선점)
  · 유입 트래픽으로 수요 측정
  예상: ₩0~500,000/월 · 주 5시간

Phase 2 (2~5개월): 서비스화
  · 설치 대행 (₩150,000/건), Managed 플랜 (₩29,000/월)
  · 베타 고객 5~10팀
  예상: ₩1,000,000~3,000,000/월 · 주 15시간

Phase 3 (5~10개월): AI 레이어 (승부처)
  · AI 메일 비서 MVP (웹훅 + Claude API)
  · MCP 서버 출시 → Claude 사용자층 공략
  · Product Hunt / GitHub / 개발자 커뮤니티 런치
  예상: ₩3,000,000~10,000,000/월 · 풀타임 전환 검토

Phase 4 (10개월+): 수직 특화 또는 인프라
  · 데이터로 확인된 최고 반응 업종 1개 집중
  · 또는 "AI 에이전트용 이메일 인프라"로 피벗
  예상: ₩10,000,000+/월
```

### 리스크 대응

| 리스크 | 대응 |
|---|---|
| AGPL 위반 | 프로세스 분리(전략 B) + 변호사 검토 + 원작자 협의 |
| 업스트림 변경 | 포크 고정 + API 계층만 의존 |
| Cloudflare 정책 변경 | Docker 셀프호스팅 모드가 이미 존재 (헷지 완료) |
| 원작자의 직접 SaaS 출시 | AI 레이어·수직 특화로 차별화 |
| 메일 발송 평판 문제 | CF Email Sending 또는 SES 사용, 대량발송 금지 |

### 하나만 고른다면

**"AI 메일 에이전트 + MCP 서버" (아이디어 2)**

```
AGPL 리스크 최소 (완전 별개 프로세스)
기술 난이도 낮음 (MVP 약 1주)
차별화 최대 (Mailflare에 AI 전무)
시장 타이밍 적절 (2026 AI 에이전트 붐)
확장성 좋음 (다른 메일 서비스로도 확장 가능)
```

---

## 10. 핵심 주의사항 (Gotchas)

이 저장소를 실제로 만지기 전에 반드시 알아야 할 것들.

### 1) Worker 이름은 반드시 `mailflare`

`wrangler.jsonc`의 `services[].service`, `CF_EMAIL_WORKER_NAME`,
배포된 Worker 이름이 모두 일치해야 한다. Cloudflare 서비스 바인딩은
리터럴 문자열만 받으며 최상위 `name`을 참조할 수 없다.

### 2) 스키마 이중 관리

`src/lib/setup/migration.ts`가 전체 스키마를 inline SQL로 중복 보유한다.
마이그레이션을 추가할 때는:

```bash
npm run db:generate
# 그리고 src/lib/setup/migration.ts 의
#   INITIAL_SCHEMA_SQL  ← 새 DDL 추가
#   MIGRATION_NAMES     ← 새 파일명 추가
```

`MIGRATION_NAMES`에서 이름을 빼면 해당 마이그레이션은 이후 Wrangler apply
대상으로 남는다 (`0013_add_license_settings.sql`이 현재 그렇게 처리됨).

### 3) `opennextjs-cloudflare deploy`로 배포하지 말 것

`worker.ts`가 진입점이므로, OpenNext로 빌드하고 Wrangler로 업로드해야 한다.
`npm run deploy` 스크립트가 그 순서를 지킨다.

### 4) 타입 체크가 빌드에서 잡히지 않음

`next.config.ts`의 `typescript.ignoreBuildErrors: true`,
`tsconfig.json`의 `noImplicitAny: false` 때문이다.
실제 타입 체크는 `npx tsc --noEmit`으로 직접 실행한다.

### 5) 테스트 러너 없음

`tests/`에 `.mjs` 스크립트 2개가 있을 뿐, 테스트 프레임워크는 설정되지 않았다.

### 6) `requireUser`는 throw 한다

Next가 이를 500으로 노출한다. 401을 돌려주려면
`src/lib/api/auth.ts`의 `requireSessionUser`를 쓴다.
(기존 라우트 다수가 아직 `requireUser`를 쓰고 있다.)

### 7) 메시지 쿼리는 `userId`가 아니라 접근 가능 메일박스 ID로 스코프

`src/lib/mailboxes/access.ts`의 `listAccessibleMailboxIds`를 사용한다.
표준 패턴은 `src/app/api/messages/route.ts` 참고.

### 8) `DialogContent`에 max-height가 없다

필드가 많은 다이얼로그는 뷰포트를 넘쳐 제출 버튼에 도달할 수 없게 된다.
`max-h-[calc(100vh-4rem)] overflow-y-auto`를 추가한다.

### 9) `.env.docker.example`이 저장소에 없다

`docs/self-hosting.md`는 이 파일 복사를 안내하지만 실제로는 존재하지 않는다.
같은 문서의 설정 변수 표를 보고 직접 작성해야 한다.

### 10) 문서와 코드의 파일명 불일치

README/문서는 업데이트 워크플로를 `update.yml`이라 부르지만 실제 파일은
`.github/workflows/deploy-update.yml`이다. 코드가 기준이다.

### 11) `wrangler d1 export`가 동작하지 않는다

FTS5 가상 테이블(`messages_fts`)이 있는 DB에서는 실패한다.
앱 자체의 JSON 백업 기능은 영향받지 않는다.

### 12) 컨벤션

- 탭 인덴트
- `@/*` → `src/*`
- 타입은 `*-types.d.ts`, 순수 헬퍼는 `*-utils.ts` 형제 파일로 분리
- 바인딩 접근은 `getEnv()` / `getEnvAsync()` → `getDb(env)`
  (`getCloudflareContext` 직접 import 금지)
- API 실패 응답은 `NextResponse.json({ error: "..." }, { status })`
- `cloudflare-env.d.ts`는 생성 파일 (500KB) — `npm run cf-typegen`으로 재생성,
  직접 수정 금지

---

## 참고 링크

- 원본 저장소: https://github.com/hieunc229/mailflare
- 이 포크: https://github.com/bmshin94/mailflare
- 공식 사이트: https://mailflare.co/
- 원클릭 배포: https://deploy.workers.cloudflare.com/?url=https://github.com/hieunc229/mailflare
- DeepWiki 문서: https://deepwiki.com/hieunc229/mailflare
- 배포 가이드: [docs/deployment.md](./deployment.md)
- API 문서: [docs/api.md](./api.md)
- 셀프호스팅: [docs/self-hosting.md](./self-hosting.md)
- 스팸 보호: [docs/spam-protection.md](./spam-protection.md)
- 트러블슈팅: [docs/troubleshooting.md](./troubleshooting.md)
- Cloudflare API 토큰 권한 참고:
  https://github.com/hieunc229/mailflare/issues/24#issuecomment-5523686105
- AGPL-3.0 전문: [LICENSE](../LICENSE)
