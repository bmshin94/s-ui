# S-UI 전수조사 분석 리포트 (한국어)

> 이 문서는 S-UI 저장소 전체를 코드 레벨까지 훑어보고 정리한 분석 노트입니다.
> 대상 버전: **v1.6.3** (`config/version`)

---

## 📎 관련 GitHub 주소

| 구분 | 주소 |
|------|------|
| 원본 저장소 (Upstream) | https://github.com/alireza0/s-ui |
| 이 저장소 (Fork) | https://github.com/bmshin94/s-ui |
| 프론트엔드 (Git Submodule) | https://github.com/alireza0/s-ui-frontend |
| 프록시 코어 (sing-box) | https://github.com/SagerNet/sing-box |
| 공식 문서 (Wiki) | https://github.com/alireza0/s-ui/wiki |
| API 문서 | https://github.com/alireza0/s-ui/wiki/API-Documentation |
| 구독 서비스 문서 | https://github.com/alireza0/s-ui/wiki/Subscription-Service |
| 설정 객체 문서 | https://github.com/alireza0/s-ui/wiki/Configuration-Objects |
| 설정값 레퍼런스 | https://github.com/alireza0/s-ui/wiki/Settings-Reference |
| Docker Hub 이미지 | https://hub.docker.com/r/alireza7/s-ui |
| 릴리스 | https://github.com/alireza0/s-ui/releases/latest |

### 서드파티 생태계
- https://github.com/itning/reset-s-ui-traffic — 전체 사용자 주기적 트래픽 리셋
- https://github.com/zqh2333/s-ui-traffic-reset — 트래픽 리셋 도구
- https://github.com/Sownix21/SUI-Bot — 텔레그램 봇

---

## 1. 한 줄 요약

**S-UI** 는 SagerNet 의 프록시 엔진 `sing-box` 를 **웹 브라우저에서 GUI 로 관리**할 수 있게 해 주는
관리자 패널(Admin Panel) 입니다. 거대한 JSON 설정 파일을 손으로 편집하는 대신,
클릭 몇 번으로 인바운드/아웃바운드/사용자/라우팅을 구성하고 코어를 재시작할 수 있습니다.

- **언어/스택**: Go 1.26 (백엔드) + Vue 3 (프론트엔드, 별도 저장소 서브모듈)
- **웹 프레임워크**: Gin
- **ORM/DB**: GORM + SQLite (WAL 모드)
- **라이선스**: GPL v3
- **지원 플랫폼**: Linux(amd64/arm64/armv7/armv6/armv5/386/s390x), Windows(amd64/386/arm64), macOS(실험적)
- **지원 언어**: 영어, 페르시아어, 베트남어, 중국어 간체/번체, 러시아어

---

## 2. 폴더 전수조사

| 경로 | 역할 |
|------|------|
| `main.go`, `app/app.go` | 진입점. DB 초기화 → 크론/웹/구독 서버/코어 순차 기동. SIGHUP=재시작, SIGTERM=종료 |
| `core/` | sing-box 를 **Go 라이브러리로 임베드**해 같은 프로세스에서 구동. `core/protocol/` 밑에 프로토콜별 유저 관리 |
| `web/` | 패널 웹서버(기본 2095). `//go:embed *` 로 프론트엔드 빌드 산출물을 바이너리에 내장 |
| `api/` | REST API 2종. `/api`(세션 쿠키·웹 UI용), `/apiv2`(토큰 인증·외부 연동용) |
| `sub/` | 구독 서버(기본 2096). 링크/sing-box JSON/Clash YAML 3가지 포맷 제공 |
| `service/` | 비즈니스 로직 본체. inbounds/outbounds/clients/tls/stats/setting CRUD, DB→sing-box JSON 변환 |
| `database/` | SQLite + GORM. 스키마, 마이그레이션, 백업 |
| `cronjob/` | 백그라운드 작업 7종 (아래 상세) |
| `cmd/` | CLI 명령어: `admin`, `setting`, `backup`, `migrate`, `healthcheck`, `uri` |
| `middleware/` | CSRF 방어, 도메인 검증 |
| `network/` | HTTP/HTTPS 자동 감지 리스너 (한 포트로 둘 다 처리) |
| `util/` | 링크↔JSON 변환, 인증서, 패스워드, base64 유틸 |
| `windows/` | 윈도우 설치/제거/빌드 배치·PowerShell 스크립트 |
| `install.sh` (31KB) | 원클릭 설치 스크립트 (OS/아키 감지, systemd/OpenRC 등록) |
| `s-ui.sh` (31KB) | 서버에서 쓰는 대화형 관리 메뉴 |
| `frontend/` | **Git 서브모듈** — 초기화 전에는 비어 있음 |

