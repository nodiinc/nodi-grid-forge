# CEMS ↔ GridNote 연동 검토

CEMS 내 GridNote 메뉴로 AI SCADA를 임베드하기 위한 연동 방안 비교 및 구현 계획입니다.

---

## 1. 연동 방안 비교

### 1.1 후보 방안

| 방안 | 설명 |
|------|------|
| A. CEMS 환경 직접 개발 | CEMS의 Spring Boot + React 환경에 맞춰 AI SCADA를 직접 구현 |
| B. iframe | AI SCADA를 독립 앱으로 배포, CEMS 페이지에 iframe으로 임베드 |
| C. 마이크로프론트엔드 | Module Federation / Single-SPA로 런타임 통합 |
| D. Web Components | AI SCADA를 `<grid-note>` 커스텀 엘리먼트로 제공 |

### 1.2 방안별 평가

| 기준 | A. 직접 개발 | B. iframe | C. 마이크로FE | D. Web Comp. |
|------|-------------|-----------|-------------|-------------|
| 기술 스택 독립성 | ✗ | ◎ | ○ | ○ |
| CSS/JS 격리 | ✗ (공유) | ◎ (완전) | △ (부분) | ○ (Shadow) |
| 독립 배포 | ✗ (공동 배포) | ◎ | ○ | △ |
| CEMS 코드 수정량 | 대규모 | 최소 | 중간 | 중간 |
| 장애 격리 | ✗ | ◎ | △ | △ |
| 구현 복잡도 | 높음 | 낮음 | 높음 | 중간 |
| UX 자연스러움 | ◎ | ○ | ◎ | ◎ |
| 높이/스크롤 제약 | 없음 | △ | 없음 | 없음 |
| 초기 구축 속도 | 매우 느림 | 빠름 | 느림 | 중간 |
| 유지보수 부담 | 전체 공동 관리 | 각자 독립 | 버전 호환 관리 | 번들 통합 관리 |

UX 자연스러움을 제외한 모든 기준에서 B안(iframe)이 우위에 있습니다. C, D안은 부분적 이점이 있으나 구현 복잡도 대비 실익이 크지 않습니다.

### 1.3 띵스파이어(CEMS 개발사) 관점

CEMS는 운영 중인 프로덕션 서비스이므로, 연동 방식 선택 시 기존 서비스에 대한 영향이 핵심 고려사항입니다.

| 기준 | A. 직접 개발 | B. iframe |
|------|-------------|-----------|
| CEMS 수정량 | 라우팅/상태관리/스타일 전면 통합 | iframe 1개 + postMessage 리스너 |
| 개발 공수 | CEMS 리소스 상당 부분 투입 | 수일 내 완료 가능 |
| 프로덕션 영향 | AI SCADA 코드 포함 — 장애 전파 가능 | 완전 격리 |
| 배포 | AI SCADA 변경마다 CEMS 공동 배포 | 각자 독립 배포 |

iframe 방식은 띵스파이어 측 개발 공수를 최소화하고, 기존 CEMS 개발/배포 프로세스를 변경 없이 유지할 수 있습니다.

### 1.4 HD현대일렉트릭(최종 고객) 관점

최종 사용자에게는 내부 구현 방식보다 "하나의 서비스로 느껴지는가"가 핵심입니다.

| 기준 | A. 직접 개발 | B. iframe |
|------|-------------|-----------|
| UX | 하나의 앱 느낌 | 테마/인증/네비 동기화로 사실상 동일 |
| 장애 영향 | CEMS 전체 파급 가능 | GridNote만 영향, CEMS 핵심 기능 유지 |
| 업데이트 속도 | CEMS 배포 주기에 종속 | AI SCADA 독립 배포로 즉시 반영 |

iframe 방식도 인증/테마/네비게이션 연동으로 매끄러운 통합 UX를 제공할 수 있으며, 장애 격리와 빠른 업데이트 주기는 운영 안정성 면에서 오히려 유리합니다.

### 1.5 추천: B. iframe

위 평가를 종합하면, iframe 방식이 모든 이해관계자에게 가장 균형 잡힌 선택입니다.

- **기술적으로** — 기술 스택 독립, 완전한 격리, 검증된 패턴 (Stripe, Google Maps 등 업계 표준)
- **띵스파이어에게** — CEMS 수정 최소, 개발 공수 최소, 프로덕션 리스크 없음
- **HD현대일렉트릭에게** — 통합 UX 제공, 장애 격리로 서비스 안정성 향상, 빠른 기능 업데이트