> ⚠️ 소스 빌드 전에 반드시: `git submodule update --init --recursive`

---

## 3. 데이터 모델 (`database/model/model.go`)

```
User      관리자 계정 (기본 admin/admin)
Client    실제 사용자 — 이름, 트래픽 한도, 만료일, 사용량(up/down)
Inbounds  들어오는 연결 설정
Outbounds 나가는 연결 설정
Endpoints WireGuard/Tailscale 등 엔드포인트
Tls       SSL 인증서 관리
Stats     트래픽 통계 (시간 버킷별)
Changes   감사 로그 — 누가/언제/무엇을 변경했는지
Tokens    API 토큰 (만료시간 포함)
Setting   key-value 설정 저장소
```

`Client` 구조체에서 주목할 필드:

```go
Volume     int64  // 트래픽 한도(바이트)
Expiry     int64  // 만료일(유닉스 타임)
AutoReset  bool   // 자동 리셋 여부
ResetDays  int    // 리셋 주기(일)
DelayStart bool   // 첫 접속 시점부터 카운트 시작
TotalUp/TotalDown int64 // 누적 사용량
```

→ "월 100GB / 30일 상품" 같은 **구독 상품 모델**을 그대로 구현할 수 있는 설계입니다.

---

## 4. 백그라운드 작업 (`cronjob/cronJob.go`)

| 주기 | 잡 | 역할 |
|------|-----|------|
| `@every 10s` | `statsJob` | 코어에서 트래픽 수집 → DB 저장 |
| `@every 1m` | `depleteJob` | 한도 초과/만료 사용자 자동 차단 |
| `@every 5s` | `checkCoreJob` | 코어 헬스체크 & 자동 재시작 (워치독) |
| `@every 10m` | `WALCheckpointJob` | SQLite WAL 체크포인트 |
| `@daily` | `delStatsJob` | 오래된 통계 정리 |
| 설정값 | `resetTrafficJob` | 전체 트래픽 주기적 초기화 |

크론 체인에 `cron.Recover`(패닉 복구) + `cron.SkipIfStillRunning`(중복 실행 방지)을 걸어
잡이 겹쳐 돌며 카운터를 중복 차감하는 문제를 막습니다.

---

## 5. 코드 품질에서 눈에 띈 점