**A안(직접 개발) 비추천 사유:**

- AI SCADA의 코드 생성/컴파일 파이프라인이 Node.js 생태계에 강하게 의존하여, Spring Boot로 이식 시 우회 구현이 필요합니다
- 개발/유지보수 비용 증가 대비 기술적 이점이 적습니다
- CEMS와 AI SCADA의 배포 주기가 강제 동기화됩니다

---

## 2. iframe 연동 구현 계획

### 2.1 전체 구조

```text
┌─ 사용자 브라우저 ──────────────────────────────────────────────┐
│                                                                │
│  ┌─ CEMS (React SPA, CSR) ──────────────────────────────────┐ │
│  │  상단 네비게이션                                           │ │
│  │  [대시보드] [모니터링] [GridNote] [설정]    🔔 3           │ │
│  │                                                           │ │
│  │  ┌─ iframe (AI SCADA) ─────────────────────────────────┐ │ │
│  │  │  AI Chat │ Tag Panel │ SCADA Preview                │ │ │
│  │  │          │           │                              │ │ │
│  │  └──────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
        │                              │
        ▼                              ▼
┌─ 같은 VPC ─────────────────────────────────────────────────────┐
│  CEMS 서버 (Spring Boot)      AI SCADA 서버 (Next.js)          │
│  - 인증/사용자 관리            - AI 엔진 (LLM API 호출)         │
│  - CEMS 비즈니스 로직          - 코드 컴파일 (esbuild)          │
│  - 토큰 검증 API 제공          - 태그 릴레이 (WebSocket)        │
│                                - 화면 저장/버전 관리            │
│                                                                │
│  ※ 서버 간 통신: VPC 내부 HTTP (사설 IP)                        │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 인증 연동

CEMS가 인증을 소유하고, AI SCADA는 위임받아 사용합니다.

```text
인증 흐름:

  1. 사용자가 CEMS에 로그인 (CEMS JWT 발급)
  2. [GridNote] 클릭 → CEMS 프론트가 일회용 인증 코드 요청
     CEMS 프론트 → CEMS 서버: POST /api/auth/embed-code
     응답: { code: "abc123", expiresIn: 30 }  (30초 유효)
  3. iframe 생성
     <iframe src="https://gridnote.{domain}/embed?code=abc123" />
  4. AI SCADA 서버가 코드로 토큰 교환
     AI SCADA → CEMS 서버: POST /api/auth/exchange (VPC 내부)
     요청: { code: "abc123" }
     응답: { userId, orgId, role, siteIds, token }
  5. AI SCADA 세션 생성 → 해당 프로젝트/태그 로드

※ JWT를 URL에 직접 노출하지 않습니다 (브라우저 히스토리, 로그 노출 방지)
```

**CEMS가 제공할 API:**

```text
POST /api/auth/embed-code     일회용 코드 발급 (CEMS 프론트 → CEMS 서버)
POST /api/auth/exchange       코드 → 사용자 정보 교환 (AI SCADA 서버 → CEMS 서버)
GET  /api/users/{id}          사용자 상세 조회 (선택)
GET  /api/sites/{id}/tags     사이트별 태그 목록 조회 (선택)
```

### 2.3 데이터 교환

아래는 현재 식별된 교환 항목이며, 양측 협의를 통해 추가/변경될 수 있습니다.

```text
방향              채널           데이터                    시점
─────────────────────────────────────────────────────────────────
CEMS → GridNote  postMessage   인증 갱신 (토큰 리프레시)    토큰 만료 전
                 postMessage   테마/언어 변경              사용자 설정 변경 시
                 postMessage   컨텍스트 전환 (siteId 등)   사이트 전환 시
                 ...           (추가 항목 협의 가능)

GridNote → CEMS  postMessage   알람 카운트 갱신            알람 발생/해제 시
                 postMessage   네비게이션 요청              다른 CEMS 메뉴로 이동 시
                 postMessage   화면 타이틀 변경            화면 전환 시
                 ...           (추가 항목 협의 가능)
```

**postMessage 프로토콜 (예시):**

```typescript
// 메시지 타입 정의 — 양측 협의로 확장
type CemsToGridNote =
  | { type: 'AUTH_REFRESH'; token: string }
  | { type: 'THEME_CHANGE'; theme: 'light' | 'dark' }
  | { type: 'SITE_CHANGE'; siteId: string }
  // ... 추가 메시지 타입