- **타이밍 공격 방어**: 토큰 비교에 `subtle.ConstantTimeCompare` 사용
- **프록시 스푸핑 방어**: `SetTrustedProxies` 로 루프백/사설망만 `X-Forwarded-*` 신뢰
- **동시성**: 토큰 슬라이스를 `sync.RWMutex` 로 보호, 락 순서를 주석에 명시
- **DB 권한**: DB 디렉토리를 `0700` 으로 강제 `chmod`
- **SQLite 튜닝**: `_busy_timeout=10000&_journal_mode=WAL&_cache_size=-200&_txlock=immediate`
  (`_txlock=immediate` 로 "database is locked" 이슈 #1209 해결)
- **토큰 마스킹**: 목록 조회 시 `Select("id,desc,'****' as token,...")`
- 주석이 "왜 이렇게 했는지"를 이슈 번호까지 달아 설명 → 기여자 친화적
- `*_test.go` 25개 이상 + GitHub Actions (gofmt 검사 + 테스트 + 멀티아키 릴리스)

---

## 6. 설치 및 사용법

### 리눅스 원클릭
```sh
bash <(curl -Ls https://raw.githubusercontent.com/alireza0/s-ui/master/install.sh)
```
설치 스크립트 언어 선택: `SUI_LANG=en|fa|ru|vi|zhcn|zhtw`
Alpine 은 `apk add bash` 먼저 실행 (OpenRC 자동 감지).

### Docker
```sh
mkdir s-ui && cd s-ui
wget -q https://raw.githubusercontent.com/alireza0/s-ui/master/docker-compose.yml
docker compose up -d
```

### 소스 빌드 (개발용)
```sh
git clone https://github.com/alireza0/s-ui
cd s-ui
git submodule update --init --recursive   # 필수
./runSUI.sh
# 또는 수동
rm -fr web/html/* && cp -R frontend/dist/ web/html/
go build -o sui main.go && ./sui
```

### 기본 접속 정보
```
패널: http://<서버IP>:2095/app/
구독: http://<서버IP>:2096/sub/
계정: admin / admin   ← 설치 직후 반드시 변경
```

### CLI
```sh
./sui admin        # 관리자 계정/비밀번호 변경 (비번 분실 시 복구)
./sui setting      # 포트/경로 등 설정 변경
./sui backup       # DB 백업
./sui migrate      # 스키마 마이그레이션
./sui healthcheck  # 헬스체크
./sui uri          # 접속 URI 출력
```

### 환경변수
| 변수 | 기본값 | 설명 |
|------|--------|------|
| `SUI_LOG_LEVEL` | `info` | debug/info/warn/error |
| `SUI_DEBUG` | `false` | gin 디버그 모드 |
| `SUI_DB_FOLDER` | `db` | DB 저장 경로 |
| `SUI_BIN_FOLDER` | `bin` | 바이너리 경로 |
| `SINGBOX_API` | - | 외부 sing-box API 주소 |

---

## 7. 자주 나온 질문 정리

### Q. 플러그인인가요? 스킬인가요? MCP 인가요?
**셋 다 아닙니다.** `SKILL.md`, `.mcp.json`, 플러그인 매니페스트 모두 없고
`main.go` + `go.mod` 로 구성된 **독립 실행형 서버 애플리케이션**입니다.
다만 `/apiv2` REST API 가 있으므로 이를 감싸 **MCP 서버로 만들 수는 있습니다.**

### Q. API 토큰을 써야 하나요?
**외부 유료 API 토큰(OpenAI/Anthropic 등)은 전혀 필요 없습니다.** AI 와 무관한 프로젝트입니다.
다만 S-UI 가 **자체 발급하는 토큰**이 있습니다 (`common.Random(32)`, `service/user.go`).

| 경로 | 인증 방식 | 용도 |
|------|-----------|------|
| `/api/*` | 세션 쿠키 | 웹 UI 내부 통신 |
| `/apiv2/*` | `Token` 헤더 | 외부 프로그램/봇 연동 |

```sh
curl -H "Token: <32자토큰>" http://서버:2095/apiv2/clients
```

비용은 VPS 임대료 + 도메인 정도이고 소프트웨어 자체는 무료입니다.

### Q. 왜 GitHub 에서 인기가 있나요?
1. sing-box JSON 편집이라는 **명확한 고통을 해결**
2. 설치 한 줄 → **진입 장벽이 거의 0**
3. `x-ui` → `3x-ui` → `s-ui` 로 이어지는 **기존 생태계/사용자 풀**
4. Xray 기반 경쟁 패널 대비 **sing-box 전용**이라는 차별점 (Hysteria2, TUIC, AnyTLS, Snell 등 최신 프로토콜)
5. 6개 언어 × 12개 플랫폼 빌드로 **커버리지가 넓음**
6. 주석/테스트/CI/CONTRIBUTING 등 **코드 품질과 기여 친화성**
7. Docker 공식 이미지, 다크 테마, 스크린샷, 서드파티 생태계

### Q. 로컬 에이전트 구축에 도움이 되나요?
**직접적으로는 무관하지만, 인프라 패턴 측면에서 매우 유용합니다.**

| S-UI 구성요소 | 에이전트 시스템으로 치환 |
|---------------|--------------------------|
| `core/` sing-box 임베드 | LLM 런타임 임베드 |
| `Client` 트래픽/만료 | 에이전트 토큰 쿼터/만료 |
| `cronjob/checkCoreJob` | 에이전트 헬스체크·자동 재시작 |
| `Changes` 감사 로그 | 에이전트 실행 이력 |
| `/apiv2` 토큰 API | 에이전트 제어 API |
| `Stats` 시간 버킷 | 토큰 소비량 시계열 |

단, LLM 호출·툴 콜링·RAG·MCP 프로토콜은 전혀 포함되어 있지 않습니다.

### Q. React 나 PHP 로 만들 수 있나요?

| 계층 | React | PHP |
|------|:-----:|:---:|
| UI (화면) | ✅ 매우 적합 | ✅ 가능 |
| API / 비즈니스 로직 | ❌ | ⚠️ 가능하나 비권장 |
| 프록시 코어 임베드 | ❌ | ❌ 불가능 |

- **React**: 프론트엔드가 별도 저장소로 완전히 분리돼 REST API 로만 통신하므로
  **React 로 통째로 교체 가능**합니다. (Vite + TypeScript + TanStack Query + shadcn/ui + Recharts 권장)
- **PHP**: sing-box 는 Go 라이브러리라 PHP 에서 임베드할 수 없고,
  수천 커넥션 동시 처리 · 상주 프로세스 · 5초 주기 워치독도 PHP 모델과 맞지 않습니다.
  대신 **Laravel 등으로 결제·회원·정산 레이어를 만들고 실제 서버 조작은 `/apiv2` 에 위임**하는 방식이 현실적입니다.

**권장 조합**
```
React (고객 대시보드)
   ↓
Laravel/PHP (결제·회원·정산)
   ↓
S-UI /apiv2 (Go, 그대로 사용)
```

---

## 8. 수익화 아이디어

### 먼저 확인해야 할 리스크

1. **법적 리스크** — 프록시/VPN 서비스 *판매* 는 국가별 규제가 크게 다릅니다.
   README 에도 "개인 학습·교류 목적, 불법 사용 금지" 가 명시돼 있습니다.
   따라서 **서비스 판매보다 도구·기술·콘텐츠 판매 방향**이 안전합니다.
2. **GPL v3** — S-UI 코드를 수정해 배포하면 소스 공개 의무가 발생합니다.
   반면 **코드를 건드리지 않고 API 만 호출**하면 자체 코드는 공개 의무가 없습니다.
   → "API 래핑" 전략이 라이선스 안전지대입니다.

### 아이디어 목록

| # | 아이디어 | 난이도 | 기간 | 수익성 | 리스크 |
|---|---------|--------|------|--------|--------|
| 1 | 프리미엄 React 대시보드 판매 | 중 | 2–3개월 | 높음 | 낮음 |
| 2 | S-UI MCP 서버 + AI 운영 어시스턴트 | 하~중 | 2–4주 | 높음 | 낮음 |
| 3 | 멀티서버 통합 관제 SaaS | 중상 | 3–6개월 | 매우 높음 | 중간 |
| 4 | 결제 연동 자동화 (Laravel/WooCommerce) | 중 | 1–2개월 | 높음 | 중간 |
| 5 | 기술 교육 콘텐츠 (강의/전자책/유튜브) | 하 | 1–2개월 | 중간 | 매우 낮음 |
| 6 | 아키텍처 재활용 → 다른 도메인 SaaS | 상 | 4–8개월 | 최상 | 낮음 |

#### 1. 프리미엄 React 대시보드
현재 Vue 프론트엔드를 React 로 재구현한 고급 테마 판매.
프론트엔드가 별도 저작물이고 API 로만 통신하므로 GPL 측면에서 안전합니다.
모바일 반응형, 실시간 트래픽 차트, 대량 클라이언트 관리, 화이트라벨 브랜딩이 차별점.
가격 예시: Personal $29 / Pro $79 / Agency $199. 판매처: Gumroad, LemonSqueezy, ThemeForest.

#### 2. S-UI MCP 서버
`/apiv2` 를 MCP 툴로 감싸 AI 가 직접 서버를 조회·조작하게 합니다.

```
list_clients       → GET  /apiv2/clients
add_client         → POST /apiv2/save
get_traffic_report → GET  /apiv2/stats
restart_core       → POST /apiv2/restartSb
```

기본 MCP 서버는 오픈소스로 공개해 인지도를 확보하고,
멀티서버 통합·이상 탐지·자동 리포트를 유료 SaaS($9–29/월)로 제공하는 모델.
MCP 생태계 초기라 선점 효과가 큽니다.

#### 3. 멀티서버 통합 관제 SaaS
S-UI 는 서버 1대 = 패널 1개라 다중 서버 운영 시 불편합니다.
여러 패널의 `/apiv2` 를 폴링해 통합 대시보드, 다운 알림(텔레그램/슬랙),
클라이언트 서버 간 마이그레이션, 월간 리포트, 전체 백업, 팀 권한(RBAC) 제공.
가격 예시: 3대 무료 → 10대 $19/월 → 무제한 $49/월.

#### 4. 결제 연동 자동화
`결제(Stripe/아임포트/토스) → Webhook → Laravel → POST /apiv2/save → 구독 링크 자동 발송 → 만료 전 갱신 알림`
셀프호스팅 스크립트 $49–149, WooCommerce 플러그인 $39, 설치 대행 $200–500.
"서비스 판매"가 아니라 "도구 판매"라는 선을 이용약관으로 명확히 해야 합니다.

#### 5. 기술 교육 콘텐츠
실제 오픈소스를 교재로 쓰는 Go 백엔드 아키텍처 강의.
커리큘럼 예시: `embed.FS` 단일 바이너리 / Gin+GORM 계층 분리 / robfig cron 안전 설계 /
SQLite WAL 튜닝 / 토큰 인증과 타이밍 공격 방어 / 외부 Go 라이브러리 임베드 / GitHub Actions 멀티아키 릴리스.
한국어 콘텐츠가 거의 없어 블루오션이며 리스크가 가장 낮습니다.

#### 6. 아키텍처 재활용 (가장 큰 기회)
S-UI 에서 "프록시"만 제거하면 남는 것은 **모든 구독형 SaaS 의 공통 뼈대**입니다.
멀티테넌트 유저 관리 · 사용량 계량 · 쿼터 제한 · 만료/갱신 자동화 · 시계열 통계 ·
토큰 API · 감사 로그 · 단일 바이너리 배포.

| 전환 대상 | 치환 내용 |
|-----------|-----------|
| LLM API 게이트웨이 | 프록시 → LLM 프록시, 트래픽 → 토큰 사용량 |
| 셀프호스팅 파일 공유 | 클라이언트 → 사용자, 볼륨 → 저장 용량 |
| API 키 관리 플랫폼 | inbound → API 엔드포인트, 쿼터 그대로 |
| 게임서버 관리 패널 | sing-box → 게임 서버, 클라이언트 → 플레이어 |
| AI 에이전트 관제 | 코어 → 에이전트 런타임, 트래픽 → 토큰 소비 |

특히 **LLM API 게이트웨이**(부서별 토큰 사용량 계량·한도·리포트)는
기업 대상이라 객단가가 높고 법적 리스크가 사실상 없습니다.

### 권장 로드맵
```
1단계 (1개월)   #2 MCP 서버      — 작고 빠르게, 오픈소스로 인지도 확보
2단계 (2–3개월) #1 React 대시보드 — 확보한 사용자 대상 첫 매출
3단계 (6개월~)  #6 LLM 게이트웨이 — 축적한 경험·코드 자산으로 본격 확장
```
처음부터 #3, #6 같은 대형 과제로 시작하지 않는 것이 핵심입니다.

---

## 9. 이 저장소가 주는 학습 가치

1. **Go 백엔드 아키텍처 교과서** — `handler → service → database` 계층 분리의 정석
2. **구독형 SaaS 의 축소판** — 유저 관리 + 계량 + 한도 + 만료 + 감사 로그 + 토큰 API
3. **단일 바이너리 배포 전략** — `embed.FS` 로 프론트엔드까지 내장
4. **크론/워치독 패턴** — `Recover` + `SkipIfStillRunning` 체인
5. **CI 레퍼런스** — gofmt 검사, 테스트 게이트, 멀티아키 릴리스 워크플로

---

## 10. 주의사항

- README 명시: *"This project is only for personal learning and communication,
  please do not use it for illegal purposes, please do not use it in a production environment"*
- 프록시 도구는 국가·지역별 법규가 다르므로, 개인 서버 관리 또는
  **코드/아키텍처 학습 목적**으로 활용하는 것이 가장 안전합니다.
- GPL v3 이므로 코드를 수정해 배포할 경우 소스 공개 의무가 있습니다.
- 설치 직후 기본 계정(`admin`/`admin`)은 반드시 변경해야 합니다.