type GridNoteToCems =
  | { type: 'ALARM_COUNT'; critical: number; warning: number }
  | { type: 'NAVIGATE'; path: string }
  | { type: 'TITLE_CHANGE'; title: string }
  | { type: 'READY' }  // iframe 로드 완료 신호
  // ... 추가 메시지 타입

// 수신 측 (양쪽 동일 패턴)
window.addEventListener('message', (event) => {
  if (event.origin !== ALLOWED_ORIGIN) return;  // origin 검증
  handle(event.data);
});
```

### 2.4 보안

| 항목 | 대응 |
|------|------|
| HTTPS | iframe 내부도 HTTPS 적용 (Mixed Content 차단 방지). AWS ALB + ACM 인증서 등 활용 |
| Origin 검증 | postMessage 수신 시 `event.origin` 화이트리스트 체크. CEMS 도메인만 허용 |
| X-Frame-Options | AI SCADA 서버에서 CEMS 도메인만 iframe 허용. `Content-Security-Policy: frame-ancestors https://cems.{domain}` |
| 토큰 노출 방지 | URL에 JWT 직접 전달 지양 → 일회용 코드 교환 방식. 코드는 단시간 유효, 1회 사용 후 폐기 |
| CORS 설정 | AI SCADA API의 CORS에 CEMS 도메인 허용. `credentials: true` (쿠키/인증 헤더 포함 시) |
| 세션 관리 | AI SCADA 자체 세션 발급 (httpOnly 쿠키). CEMS 로그아웃 시 → postMessage로 GridNote 세션 종료 |

### 2.5 화면 레이아웃

```text
CEMS의 iframe 컨테이너 CSS:

  .gridnote-container {
    width: 100%;
    height: calc(100vh - 60px);  /* CEMS 네비게이션 높이 제외 */
    border: none;
    overflow: hidden;
  }

AI SCADA /embed 라우트:
  - 자체 네비게이션 바 숨김 (CEMS 네비게이션 사용)
  - 프로젝트 내 화면 전환은 AI SCADA 내부 탭/사이드바로 처리
  - 전체 뷰포트를 채우는 레이아웃 (100vw × 100vh)

높이 동적 조절 (필요 시):
  GridNote → CEMS: postMessage({ type: 'RESIZE', height: 1200 })
  ※ SCADA 화면은 보통 뷰포트에 맞추므로 고정 높이로 충분합니다
```

### 2.6 CEMS 측 구현 사항

```text
CEMS 프론트엔드:
  1. GridNote 메뉴 라우트 추가 (/gridnote)
  2. iframe 렌더링 컴포넌트 (src, 인증 코드 전달)
  3. postMessage 리스너 (알람 배지 갱신, 네비게이션 처리)

CEMS 백엔드:
  1. POST /api/auth/embed-code  — 일회용 인증 코드 발급
  2. POST /api/auth/exchange    — 코드 → 사용자 정보 교환 (VPC 내부 전용)
```

### 2.7 AI SCADA 측 구현 사항

```text
프론트엔드:
  1. /embed 라우트        — iframe 전용 (자체 네비 숨김, 뷰포트 풀사이즈)
  2. postMessage 핸들러   — 인증 갱신, 테마 동기화, 사이트 전환
  3. postMessage 발신     — 알람 카운트, 네비 요청, READY 신호

백엔드:
  1. 인증 코드 교환 API    — CEMS 서버와 VPC 내부 통신
  2. CEMS 사용자 → AI SCADA 프로젝트 매핑
  3. CORS + CSP 설정      — CEMS 도메인 허용
  4. HTTPS 인증서 설치
```

### 2.8 구현 순서

```text
Step 1: 기반 연결
  - AI SCADA /embed 라우트 생성
  - CEMS iframe 컴포넌트 추가
  - HTTPS 설정
  → 결과: CEMS에서 GridNote 메뉴 클릭 → AI SCADA 화면 표시

Step 2: 인증 연동
  - CEMS embed-code / exchange API 구현
  - AI SCADA 인증 코드 수신 → 세션 생성
  - 사용자 → 프로젝트/태그 매핑
  → 결과: 로그인한 사용자에 맞는 프로젝트 자동 로드

Step 3: 실시간 동기화
  - postMessage 프로토콜 구현 (양방향)
  - 알람 배지, 테마 동기화, 로그아웃 연동
  → 결과: CEMS와 GridNote 간 실시간 상태 동기화
```
