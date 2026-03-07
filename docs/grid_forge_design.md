# Grid Forge Design

AI 기반 SCADA 생성 플랫폼 — Edge Gateway 연동 에너지 관리 시스템

---

## 1. 개요

Grid Forge는 **자연어와 시각적 입력으로 SCADA 화면을 생성하는 AI 플랫폼**이다.

기존 SCADA는 전문 엔지니어가 복잡한 에디터에서 도형 배치, 태그 연결, 알람 설정, 애니메이션 구성을 수작업으로 수행한다. 이 과정은 느리고, 변경 비용이 높으며, 엔지니어 의존도가 크다.

Grid Forge는 이 에디터를 **AI 대화 + 이미지 입력**으로 대체한다. AI가 React 코드를 직접 생성하는 **바이브코딩 방식**으로 동작한다.

```text
┌─────────────────────────────────────────────────────┐
│                     기존 SCADA                       │
│                                                      │
│   엔지니어 → SCADA Editor → 수작업 설계 → Runtime    │
│                                                      │
├──────────────────────────────────────────────────────┤
│                    Grid Forge                        │
│                                                      │
│   사용자 → 프롬프트 & 이미지 → AI 코드 생성 → Runtime │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**사용자 요청 예시:**

> "변압기 3개와 차단기 2개가 있는 수배전반 화면을 만들어줘.
> 각 변압기의 전류값을 표시하고 과전류 시 빨간색으로 표시해.
> 차단기에는 투입/개방 버튼을 달아줘."


## 2. 시스템 아키텍처

```text
┌───────────────────────────────────────────────────────┐
│                    사용자 브라우저                      │
│                                                       │
│  ┌──────────┐  ┌────────┐  ┌────────────────────────┐ │
│  │ AI Chat  │  │  Tag   │  │      Preview           │ │
│  │  Panel   │  │ Panel  │  │  ┌──────────────────┐  │ │
│  │          │  │        │  │  │ Drawing Layer    │  │ │
│  │ [이미지] │  │        │  │  │ (투명 오버레이)   │  │ │
│  │ [첨부]   │  │        │  │  ├──────────────────┤  │ │
│  │          │  │        │  │  │ ScreenRenderer   │  │ │
│  │ [진행률] │  │        │  │  │ SCADA 화면 렌더링 │  │ │
│  │          │  │        │  │  └──────────────────┘  │ │
│  └────┬─────┘  └───┬────┘  └───────┬────────────────┘ │
│  ┌────┴─────────────┴──────────────┘                  │
│  │ 알람 배너 / 네비게이션 바 (정형화된 UI)              │
│  └────────────────────────────────────────────────────┘│
└───────┼───────────┼──────────────────────┼────────────┘
                │               │         │
                ▼               │         │ WebSocket
      ┌──────────────────┐      │         │
      │   Next.js API    │      │         │
      │                  │      │         │
      │  ┌────────────┐  │      │         │
      │  │  LLM API   │  │      │         │
      │  │ (external)  │  │      │         │
      │  └────────────┘  │      │         │
      │  ┌────────────┐  │      │         │
      │  │ Knowledge  │  │      │         │
      │  │   Files    │  │      │         │
      │  └────────────┘  │      │         │
      │  ┌────────────┐  │      │         │
      │  │  Screen    │  │      │         │
      │  │  Storage   │  │      │         │
      │  │  (DB+캐시)  │  │      │         │
      │  └────────────┘  │      │         │
      └────────┬─────────┘      │         │
               │     WebSocket  │         │
               │    (localhost)  │         │
               ▼                ▼         ▼
      ┌──────────────────────────────────────┐
      │    TagBus Bridge (Python, systemd)   │
      │    태그 릴레이 + 제어 명령             │
      └────────────────┬─────────────────────┘
                       │ TagBus (eCAL)
      ┌──────────────────────────────────────┐
      │    Data Worker (Python, systemd)     │
      │    알람 평가 + 시계열 DB 저장          │
      └────────────────┬─────────────────────┘
                       │ TagBus (eCAL)
                       ▼
      ┌──────────────────────────────────────┐
      │           Edge Gateway               │
      │   Modbus / OPC UA / IEC 61850 / SNMP │
      └────────────────┬─────────────────────┘
                       │
                       ▼
      ┌──────────────────────────────────────┐
      │             전력 설비                  │
      │   변압기 / 차단기 / 수배전반 / 계전기   │
      └──────────────────────────────────────┘
```


## 3. 핵심 설계 결정

### 3.1 AI 생성물: React/JSX 직접 생성 (바이브코딩)

AI는 미리 정의된 컴포넌트를 조합하는 것이 아니라, **React/TSX 코드를 직접 생성**한다.

**이유:**

- 사전 컴포넌트화는 사용자의 모든 요구를 예측할 수 없음
- SCADA 화면은 현장마다 레이아웃, 장비 구성, 표현 방식이 천차만별
- LLM은 React/Tailwind 코드 생성에 이미 높은 역량을 보유

**LLM 호출 횟수:** 고정된 단계가 아니라 요청 복잡도에 따라 유동적이다. 단순 화면은 1회 호출, 복잡한 화면은 대화를 이어가며 점진적으로 완성한다.

**화면 해상도:** FHD(1920×1080)를 기본 디자인 타깃으로 한다. Knowledge 파일에 반응형 가이드라인을 포함하여 AI가 다양한 화면 크기에 대응하는 코드를 생성하도록 한다 (§10 참조).

**AI가 생성하는 코드 예시:**

```tsx
export default function SubstationPanel({ tags, setTags, screenState }) {
  return (
    <div className="grid grid-cols-3 gap-4 p-6 bg-gray-900 min-h-screen">
      {/* 변압기 1 */}
      <div className="border border-gray-700 rounded-lg p-4">
        <h3 className="text-white text-sm">변압기 TR-1</h3>
        <img src="/assets/transformer.svg" className="w-24 mx-auto" />
        <div className={`text-2xl font-bold ${
          tags['/site1/tr1/current']?.v > 400 ? 'text-red-500' : 'text-green-400'
        }`}>
          {tags['/site1/tr1/current']?.v ?? '--'} A
        </div>
      </div>

      {/* 차단기 CB-1 */}
      <div className="border border-gray-700 rounded-lg p-4">
        <h3 className="text-white text-sm">차단기 CB-1</h3>
        <div className={`w-4 h-4 rounded-full ${
          tags['/site1/cb1/status']?.v === 1 ? 'bg-green-500' : 'bg-red-500'
        }`} />
        <div className="flex gap-2 mt-2">
          <button
            className="px-3 py-1 bg-green-600 text-white rounded"
            onClick={() => setTags({ '/site1/cb1/command': 1 })}
          >투입</button>
          <button
            className="px-3 py-1 bg-red-600 text-white rounded"
            onClick={() => setTags({ '/site1/cb1/command': 0 })}
          >개방</button>
        </div>
      </div>
    </div>
  );
}
```

### 3.2 코드 저장: DB 원본 + 파일시스템 빌드 캐시

AI가 생성한 코드의 **원본은 DB(Screen.code)**에 저장한다. 파일시스템은 esbuild 컴파일을 위한 **빌드 캐시**로만 사용한다.

**DB를 원본으로 선택한 이유:**

- 서버 마이그레이션 = DB 이동(pg_dump)만으로 완료
- 백업/복구 = DB 백업이 곧 전체 백업
- 다중 인스턴스 운영 시 각 인스턴스가 자체 캐시를 빌드
- 디스크 관리: 캐시는 LRU 정책으로 정리 가능

**파일시스템을 캐시로 유지하는 이유:**

- esbuild, TypeScript 컴파일러 등 기존 도구를 그대로 활용
- 에러 발생 시 파일명:행번호로 정확한 위치 추적
- 개발자가 직접 파일을 열어 디버깅 가능

**동작 흐름:**

```text
AI 코드 생성 완료
    │
    ├─ 1. DB 저장: Screen.code = newCode, version++
    │      ScreenHistory에 이전 코드 백업
    │
    ├─ 2. 캐시 쓰기: src/screens/{projectId}/{slug}.tsx
    │      (esbuild 입력용 임시 파일)
    │
    ├─ 3. esbuild 컴파일 → .screen-builds/{projectId}/{slug}.js
    │
    └─ 4. WebSocket 알림 → 브라우저 번들 로드

서버 재시작 / 신규 인스턴스:
    요청 시 DB에서 Screen.code 읽기 → 파일 쓰기 → 컴파일 (온디맨드)
```

**디렉토리 구조 (프로젝트별 완전 격리, slug 기반):**

```text
src/screens/                     ← 빌드 캐시 (DB에서 복원 가능)
├── {projectId_A1}/              ← A사 "본사" 프로젝트
│   ├── substation-panel.tsx     ← slug: "substation-panel"
│   └── overview.tsx
├── {projectId_A2}/              ← A사 "공장" 프로젝트
│   └── motor-control.tsx
└── {projectId_B1}/              ← B사 프로젝트
    └── factory-overview.tsx

.screen-builds/                  ← 컴파일된 번들 캐시
├── {projectId_A1}/
│   ├── substation-panel.js
│   └── overview.js
└── ...

slug → 파일 경로 매핑:
  빌드 캐시: src/screens/{projectId}/{slug}.tsx
  번들 캐시: .screen-builds/{projectId}/{slug}.js
  별도의 filename 필드 없이 slug로 통일
```

**파일 조작 규칙:**

```text
허용 경로:    src/screens/{projectId}/** (프로젝트별 격리)
금지:         그 외 모든 경로

허용 확장자:  .tsx, .ts, .css
금지:         .js, .json, .env 등 설정 파일

허용 작업:    파일 생성, 파일 덮어쓰기, 파일 삭제
금지:         디렉토리 탈출 (../ 등), 심볼릭 링크
```

**파일 접근 시 격리 검증:**

```text
모든 파일 조작 요청 시:
1. projectId가 요청 사용자의 Organization 소속인지 DB에서 확인
2. 파일 경로가 해당 projectId 디렉토리 안에 있는지 확인
3. 경로 탈출 (../) 시도 차단
→ 위반 시 즉시 거부
```

**LLM 응답 형식:**

```json
{
  "files": [
    {
      "path": "substation-panel.tsx",
      "action": "write",
      "content": "export default function SubstationPanel({ tags, setTags }) { ... }"
    },
    {
      "path": "old-panel.tsx",
      "action": "delete"
    }
  ],
  "entryPoint": "substation-panel.tsx"
}
```

서버는 `{projectId}/` 하위에만 파일을 반영한다.

### 3.3 코드 반영 파이프라인: DB → 컴파일 → 무중단 교체

AI가 생성한 코드가 브라우저에 반영되기까지의 전체 흐름:

```text
1. 사용자: "차단기에 버튼 추가해줘"
   │
2. 서버: LLM API 호출 (2~10초)
   │  AI Progress: "코드 생성 중..."
   │
3. 서버: LLM 응답 수신 — TSX 코드
   │
4. 서버: 코드 검증 (AST 분석 + TypeScript)                 (~100ms)
   │
   ├─ 실패 → LLM에 오류 피드백 후 재생성 (최대 3회)
   │
5. 서버: DB 저장 + 파일 캐시 쓰기                           (~5ms)
   │  Screen.code = newCode, version++
   │  ScreenHistory에 이전 버전 백업
   │  src/screens/{projectId}/{slug}.tsx
   │
6. 서버: esbuild 컴파일                                     (~10ms)
   │  TSX → JS (ES module)
   │
7. 서버: WebSocket 브로드캐스트
   │  { type: "screen_updated", projectId, slug, version }
   │
8. 브라우저: dynamic import로 새 번들 로드                    (~50ms)
   │  ScreenRenderer가 컴포넌트 교체 (React re-render)
   │
   총 지연: LLM 응답 시간 + ~100ms
```

**서버 사이드 컴파일:**

```text
DB Screen.code
        │
        │  파일 캐시에 쓰기
        ▼
src/screens/{projectId}/{slug}.tsx
        │
        │  esbuild (TSX → JS, ~10ms)
        ▼
GET /api/screens/{projectId}/{slug}/bundle.js
        │
        │  서버: projectId가 사용자의 Organization 소속인지 확인
        │  실패 → 403 Forbidden
        │
        │  브라우저에서 dynamic import()로 로드
        ▼
React 컴포넌트로 렌더링
```

**브라우저 사이드 무중단 교체:**

```text
시간 ──────────────────────────────────────────────►

[기존 화면 v1 표시 중]
                    │ WebSocket: "screen_updated"
                    │
                    │ dynamic import()로 새 번들 로드
                    │ v1은 계속 표시됨 ← 끊김 없음
                    │
                    │ import 완료 → setComponent(newComponent)
                    │
                    [v2로 즉시 교체 ── React re-render]
```

사용자는 v1 화면을 보다가 순간적으로 v2로 바뀌는 것만 느낀다. 흰 화면이나 로딩 스피너 없음.

### 3.4 Direct Rendering: 메인 React 트리에 직접 렌더링

AI가 생성한 코드는 **메인 React 트리에 직접 렌더링**된다. iframe을 사용하지 않으므로 postMessage 직렬화, 델타 프로토콜, Mini React Runtime 등의 복잡성이 없다.

```text
┌─ ScreenRenderer ────────────────────────────────────┐
│                                                      │
│  hooks: tags, alarms, chartData, screenState ...     │
│                                                      │
│  ┌─ AI 생성 컴포넌트 (dynamic import) ────────────┐  │
│  │                                                │  │
│  │  React props로 직접 전달:                       │  │
│  │    tags, setTags, alarms, chartData,           │  │
│  │    screenState, userSettings, assets, navigate │  │
│  │                                                │  │
│  │  메인 앱의 React 인스턴스 공유                    │  │
│  │  별도 런타임 불필요                              │  │
│  │                                                │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**ScreenRenderer 핵심 구현:**

```tsx
function ScreenRenderer({ projectId, slug }) {
  const [Component, setComponent] = useState(null);
  const tags = useTagSubscription(projectId);
  const setTags = useTagControl(projectId);
  const alarms = useAlarms(projectId);
  const chartData = useChartData(projectId);
  const screenState = useScreenState(projectId, slug);
  // ...

  // 번들 로드 (초기 + hot update)
  useEffect(() => {
    import(`/api/screens/${projectId}/${slug}/bundle.js?v=${version}`)
      .then(mod => setComponent(() => mod.default));
  }, [version]);

  if (!Component) return <Loading />;

  return (
    <Component
      tags={tags}
      setTags={setTags}
      alarms={alarms}
      chartData={chartData}
      screenState={screenState}
      userSettings={userSettings}
      assets={assets}
      navigate={navigate}
    />
  );
}
```

**Direct Rendering의 이점:**

```text
postMessage 직렬화/역직렬화  → React props 직접 전달 (제로 오버헤드)
델타 업데이트 프로토콜         → React 리렌더링이 자동 처리
Mini React Runtime            → 메인 앱 React 인스턴스 공유
번들 load → ready 핸드셰이크   → setComponent()로 즉시 교체
init 재전송 로직               → hooks가 항상 최신 상태 유지
```

**보안 모델: AST 검증 + 인프라 레벨 방어**

iframe 브라우저 격리 대신 **AST 기반 정적 분석을 1차 방어선**으로 사용한다.

| 위협 | 방어 수단 | 설명 |
|------|-----------|------|
| `fetch`/`XMLHttpRequest` | AST 검출 + 재생성 | 네트워크 호출 패턴 검출 시 LLM에 재생성 요청 |
| `import` 외부 모듈 | esbuild 번들링 | 외부 모듈은 번들에 포함되지 않아 로드 실패 |
| `eval`/`Function` 동적 실행 | AST 검출 + 재생성 | 동적 코드 실행 패턴 검출 |
| `localStorage`/`cookie` | AST 검출 + 재생성 | 브라우저 저장소 접근 패턴 검출 |
| `document` 직접 조작 | AST 검출 + 재생성 | DOM API 직접 호출 패턴 검출 |

**이 보안 모델이 충분한 이유:**

```text
1. 코드 생성 주체: LLM (서버에서 제어된 프롬프트로 호출)
   → 사용자가 직접 임의 코드를 작성하는 것이 아님
   → LLM이 의도적으로 AST 검증을 우회하는 코드를 생성할 확률은 극히 낮음

2. 위협 모델: 인증된 조직 내 사용자가 프롬프트 인젝션으로 악성 코드 생성 유도
   → 자기 조직의 데이터를 자기가 접근하는 상황
   → 데이터 유출 대상이 자기 자신

3. SCADA 환경: 산업 네트워크는 방화벽으로 외부 접근 차단
   → 인프라 레벨에서 네트워크 격리가 이미 존재

4. 향후 강화 가능: 보안 요구사항 변경 시 iframe 래핑 추가 가능
   → props 인터페이스가 동일하므로 코드 변경 최소
```

### 3.5 런타임 상태 보존: 코드는 껍데기, 데이터는 바깥

코드가 교체되어도 **런타임 상태(실시간 데이터, 알람, 차트, 사용자 입력값)는 유지**되어야 한다.

**핵심 원칙:** 화면 코드는 렌더링 로직만 담당한다. 모든 런타임 상태는 `ScreenRenderer`의 hooks가 관리한다. `setComponent()`로 AI 생성 컴포넌트가 교체되어도 hooks는 재실행되지 않으므로, 데이터는 자연스럽게 보존된다.

```text
ScreenRenderer (hooks — 불변, 코드 교체에 영향 없음)
│
│  아래 상태들은 ScreenRenderer의 hooks가 관리
│  Component가 교체되어도 hooks는 재실행되지 않음 (React 동작 원리)
│
├── tags              실시간 태그 값 (WebSocket 구독)
├── setTags           제어 명령 전송 (콜백 함수 직접 전달)
├── alarms            알람 이력
├── chartData         차트에 로드된 시계열 데이터
├── userSettings      사용자 설정값 (테마, 필터 등)
└── screenState       화면별 사용자 입력값 (폼 필드, 선택값 등)
    │
    │  React props (직접 전달 — 직렬화 없음)
    ▼
  <Component {...props} />  (교체 대상 — props로 받은 데이터로 렌더링만 수행)
```

**각 상태의 저장 위치와 생존 범위:**

```text
상태             저장 위치          코드 교체 시    페이지 이동 시    새로고침 시
──────────────────────────────────────────────────────────────────────────
tags (실시간)    WebSocket 구독     유지 ✓         재구독           재구독
alarms           서버 + 로컬 캐시   유지 ✓         유지 ✓          서버에서 복원
chartData        로컬 메모리        유지 ✓         초기화           초기화
userSettings     DB (서버)          유지 ✓         유지 ✓          DB에서 복원
screenState      sessionStorage     유지 ✓         유지 ✓          복원 ✓
logs             서버 DB            유지 ✓         유지 ✓          서버에서 복원
```

**코드 교체 시 실제 동작:**

```text
[상황] 사용자가 런타임에서 2시간 운영 후, 에디터에서 화면 수정 요청

운영 중 누적된 상태:
  - tags: 실시간 값 수신 중
  - alarms: 30건 누적
  - chartData: 최근 1시간 전력량 트렌드
  - screenState: { targetVoltage: 225, activeTab: "detail" }

AI가 코드 수정 → DB 저장 → 컴파일 → WebSocket 알림
  │
  ▼
ScreenRenderer:
  1. WebSocket: { type: "screen_updated", version: N }
  2. dynamic import(`bundle.js?v=N`) → 새 컴포넌트 로드
  3. setComponent(newComponent) → React re-render
     │
     ├── tags: 그대로 (hooks 유지, WebSocket 구독 유지)
     ├── alarms: 그대로 (30건 유지)
     ├── chartData: 그대로 (트렌드 유지)
     ├── screenState: 그대로 (targetVoltage: 225)
     └── 새 코드의 UI만 변경 (예: 버튼 추가, 레이아웃 변경)

사용자 체감: 화면이 살짝 바뀌지만 데이터/입력값은 그대로
```

### 3.6 AI에게 제공되는 API 계약

화면 코드에서 사용할 수 있는 React props (ScreenRenderer가 직접 전달):

| Props | 타입 | 용도 |
|-------|------|------|
| `tags` | `{ [tagId]: { v, q, t } }` | 실시간 태그 값 읽기 |
| `setTags` | `(pairs) => void` | 제어 명령 전송 |
| `alarms` | `{ active, history }` | 알람 목록 표시 |
| `chartData` | `{ query(tagId, from, to) }` | 시계열 데이터 조회 |
| `userSettings` | `{ theme, ... }` | 사용자 설정 |
| `screenState` | `{ state, setState }` | 사용자 입력값 보존 |
| `assets` | `string[]` | 업로드된 이미지 URL 목록 |
| `navigate` | `(screenSlug) => void` | 화면 간 이동 |

화면 코드 수정은 **AI를 통해서만** 가능하다. 직접 코드 편집 기능은 제공하지 않는다. 이는 AI 세션 컨텍스트와의 일관성을 보장하고, 검증 파이프라인을 우회하는 경로를 차단한다.

**금지 패턴 (AST 분석으로 검출 → LLM에 재생성 요청):**

- `useState`, `useReducer` — `screenState` 사용 (코드 교체 시 내부 state 소실 방지)
- `useEffect`로 데이터 fetch — `chartData.query` 사용
- `fetch`, `XMLHttpRequest`, `WebSocket` — 네트워크 접근 금지
- `import` — 외부 모듈 로드 금지 (esbuild 번들링에서도 차단)
- `eval`, `Function` — 동적 코드 실행 금지
- `document`, `window.location` — DOM/브라우저 API 직접 접근 금지
- `localStorage`, `sessionStorage`, `cookie` — 브라우저 저장소 접근 금지

AST 분석이 보안과 품질 모두를 담당한다 (§3.4 보안 모델 참조).

### 3.7 코드 검증

파일 저장 전에 서버에서 AST 기반 분석을 수행한다.

```text
검증 단계:
1. AST 분석 — 금지 패턴 검출 (useState, useReducer 등)
   정규식이 아닌 AST 파싱으로 정확한 검출
2. TypeScript 컴파일 (타입 오류 검출)
3. esbuild 트랜스파일 (구문 오류 검출)

검증 역할:
  보안 + 품질 → AST 분석이 단일 방어선으로 담당
  보안 패턴: fetch, eval, import, document 등 검출
  품질 패턴: useState, useReducer 등 검출

검증 실패 시:
  → LLM에 오류 메시지와 함께 재요청
  → "생성된 코드에 다음 문제가 있습니다:
      - useState 사용: screenState.setState를 사용해주세요
      규칙을 준수하여 다시 생성해주세요."
  → 최대 3회 재시도 → 실패 시 사용자에게 오류 표시
```


## 4. 시각적 입력과 화면 수정

사용자는 **3가지 방식**으로 시각적 입력을 제공할 수 있다:

1. **이미지 업로드** — 외부에서 그린 스케치, 설계 도면, 기존 SCADA 캡처 등을 첨부
2. **Drawing Panel** — Preview 위의 투명 오버레이에서 직접 드로잉 (tldraw 기반)
3. **이미지 자산** — 화면에서 사용할 SVG/PNG 아이콘 업로드

세 방식 모두 최종적으로 이미지 + 텍스트로 LLM에 전달되므로, AI 파이프라인은 동일하다.

### 4.1 신규 화면 생성

```text
┌──────────────────────────────────────────────────────┐
│ 사용자                                               │
│                                                      │
│  1. 텍스트: "수배전반 화면 만들어줘"                    │
│                                                      │
│  2. (선택) 시각적 입력:                                │
│     a. 이미지 업로드 — Paint 등 외부 도구에서 그린 스케치 │
│     b. Drawing Panel — 빈 캔버스에 직접 레이아웃 스케치  │
│     c. 설계 도면 사진 첨부                              │
│                                                      │
│  → AI가 코드 생성 → Preview에 즉시 반영                │
└──────────────────────────────────────────────────────┘
```

### 4.2 기존 화면 수정

기존 Runtime 화면을 수정하고 싶을 때의 두 가지 방법:

**방법 A: Drawing Panel (인앱 어노테이션)**

```text
┌──────────────────────────────────────────────────────┐
│                                                      │
│  1. [드로잉 모드] 토글 ON                              │
│     → Preview 위에 투명 Drawing Layer 활성화           │
│     → Runtime 화면은 그대로 보이되 클릭 비활성화         │
│                                                      │
│  2. Drawing Layer 위에 직접 어노테이션                  │
│     → 펜 (빨간색 등): 원하는 위치에 메모                │
│     → 화살표: "이거 여기로 옮겨"                       │
│     → 영역 박스: "이 부분 삭제" / "여기에 추가"         │
│     → 텍스트: 직접 설명 작성                           │
│     → 도형: 원, 사각형 등으로 위치 지정                 │
│                                                      │
│  3. [전송] 클릭                                       │
│     → Drawing Layer를 이미지로 export                  │
│     → Runtime 화면을 캡처 (html2canvas)                │
│     → 두 이미지를 합성하여 AI에 전달                    │
│     → 텍스트 설명과 함께 LLM 호출                      │
│                                                      │
│  4. AI가 수정된 코드 생성 → 무중단 교체                 │
│     → 드로잉 모드 자동 OFF                             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**방법 B: 외부 이미지 업로드 (기존 방식)**

```text
┌──────────────────────────────────────────────────────┐
│                                                      │
│  1. [캡처] 버튼 클릭                                  │
│     → 현재 Runtime 화면을 자동 캡처 → 다운로드          │
│                                                      │
│  2. 외부 도구(Paint 등)에서 캡처 이미지 위에 편집        │
│                                                      │
│  3. 편집된 이미지를 Chat Panel에 업로드 + 텍스트 입력    │
│     예: "빨간 동그라미 친 부분에 트렌드 차트 추가해줘"   │
│                                                      │
│  4. AI가 수정된 코드 생성 → 무중단 교체                 │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 4.3 Drawing Panel 구조

```text
┌──────────────────────────────────────┐
│          Preview 영역                │
│                                      │
│  ┌────────────────────────────────┐  │
│  │     Drawing Layer (tldraw)     │  │  z-index: 위
│  │     투명 배경                   │  │  드로잉 모드 ON: pointer-events: auto
│  │     펜/화살표/박스/텍스트        │  │  드로잉 모드 OFF: pointer-events: none
│  ├────────────────────────────────┤  │
│  │     ScreenRenderer             │  │  z-index: 아래
│  │     실제 SCADA 화면             │  │  드로잉 모드 ON: pointer-events: none
│  │     실시간 데이터 표시 중        │  │  드로잉 모드 OFF: pointer-events: auto
│  └────────────────────────────────┘  │
│                                      │
│  [드로잉 모드 ON/OFF]  [전송]  [캡처] │
└──────────────────────────────────────┘
```

드로잉 모드 토글로 Drawing Layer와 Runtime 간 입력 전환. Runtime은 드로잉 모드에서도 계속 실시간 데이터를 표시한다.

### 4.4 AI에게 전달되는 수정 컨텍스트

Drawing Panel이든 이미지 업로드든, LLM에 전달되는 형식은 동일하다:

```text
[System] 기존 화면 코드:
  <현재 TSX 코드>

[System] 사용 가능한 태그 목록:
  /site1/tr1/voltage, /site1/tr1/current, ...

[System] 활성 알람 규칙:
  <해당 프로젝트의 AlarmRule 목록>

[User] 이미지: <어노테이션된 스크린샷 (Runtime + 드로잉 합성)>
[User] "빨간 동그라미 친 부분에 전력량 트렌드 차트 추가해줘"
```

### 4.5 이미지 자산 관리

사용자가 SVG/PNG 등의 이미지를 업로드하면 **DB(ScreenAsset)**에 저장하고, 화면 코드에서 API URL로 참조한다.

```text
사용자: "이 변압기 아이콘 사용해줘" + transformer_custom.svg 업로드
  → DB ScreenAsset에 저장 (binary data + metadata)
  → 서빙 URL: /api/assets/{assetId}/{filename}
  → AI가 코드에서 참조: <img src="/api/assets/{assetId}/transformer_custom.svg" />
  → assets props에 URL 목록 포함
```

DB에 저장함으로써 코드(Screen.code)와 동일한 백업/복구/마이그레이션 경로를 따른다.


## 5. 실시간 데이터 경로

### 5.1 Edge Gateway 등록과 태그 스코핑

TagBus에는 모든 고객의 Edge Gateway가 발행하는 태그가 섞여 있다. 각 고객(Organization)이 자기 태그만 볼 수 있도록 **Gateway 등록 → 태그 접두사 기반 격리**를 적용한다.

```text
TagBus (로컬 머신 — 모든 태그)
├── /acme/plant1/tr1/voltage       ← A사 게이트웨이
├── /acme/plant1/tr1/current
├── /acme/plant2/cb1/status
├── /globex/factory/motor1/rpm     ← B사 게이트웨이
├── /globex/factory/motor1/temp
└── ...

Gateway 등록:
  A사 → Gateway "edge-plant1" (prefix: /acme/plant1)
       → Gateway "edge-plant2" (prefix: /acme/plant2)
  B사 → Gateway "edge-factory" (prefix: /globex/factory)

결과:
  A사 사용자 → /acme/plant1/**, /acme/plant2/** 만 접근 가능
  B사 사용자 → /globex/factory/** 만 접근 가능
```

**태그 접근 제어 흐름:**

```text
사용자 로그인
    │
    ▼
Organization 확인 (UserOrganization)
    │
    ▼
Project 선택
    │
    ▼
Project에 연결된 Gateway 목록 조회 (ProjectGateway)
    │
    ▼
각 Gateway의 tagPrefix 수집
    예: ["/acme/plant1", "/acme/plant2"]
    │
    ▼
TagBus Bridge: 해당 prefix의 태그만 구독
    │
    ▼
WebSocket: 필터링된 태그만 브라우저에 전달
    │
    ▼
Tag Panel: 사용자에게 자기 태그만 표시
AI Engine: LLM에 자기 태그 목록만 전달
```

**tagPrefix 겹침 방지:**

```text
Gateway 등록 시 tagPrefix 유효성 검증:
  1. 동일 Organization 내 기존 Gateway들의 tagPrefix와 비교
  2. 포함 관계 차단:
     - 기존: /acme/plant1       → 신규: /acme/plant1/sub1  (거부 — 기존의 하위)
     - 기존: /acme/plant1/sub1  → 신규: /acme/plant1       (거부 — 기존을 포함)
  3. 동일 prefix 차단: 이미 /acme/plant1이 있으면 동일 값 거부
  4. 다른 Organization의 prefix와는 겹침 허용 (격리되므로 무관)
```

**태그 자동 검색 (Discovery):**

Gateway를 등록하면 해당 prefix의 태그를 자동으로 수집한다.

```text
Gateway 등록: appId="edge-plant1", tagPrefix="/acme/plant1"
    │
    ▼
TagBus Bridge가 주기적으로 태그 목록 스캔
    │
    ▼
"/acme/plant1" 로 시작하는 태그 필터링
    /acme/plant1/tr1/voltage
    /acme/plant1/tr1/current
    /acme/plant1/cb1/status
    │
    ▼
TagConfig에 자동 등록 (label, unit 등은 관리자가 나중에 설정)
새 태그 추가 시 자동 발견, 삭제된 태그는 stale 표시
```

### 5.2 TagBus Bridge + Data Worker: 역할 분리

실시간 태그 릴레이와 데이터 처리를 **두 개의 독립 프로세스**로 분리한다. 둘 다 systemd 관리 서비스로 운영된다.

```text
┌─────────────────────────────────────────────────────────────────┐
│ 서버 머신                                                        │
│                                                                 │
│  ┌─ Next.js ────────────┐                                       │
│  │  API Routes          │                                       │
│  │  WebSocket Server    │                                       │
│  │  Health Monitor      │                                       │
│  └──┬──────────┬────────┘                                       │
│     │          │                                                │
│     │ WS(:9100)│ WS(:9101)                                      │
│     │ + token  │ + token                                        │
│     ▼          ▼                                                │
│  ┌──────────────────┐   ┌──────────────────────┐                │
│  │ TagBus Bridge    │   │ Data Worker          │                │
│  │ (Python/systemd) │   │ (Python/systemd)     │                │
│  │                  │   │                      │                │
│  │ • 태그 릴레이     │   │ • AlarmRule 평가     │                │
│  │   (TagBus→WS)    │   │ • AlarmEvent 생성    │                │
│  │ • 제어 명령 릴레이 │   │ • 시계열 DB 저장     │                │
│  │   (WS→TagBus)    │   │ • AlarmRule 핫리로드  │                │
│  │ • 태그 Discovery  │   │                      │                │
│  └────────┬─────────┘   └──────────┬───────────┘                │
│           │ TagBus (eCAL)          │ TagBus (eCAL)              │
│           └────────────┬───────────┘                            │
│                        ▼                                        │
│              Edge Gateway (Modbus / OPC UA / ...)               │
└─────────────────────────────────────────────────────────────────┘
```

**역할 분리 이유:**

| | TagBus Bridge | Data Worker |
|---|---|---|
| 역할 | 실시간 태그 중계, 제어 명령 | 알람 평가, 시계열 저장 |
| 지연 요구 | 엄격 (100ms 이내) | 유연 (수초 허용) |
| 장애 영향 | 실시간 모니터링 중단 | 알람/이력 일시 중단, 모니터링 정상 |
| TagBus 연결 | 독립 구독 | 독립 구독 |

**Bridge ↔ Next.js 인증:**

localhost WebSocket이지만, 공유 토큰으로 인증한다. 환경변수 `BRIDGE_AUTH_TOKEN`을 Next.js와 Bridge/Worker가 공유하고, WebSocket 연결 시 첫 메시지로 토큰을 검증한다.

**Data Worker AlarmRule 핫리로드:**

```text
1. Worker 시작 시: Next.js API로 전체 AlarmRule 로드
2. 사용자가 알람 규칙 변경 시:
   Next.js → Worker WebSocket: { type: "alarm_rules_updated" }
   Worker → Next.js API: 변경된 AlarmRule만 재로드
3. 잠깐의 알람 누락은 허용 (규칙 리로드 ~100ms 동안)
```

**장애 복구 시나리오:**

```text
Bridge 크래시:
  1. systemd가 2초 내 자동 재시작
  2. Next.js에 활성 구독 목록 요청 → 재구독
  3. 복구까지 (~3~5초): 브라우저에 stale 배너 표시
  4. 복구 후: 배너 자동 제거

Data Worker 크래시:
  1. systemd가 2초 내 자동 재시작
  2. AlarmRule 전체 로드 → 알람 평가 재개
  3. 실시간 모니터링은 영향 없음 (Bridge 독립)
  4. 크래시 동안 알람 누락 가능 (수초, 허용)

Next.js ↔ Bridge/Worker 연결 프로토콜:
  프로세스 시작 → Next.js에 WebSocket 연결 + 토큰 인증
  Next.js가 ping (3초 간격) → pong
  3회 연속 실패 → 다운 판정 → 브라우저에 경고
  재연결 시 → 구독/규칙 동기화 → 정상 복귀
```

### 5.3 브라우저 ↔ 서버 통신

```text
┌────────────┐  WebSocket   ┌──────────────┐  WS(:9100)  ┌──────────┐
│  브라우저   │◄────────────►│  Next.js API │◄───────────►│  TagBus  │
│            │              │              │             │  Bridge  │
│  tags 수신  │  subscribe   │              │  WS(:9101)  ├──────────┤
│  setTags   │  command     │              │◄───────────►│  Data    │
└────────────┘              └──────────────┘             │  Worker  │
                                                        └──────────┘
```

- **모니터링**: 태그 값 변경 → TagBus 콜백 → Bridge WebSocket → Next.js → 브라우저 (React state 업데이트 → 리렌더링)
- **제어**: `setTags()` 콜백 호출 → Next.js API → 서버 검증 (권한 + rate limit) → Bridge → `tb.set_tags()` → Edge Gateway → 설비
- **알람**: 태그 값 변경 → Data Worker 조건 평가 → AlarmEvent → Next.js WebSocket → 브라우저 알람 배너
- **구독 관리**: Project의 Gateway prefix에 해당하는 태그만 구독. 화면 전환 시 구독 태그 목록 갱신

### 5.4 멀티테넌시 실시간 데이터 관리

다수의 조직/사용자가 동시 접속해도 효율적으로 동작하기 위한 3계층 구조:

```text
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: TagBus Connection (싱글톤)                          │
│                                                             │
│   TagBus 연결 1개. 현재 필요한 모든 태그를 합산하여 구독.      │
│   태그별 레퍼런스 카운팅으로 구독/해제 관리.                    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Project Channel (프로젝트별 공유)                    │
│                                                             │
│   같은 프로젝트를 보는 사용자들은 하나의 채널을 공유.           │
│   태그 업데이트를 받아서 해당 채널의 WebSocket들에 브로드캐스트. │
│   첫 사용자 접속 시 생성, 마지막 사용자 퇴장 시 소멸.          │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: User Session (사용자별 격리)                         │
│                                                             │
│   WebSocket 연결. 인증/권한 검증. 제어 명령 처리.              │
│   자기 Project Channel의 데이터만 수신.                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**동작 예시:**

```text
A사 — Project "본사" (태그 500개)
  ├── User A (Operator) ── ws ──┐
  ├── User B (Engineer) ── ws ──┼── Project Channel "본사"
  └── User C (Viewer)   ── ws ──┘        │
                                         │ 태그 500개 구독
B사 — Project "공장" (태그 200개)          │
  ├── User D (Operator) ── ws ──┐        │
  └── User E (Viewer)   ── ws ──┼── Project Channel "B공장"
                                │        │ 태그 200개 구독
                                ▼        ▼
                      ┌───────────────────────┐
                      │ TagBus Connection     │
                      │ (싱글톤, 1개)          │
                      │ 실제 구독: 700개       │
                      │ (중복 태그 1번만 구독)  │
                      └───────────────────────┘

5명 접속 → TagBus 연결 1개, Channel 2개
```

**리소스 최적화:**

```text
화면별 태그 구독 (Fine-grained):
  User A: "수배전반" 화면 → 태그 200개만 필요
  User B: "개요" 화면     → 태그 50개만 필요
  → Channel은 합집합 250개만 구독 (500개 전체가 아님)
  → User A가 "개요"로 이동하면 "수배전반" 전용 태그 구독 해제

비활성 탭 처리:
  탭 비활성 (숨김)     → 업데이트 주기 낮춤 (1초 → 10초)
  장시간 비활성 (30분+) → 구독 일시 해제, 탭 복귀 시 스냅샷 전송
  브라우저 종료         → 세션 정리, Channel에서 제거

업데이트 스로틀링:
  태그 변경을 100ms 간격으로 배치 전송 (동일 태그는 마지막 값만)
```

### 5.5 시계열 데이터

`chartData.query(tagId, from, to)`는 **별도의 시계열 DB 서버**에서 데이터를 조회한다.

```text
AI 생성 컴포넌트: chartData.query(tagId, from, to)
    │ React props 콜백 호출
    ▼
ScreenRenderer
    │ API 호출
    ▼
Next.js API: GET /api/timeseries?tagId=...&from=...&to=...
    │
    │ Organization/Project 소속 확인
    │ tagId가 프로젝트 Gateway prefix에 속하는지 검증
    ▼
Time-Series DB (외부 서버)
    │ 쿼리 결과
    ▼
Next.js → ScreenRenderer (state 업데이트) → 컴포넌트 리렌더링
```

**Data Worker**가 TagBus에서 수신한 태그 값을 시계열 DB에 저장한다. Bridge와 독립적으로 동작하므로 실시간 릴레이에 영향을 주지 않는다.

**Data Worker의 DB 접근 경로:**

Data Worker는 PostgreSQL에 직접 접근하지 않는다. AlarmEvent 생성, AlarmRule 조회 등 모든 DB 작업은 **Next.js API를 경유**한다.

```text
Data Worker → Next.js API → Prisma → PostgreSQL

이유:
  - Prisma 스키마 단일 관리 (Python에서 별도 ORM/마이그레이션 불필요)
  - 비즈니스 로직(검증, 감사 로그)을 API에 집중
  - Data Worker는 지연에 민감하지 않으므로 API 경유 지연(~10ms) 무관
  - DB 커넥션 풀을 Next.js에서 단일 관리

API 엔드포인트 (내부용, Bridge/Worker 전용):
  POST /api/internal/alarm-events     AlarmEvent 생성
  GET  /api/internal/alarm-rules      AlarmRule 목록 조회
  POST /api/internal/alarm-events/:id/clear  AlarmEvent 해제
```


## 6. 알람 시스템

### 6.1 알람 개요

알람은 SCADA의 핵심 기능으로, **태그값 기반 조건 평가 → 이벤트 발생 → 사용자 확인(Acknowledge)** 워크플로우를 따른다. 알람 표시 UI는 AI 생성이 아닌 **정형화된 포맷**으로 제공한다.

```text
┌─ 브라우저 ──────────────────────────────────────────────┐
│                                                         │
│  ┌─ 네비게이션 바 + 알람 배너 (정형화, AI 생성 아님) ───┐  │
│  │  [프로젝트명]  [화면1] [화면2]    🔔 3  ⚠ 12       │  │
│  │                                                     │  │
│  │  ┌─ 활성 알람 배너 (위험 알람 시 자동 표시) ────────┐ │  │
│  │  │ 🔴 TR-1 과전류 (450A > 400A)  14:32  [확인]    │ │  │
│  │  │ 🟡 CB-2 통신 두절              14:30  [확인]    │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─ 중앙 영역 (AI 생성 SCADA 화면, ScreenRenderer) ────┐  │
│  │                                                     │  │
│  │  AI가 생성한 화면 (alarms props로 알람 데이터 수신)   │  │
│  │                                                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 6.2 알람 규칙 정의

사용자가 **알람 에디터 화면**에서 태그 + 조건을 지정하여 알람 규칙을 생성한다.

**지원 조건 타입:**

| 조건 | 설명 | 예시 |
|------|------|------|
| `GT` (>) | 초과 | 전류 > 400A |
| `GTE` (>=) | 이상 | 온도 >= 80°C |
| `LT` (<) | 미만 | 전압 < 200V |
| `LTE` (<=) | 이하 | 유량 <= 0 |
| `EQ` (==) | 같음 | 상태 == 0 (정지) |
| `NEQ` (!=) | 다름 | 모드 != 1 (비정상) |
| `DEADBAND` | 데드밴드 | 설정값 ± 5% 벗어남 |

**알람 심각도:**

```text
CRITICAL  — 즉시 조치 필요 (빨강)     예: 과전류, 비상 정지
WARNING   — 주의 필요 (노랑)          예: 통신 두절, 임계값 접근
INFO      — 정보성 (파랑)             예: 상태 변경, 설정값 변경
```

**알람 규칙 예시:**

```text
규칙: "TR-1 과전류"
  태그:     /acme/plant1/tr1/current (TagConfig 참조)
  조건:     GT (>)
  값:       400
  심각도:   CRITICAL
  메시지:   "변압기 TR-1 과전류 ({value}A > 400A)"
  데드밴드: 10  (410A에서 발생 후 390A 이하로 내려가야 해제)
```

### 6.3 알람 평가와 라이프사이클

알람 조건 평가는 **Data Worker**에서 수행한다. TagBus에서 직접 태그값을 수신하고, 조건을 평가하여 AlarmEvent를 생성한다.

```text
Data Worker: TagBus 태그값 수신
    │
    ▼
AlarmRule 조건 평가
    │
    ├─ 조건 충족 (새 알람):
    │   1. AlarmEvent 생성 (status: ACTIVE, triggeredAt: now)
    │   2. WebSocket → Next.js → 브라우저 알람 배너
    │   3. 감사 로그 기록
    │
    ├─ 조건 해제 (기존 알람):
    │   1. AlarmEvent.status = CLEARED (또는 CLEARED_UNACK)
    │   2. Acknowledge 되지 않은 알람은 CLEARED_UNACK으로 유지
    │   3. WebSocket → 브라우저 업데이트
    │
    └─ 변화 없음: 무시

알람 라이프사이클:
  ACTIVE → ACKNOWLEDGED → CLEARED
  ACTIVE → CLEARED_UNACK → ACKNOWLEDGED → (완료)
```

### 6.4 알람 확인 (Acknowledge) 워크플로우

```text
알람 발생
    │
    ▼
┌────────────────────────────────────────┐
│ 활성 알람 목록 (네비게이션 바 드롭다운)  │
│                                        │
│ 🔴 TR-1 과전류 (450A)    14:32  [확인] │
│ 🟡 CB-2 통신 두절         14:30  [확인] │
│                                        │
│ [전체 확인]           [알람 이력 보기]   │
└────────────────────────────────────────┘
    │
    │ 사용자가 [확인] 클릭
    ▼
API: POST /api/alarms/{eventId}/acknowledge
    │
    ├─ 인증 + 권한 확인 (Operator 이상)
    ├─ AlarmEvent.acknowledgedBy = userId
    ├─ AlarmEvent.acknowledgedAt = now
    ├─ 감사 로그 기록: "User X acknowledged alarm Y"
    └─ WebSocket → 알람 배너에서 제거 (조건도 해제된 경우)

아직 조건이 유지되는 알람:
    → ACKNOWLEDGED 상태로 표시 (배너에서는 제거, 목록에는 유지)
    → 조건 해제 시 자동 완료 처리
```

### 6.5 알람 이력 화면

프로젝트 관리 화면에 **알람 이력 뷰**를 정형화된 포맷으로 제공한다:

```text
┌──────────────────────────────────────────────────────────┐
│ 알람 이력                              [필터] [내보내기]  │
├──────────────────────────────────────────────────────────┤
│ 심각도 │ 알람명        │ 발생시각 │ 해제시각 │ 확인자  │ 확인시각 │
├────────┼──────────────┼─────────┼─────────┼────────┼─────────┤
│ 🔴     │ TR-1 과전류  │ 14:32   │ 14:45   │ 김운영 │ 14:33   │
│ 🟡     │ CB-2 통신두절 │ 14:30   │ 14:35   │ 이관리 │ 14:31   │
└──────────────────────────────────────────────────────────┘
```

### 6.6 AI 화면에서의 알람 활용

AI가 생성하는 SCADA 화면은 `alarms` props를 통해 알람 데이터를 수신할 수 있다. Knowledge 파일에 다음 지침을 포함한다:

```text
## 알람 연동 지침

화면 코드에서 alarms props를 사용하여 알람 상태를 시각적으로 표현하세요:

- alarms.active: 현재 활성 알람 목록
- alarms.history: 최근 알람 이력

활용 예시:
- 해당 장비에 활성 알람이 있으면 빨간 테두리 표시
- 알람 카운터 배지 표시
- 장비별 알람 상태 아이콘

알람 발생/해제/확인은 시스템이 관리하므로 화면 코드에서 직접 처리하지 마세요.
```


## 7. 화면 저장 및 프로젝트 관리

### 7.1 멀티테넌시 격리 원칙

격리 단위는 **Organization(고객사)**이다. Organization 간에는 어떤 데이터도 공유되지 않는다.

```text
격리 대상           격리 단위       메커니즘
──────────────────────────────────────────────────────────
DB 데이터          Organization   모든 쿼리에 organizationId 필터 강제
파일시스템 캐시     Project        src/screens/{projectId}/ 경로 격리
컴파일 번들         Project        .screen-builds/{projectId}/ 경로 격리
이미지 자산         Screen         DB ScreenAsset, API 서빙 시 소속 확인
태그 데이터         Project        Gateway prefix 기반 필터링
WebSocket 채널     Project        Project Channel 단위 라우팅
AI 세션/이력        Screen         화면별 대화 이력, LLM 컨텍스트 격리
Editor 상태        User×Screen    편집 잠금 (동시 편집 방지)
Runtime 화면       Project        프로젝트별 독립 렌더링 + 네비게이션
번들 서빙           Project        API에서 Organization 소속 확인 후 서빙
알람 규칙/이벤트    Project        프로젝트별 알람 격리
```

**DB 접근 규칙:** 모든 API는 Organization 스코프를 거친다. organizationId 없이 데이터를 조회하는 것은 금지.

```text
// 금지: 전체 조회 (다른 조직 데이터 노출 가능)
db.screen.findMany()

// 필수: Organization 스코프
db.screen.findMany({
  where: { project: { organizationId: session.organizationId } }
})
```

### 7.2 데이터 모델

```text
Organization (조직 = 고객사)
├── Gateway (Edge Gateway 등록)
│   ├── appId (TagBus app_id)
│   ├── tagPrefix (태그 접두사)
│   └── status (online/offline)
│
└── Project (프로젝트 = 사이트/현장)
    ├── settings (프로젝트별 설정 오버라이드)
    ├── Gateway 연결 (ProjectGateway)
    │
    ├── Screen (화면)
    │   ├── slug (URL 경로 + 빌드 캐시 파일명)
    │   ├── code (현재 버전 코드 — DB 원본)
    │   ├── version / compiledAt / isPublished
    │   ├── editingBy / editingAt (편집 잠금)
    │   ├── ScreenAsset (이미지 자산 — DB 저장)
    │   └── ScreenHistory (버전 이력)
    │
    ├── Screen Group (화면 그룹 = 폴더)
    │
    ├── Alarm Rule → TagConfig 참조
    │   └── Alarm Event (발생/해제/확인 이력)
    │
    ├── Tag Config (게이트웨이에서 자동 검색된 태그 + 메타데이터)
    │
    └── Navigation (메뉴 구조, 트리)

SystemConfig (전역 기본 설정)
AuditLog (감사 로그)
```

### 7.3 DB 스키마 (Prisma)

```prisma
// ─── 시스템 설정 ──────────────────────────────────────────────

model SystemConfig {
  key       String   @id       // "history.maxPerScreen", "rateLimit.setTagsPerSecond", ...
  value     Json                // 설정 값
  updatedAt DateTime @updatedAt
}

// 기본 설정값:
// history.maxPerScreen        = 100     (화면당 최대 버전 이력 수)
// history.maxStoragePerProjectMB = 500  (프로젝트당 최대 이력 용량 MB)
// alarm.retentionDays         = 365     (AlarmEvent 보존 기간, 초과 시 삭제)
// rateLimit.setTagsPerSecond  = 10      (사용자당 제어 명령 초당 최대 횟수)
// rateLimit.setTagsBurst      = 20      (버스트 허용)
// screen.maxCodeSizeKB        = 500     (화면 코드 최대 크기)
// asset.maxFileSizeMB         = 5       (자산 파일 최대 크기)
// aiSession.maxMessages       = 200     (세션당 최대 메시지 수, 초과 시 새 세션)

// ─── 조직/인증 ────────────────────────────────────────────────

model Organization {
  id        String    @id @default(cuid())
  name      String
  projects  Project[]
  gateways  Gateway[]
  users     UserOrganization[]
  createdAt DateTime  @default(now())
}

model User {
  id            String             @id @default(cuid())
  email         String             @unique
  name          String?
  passwordHash  String
  organizations UserOrganization[]
  createdAt     DateTime           @default(now())
}

model UserOrganization {
  userId         String
  organizationId String
  role           UserRole
  user           User         @relation(fields: [userId], references: [id])
  organization   Organization @relation(fields: [organizationId], references: [id])

  @@id([userId, organizationId])
}

enum UserRole {
  OWNER
  ADMIN
  ENGINEER
  OPERATOR
  VIEWER
}

// ─── 게이트웨이 ───────────────────────────────────────────────

model Gateway {
  id             String           @id @default(cuid())
  name           String
  appId          String           @unique
  tagPrefix      String
  organizationId String
  organization   Organization     @relation(fields: [organizationId], references: [id])
  projects       ProjectGateway[]
  status         GatewayStatus    @default(OFFLINE)
  lastSeenAt     DateTime?
  createdAt      DateTime         @default(now())
}

enum GatewayStatus {
  ONLINE
  OFFLINE
}

model ProjectGateway {
  projectId String
  gatewayId String
  project   Project @relation(fields: [projectId], references: [id])
  gateway   Gateway @relation(fields: [gatewayId], references: [id])

  @@id([projectId, gatewayId])
}

// ─── 프로젝트 ─────────────────────────────────────────────────

model Project {
  id             String           @id @default(cuid())
  name           String
  description    String?
  organizationId String
  organization   Organization     @relation(fields: [organizationId], references: [id])
  settings       Json?            // 프로젝트별 설정 오버라이드 (Zod 스키마로 유효 키/값 검증)
  // settings 예시: { "history.maxPerScreen": 50, "rateLimit.setTagsPerSecond": 5 }
  // 허용 키: SystemConfig의 key 중 프로젝트별 오버라이드 가능한 항목만
  gateways       ProjectGateway[]
  screens        Screen[]
  screenGroups   ScreenGroup[]
  tagConfigs     TagConfig[]
  alarmRules     AlarmRule[]
  navigation     Navigation[]
  createdAt      DateTime         @default(now())
  updatedAt      DateTime         @updatedAt
}

// ─── 화면 ─────────────────────────────────────────────────────

model ScreenGroup {
  id        String   @id @default(cuid())
  name      String
  order     Int      @default(0)
  projectId String
  project   Project  @relation(fields: [projectId], references: [id])
  screens   Screen[]
}

model Screen {
  id            String         @id @default(cuid())
  name          String
  slug          String                      // URL 경로 + 빌드 캐시 파일명 ({slug}.tsx)
  code          String         @db.Text     // 현재 버전 코드 (DB 원본)
  version       Int            @default(1)
  compiledAt    DateTime?
  isPublished   Boolean        @default(false)
  thumbnail     String?
  editingBy     String?                     // 현재 편집 중인 userId (잠금)
  editingAt     DateTime?                   // 편집 시작 시각 (5분 타임아웃)
  groupId       String?
  group         ScreenGroup?   @relation(fields: [groupId], references: [id])
  projectId     String
  project       Project        @relation(fields: [projectId], references: [id])
  histories     ScreenHistory[]
  sessions      AiSession[]
  assets        ScreenAsset[]
  navigations   Navigation[]
  createdAt     DateTime       @default(now())
  updatedAt     DateTime       @updatedAt

  @@unique([projectId, slug])
}

model ScreenAsset {
  id        String   @id @default(cuid())
  screenId  String
  screen    Screen   @relation(fields: [screenId], references: [id])
  filename  String                          // "transformer.svg"
  mimeType  String                          // "image/svg+xml"
  data      Bytes                           // binary content
  size      Int                             // bytes
  createdAt DateTime @default(now())

  @@unique([screenId, filename])
}

model ScreenHistory {
  id        String   @id @default(cuid())
  screenId  String
  screen    Screen   @relation(fields: [screenId], references: [id])
  code      String   @db.Text
  version   Int
  changedBy String                          // userId (String — User 삭제 후에도 이력 보존)
  message   String?
  createdAt DateTime @default(now())
}

// ─── AI 세션 ──────────────────────────────────────────────────

model AiSession {
  id        String          @id @default(cuid())
  screenId  String
  screen    Screen          @relation(fields: [screenId], references: [id])
  messages  AiMessage[]
  createdAt DateTime        @default(now())
}

model AiMessage {
  id        String          @id @default(cuid())
  sessionId String
  session   AiSession       @relation(fields: [sessionId], references: [id])
  role      String          // "user" | "assistant"
  content   String          @db.Text
  images    AiMessageImage[]
  createdAt DateTime        @default(now())
}

model AiMessageImage {
  id        String    @id @default(cuid())
  messageId String
  message   AiMessage @relation(fields: [messageId], references: [id])
  data      Bytes                           // binary content
  mimeType  String                          // "image/png"
  order     Int       @default(0)           // 동일 메시지 내 이미지 순서
  createdAt DateTime  @default(now())
}

// ─── 태그 ─────────────────────────────────────────────────────

model TagConfig {
  id         String      @id @default(cuid())
  projectId  String
  project    Project     @relation(fields: [projectId], references: [id])
  tagId      String      // e.g. "/site1/tr1/voltage"
  label      String?
  unit       String?     // "V", "A", "kW"
  min        Float?
  max        Float?
  writable   Boolean     @default(false)
  alarmRules AlarmRule[]

  @@unique([projectId, tagId])
}

// ─── 알람 ─────────────────────────────────────────────────────

model AlarmRule {
  id          String         @id @default(cuid())
  projectId   String
  project     Project        @relation(fields: [projectId], references: [id])
  tagConfigId String
  tagConfig   TagConfig      @relation(fields: [tagConfigId], references: [id])
  name        String
  condition   AlarmCondition
  threshold   Float
  deadband    Float          @default(0)
  severity    AlarmSeverity
  message     String         // "{value}A > {threshold}A" 변수 치환
  enabled     Boolean        @default(true)
  events      AlarmEvent[]
  createdAt   DateTime       @default(now())
  updatedAt   DateTime       @updatedAt
}

enum AlarmCondition {
  GT
  GTE
  LT
  LTE
  EQ
  NEQ
  DEADBAND
}

enum AlarmSeverity {
  CRITICAL
  WARNING
  INFO
}

model AlarmEvent {
  id             String           @id @default(cuid())
  alarmRuleId    String
  alarmRule      AlarmRule        @relation(fields: [alarmRuleId], references: [id])
  status         AlarmEventStatus
  triggerValue   Float
  triggeredAt    DateTime
  clearedAt      DateTime?
  acknowledgedBy String?
  acknowledgedAt DateTime?
  createdAt      DateTime         @default(now())
}

enum AlarmEventStatus {
  ACTIVE
  ACKNOWLEDGED
  CLEARED
  CLEARED_UNACK
}

// ─── 감사 로그 ────────────────────────────────────────────────

model AuditLog {
  id             String   @id @default(cuid())
  organizationId String
  userId         String   // String — User 삭제 후에도 감사 이력 보존
  action         String   // "control", "alarm_ack", "screen_edit", "screen_publish", ...
  target         String   // 대상 리소스 (tagId, screenId, alarmEventId 등)
  payload        Json?    // 상세 데이터
  ipAddress      String?
  createdAt      DateTime @default(now())

  @@index([organizationId, createdAt])
  @@index([userId, createdAt])
}

// ─── 네비게이션 ───────────────────────────────────────────────

model Navigation {
  id        String       @id @default(cuid())
  projectId String
  project   Project      @relation(fields: [projectId], references: [id])
  label     String
  screenId  String?
  screen    Screen?      @relation(fields: [screenId], references: [id])
  parentId  String?
  parent    Navigation?  @relation("NavTree", fields: [parentId], references: [id])
  children  Navigation[] @relation("NavTree")
  order     Int          @default(0)
}
```

### 7.4 화면 버전 관리

- AI가 화면을 수정할 때마다 DB 저장 전에 `ScreenHistory`에 이전 코드 백업
- 사용자가 원하면 특정 버전으로 롤백 가능 (히스토리 코드를 Screen.code에 복원 + 재컴파일)
- `isPublished` 플래그로 Editor(draft) vs Runtime(published) 구분

**배포(Publish) 워크플로우:**

```text
배포 조건 (publish 시 자동 검증):
  1. 컴파일 성공 확인 (compiledAt이 최신 version과 일치)
  2. AST 검증 통과 (금지 패턴 없음)
  → 조건 미충족 시 배포 거부 + 사유 표시

배포 상태 전이:
  draft (isPublished=false)
    │ Admin 이상이 [배포] 클릭
    ▼
  published (isPublished=true)
    │ Runtime에서 접근 가능
    │
    ├─ 편집 시: 편집은 가능하나 isPublished 유지
    │  → AI가 코드 수정 → 즉시 Runtime에도 반영 (hot update)
    │  → ScreenHistory에 이전 버전 백업
    │
    └─ 회수 시: Admin 이상이 [배포 회수] 클릭
       → isPublished=false
       → Runtime에서 접근 불가 (404)
```

**버전 이력 보존 정책:**

```text
SystemConfig 기본값 (관리자가 변경 가능):
  history.maxPerScreen = 100          화면당 최대 100개 버전 보존
  history.maxStoragePerProjectMB = 500  프로젝트당 최대 500MB

보존 규칙:
  - maxPerScreen 초과 시 가장 오래된 버전부터 삭제
  - maxStoragePerProjectMB 초과 시 가장 오래된 버전부터 삭제
  - publish된 버전은 삭제 대상에서 제외 (영구 보존)
  - 삭제는 백그라운드 작업으로 주기적 실행

프로젝트별 오버라이드:
  Project.settings = { "history.maxPerScreen": 50 }
  → 이 프로젝트는 50개까지만 보존

AlarmEvent 보존 정책:
  alarm.retentionDays = 365 (기본 1년)
  - 보존 기간 초과 시 가장 오래된 AlarmEvent부터 삭제
  - 삭제는 백그라운드 작업으로 주기적 실행 (ScreenHistory 정리와 동일)
  - 프로젝트별 오버라이드 가능
```

### 7.5 동시 편집 잠금

같은 화면을 여러 사용자가 동시에 편집하면 작업 손실 위험이 있으므로 **비관적 잠금(pessimistic lock)**을 적용한다.

```text
편집 시작 요청:
  1. Screen.editingBy 확인
     │
     ├─ null (비어있음) → 잠금 획득
     │   Screen.editingBy = userId
     │   Screen.editingAt = now
     │
     ├─ 다른 사용자 + 5분 이내 → 거부
     │   "현재 {userName}님이 편집 중입니다 (시작: {editingAt})"
     │   → 읽기 전용으로 표시
     │
     └─ 다른 사용자 + 5분 초과 → 타임아웃, 잠금 탈취
         이전 사용자의 잠금 해제 → 새 사용자에게 잠금 부여

편집 중:
  - 30초마다 heartbeat → editingAt 갱신
  - heartbeat 실패 (브라우저 종료 등) → 5분 후 자동 해제

편집 완료 / 이탈:
  - Screen.editingBy = null
  - Screen.editingAt = null
```


## 8. 인증 및 권한

### 8.1 인증

NextAuth.js 기반 인증. 초기에는 Credentials Provider (이메일/비밀번호), 추후 OAuth 확장.

### 8.2 역할 모델

```text
┌──────────────────────────────────────────────────┐
│ 역할           │ 설명                             │
├──────────────────────────────────────────────────┤
│ Owner          │ 조직 관리, 멤버 관리, 모든 권한    │
│ Admin          │ 프로젝트 관리, 화면 관리, 제어     │
│ Engineer       │ 화면 생성/편집, 제어              │
│ Operator       │ Runtime 조회, 제어 (편집 불가)    │
│ Viewer         │ Runtime 조회만 (제어 불가)        │
└──────────────────────────────────────────────────┘
```

### 8.3 권한 매트릭스

```text
기능                  Owner  Admin  Engineer  Operator  Viewer
──────────────────────────────────────────────────────────────
조직 관리               ✓
게이트웨이 등록/삭제     ✓      ✓
프로젝트 생성/삭제       ✓      ✓
프로젝트 게이트웨이 할당  ✓      ✓
화면 생성/편집           ✓      ✓      ✓
화면 배포 (publish)     ✓      ✓
Runtime 조회            ✓      ✓      ✓        ✓        ✓
설비 제어 (setTags)     ✓      ✓      ✓        ✓
태그 설정 관리           ✓      ✓      ✓
알람 규칙 관리           ✓      ✓      ✓
알람 확인 (ack)         ✓      ✓      ✓        ✓
알람 이력 조회           ✓      ✓      ✓        ✓        ✓
감사 로그 조회           ✓      ✓
시스템 설정 변경         ✓
```

### 8.4 API 접근 검증 체인

모든 API 요청은 다음 체인을 거친다:

```text
인증 (JWT / NextAuth 세션)
  │ userId
  ▼
Organization 소속 확인 (UserOrganization)
  │ organizationId, role
  ▼
Project 접근 권한 확인
  │ projectId가 해당 Organization 소속인지
  ▼
리소스 접근 (Screen, Tag, Bundle, Alarm, Asset ...)
  │ 해당 projectId 스코프 내에서만
  ▼
감사 로그 기록 (제어, 알람 확인 등 주요 동작)
  │
  ▼
응답

어느 단계에서든 실패하면 즉시 거부.
Organization 간 데이터가 교차할 수 있는 경로는 존재하지 않음.
```

### 8.5 제어 명령 보안

`setTags()`는 보안에 민감하므로 다중 검증 + 속도 제한을 거친다:

```text
AI 생성 컴포넌트: setTags({ '/acme/plant1/cb1/command': 1 })
    │
    │  React props 콜백 호출
    ▼
ScreenRenderer → Next.js API (서버 — 모든 검증은 서버 사이드에서 강제):
    │
    ▼
1. 인증 확인 (JWT/세션)
    │
    ▼
2. 권한 확인 (Operator 이상)
    │
    ▼
3. 속도 제한 (Rate Limit — 서버 사이드 강제)
   SystemConfig: rateLimit.setTagsPerSecond = 10, burst = 20
   (프로젝트별 오버라이드 가능)
   초과 시 → 429 Too Many Requests, 사용자에게 "제어 명령 속도 제한" 표시
    │
    ▼
4. 태그 소속 확인 (태그가 Project의 Gateway prefix에 속하는지)
    │
    ▼
5. 태그 쓰기 가능 여부 확인 (TagConfig.writable = true)
    │
    ▼
6. 값 범위 검증 (TagConfig.min/max)
    │
    ▼
7. 감사 로그 기록 (AuditLog)
   { action: "control", target: "/acme/plant1/cb1/command",
     payload: { value: 1 }, userId, ipAddress }
    │
    ▼
8. Bridge → tb.set_tags() → Edge Gateway → 설비
```


## 9. 제어 (Command) 설계

### 9.1 제어 컴포넌트 지침

AI가 화면을 생성할 때, Knowledge Files에 다음 제어 컴포넌트 가이드를 포함한다:

| 컴포넌트 | 용도 | 예시 |
|----------|------|------|
| Button | ON/OFF, 투입/개방 등 이산 제어 | 차단기 투입, 펌프 시작 |
| Number Input | 설정값 입력 | 목표 온도, 전압 설정 |
| Text Input | 문자열 설정 | 장비 라벨, 메모 |
| Range Slider | 범위 내 연속값 조절 | 출력 비율 (0~100%) |
| Dropdown | 다중 선택지 중 택 1 | 운전 모드 (자동/수동/정지) |
| Toggle Switch | 이진 상태 전환 | 활성화/비활성화 |
| Confirm Dialog | 위험 제어 시 2차 확인 | 비상 정지, 차단기 개방 |

### 9.2 제어 데이터 흐름

```text
사용자 버튼 클릭 (AI 생성 컴포넌트 내부)
    │
    │  setTags({ '/site1/cb1/command': 1 })  ← React props 콜백
    ▼
ScreenRenderer
    │
    │  WebSocket 전송
    ▼
Next.js API (WebSocket)
    │
    │  권한 확인 + rate limit + 값 검증 + 감사 로그
    ▼
TagBus Bridge (Python)
    │
    │  tb.set_tags({'/site1/cb1/command': 1})
    │  tb.commit()
    ▼
TagBus (eCAL)
    │
    ▼
Edge Gateway
    │
    │  프로토콜 변환 (Modbus/OPC UA 등)
    ▼
현장 설비 (차단기 투입)
    │
    │  상태 변경 피드백
    ▼
Edge Gateway → TagBus → Bridge → Next.js → ScreenRenderer (React state 업데이트)
    │
    │  tags['/site1/cb1/status'] = 1 (투입됨)
    ▼
UI 자동 갱신 (녹색 표시)
```


## 10. Knowledge 시스템

LLM의 도메인 지식을 마크다운 파일로 관리한다. 파인튜닝 없이 Prompt Engineering으로 동작.

```text
knowledge/
├── master.md              # AI 전체 동작 규칙 및 코드 생성 지침
├── scada_rules.md         # SCADA 설계 규칙 (배치, 색상, 표현)
├── tag_structure.md       # 데이터 모델 정의, 태그 네이밍 규칙
├── control_guide.md       # 제어 컴포넌트 가이드 (버튼, 입력, 슬라이더 등)
├── alarm_guide.md         # 알람 연동 지침 (alarms props 활용법)
├── responsive_guide.md    # 해상도 및 반응형 가이드라인
├── design_template.md     # UI 디자인 규칙 (색상 팔레트, 레이아웃 패턴)
└── code_api.md            # AI가 사용 가능한 API (tags, setTags, screenState 등)
```

`code_api.md`에 포함할 상태 관리 규칙:

```text
## 상태 관리 규칙

화면 컴포넌트 내부에서 useState/useReducer를 사용하지 마세요.
코드 업데이트 시 내부 state는 초기화됩니다.

대신 props로 제공되는 상태 관리자를 사용하세요:

- 실시간 설비 데이터 읽기    → tags['/site1/tr1/voltage'].v
- 설비 제어 명령             → setTags({ '/site1/cb1/command': 1 })
- 사용자 입력값 보존          → screenState.state.targetTemp
- 사용자 입력값 변경          → screenState.setState({ targetTemp: 50 })
- 알람 목록 표시             → alarms.active, alarms.history
- 차트 데이터 조회           → chartData.query(tagId, from, to)
- 사용자 설정               → userSettings.theme
- 화면 이동                 → navigate('screen-slug')
```

**반응형 가이드라인** (`responsive_guide.md`):

```text
## 해상도 및 반응형 규칙

디자인 타깃: FHD (1920×1080)

### 레이아웃 원칙
- 최상위 컨테이너: min-h-screen, w-full
- CSS Grid로 주요 레이아웃 구성 (grid-cols, grid-rows)
- Flexbox로 컴포넌트 내부 정렬
- 고정 px 대신 rem/Tailwind 유틸리티 사용

### 반응형 대응
- 기본: FHD (1920×1080) 기준 설계
- xl (1280px+): 주요 지원 범위
- 2xl (1536px+): 여유 공간 활용
- lg (1024px+): 축소 레이아웃 (패널 접기 등)
- md 이하: 미지원 (SCADA는 데스크탑/대형 모니터 전용)

### 폰트 및 데이터 표시
- 실시간 값: text-xl ~ text-3xl (한눈에 읽을 수 있는 크기)
- 라벨: text-sm ~ text-base
- 상태 표시: 색상 + 아이콘 (색맹 고려)

### SCADA 특화
- 장비 배치: grid 기반 고정 위치 (자유 배치보다 정렬 우선)
- 데이터 밀도: 한 화면에 과도한 정보 금지 (화면 분할 권장)
- 터치 대응: 제어 버튼 최소 44×44px (태블릿 사용 고려)
```

**SCADA 설계 규칙 예시** (`scada_rules.md`):

- 차단기는 선로(bus) 위에 배치
- 변압기는 버스 사이에 배치
- 전류/전압 값은 해당 장비 옆에 표시
- 과전류 시 빨간색, 정상 시 녹색
- 제어 버튼은 해당 장비 컴포넌트 내에 배치
- 위험 제어(차단기 개방 등)는 확인 다이얼로그 필수


## 11. 세션 관리

하나의 화면 생성/수정 과정은 **하나의 AI 세션**으로 관리한다.

```text
AiSession
├── User: "수배전반 화면 만들어줘" + [스케치 이미지]
├── AI: 코드 생성 → v1
├── User: [Drawing Panel 어노테이션] + "여기에 트렌드 차트 추가"
├── AI: 코드 수정 → v2
├── User: "차단기에 확인 다이얼로그 넣어줘"
└── AI: 코드 수정 → v3
```

세션 내 모든 대화가 LLM 컨텍스트로 누적되므로 맥락을 유지한 점진적 수정이 가능하다.

**AI 세션 생명주기:**

```text
세션 시작:
  편집 모드 진입 시 사용자에게 선택 제공:
  - "이전 대화 이어서" → 가장 최근 AiSession 로드
  - "새 대화 시작"    → 새 AiSession 생성 (현재 코드를 초기 컨텍스트로)

자동 새 세션:
  - 메시지 수가 aiSession.maxMessages (기본 200) 초과 시
    → 자동으로 새 세션 생성
    → 현재 코드 + 마지막 N개 메시지 요약을 초기 컨텍스트로 이관
    → 사용자에게 "새 세션으로 전환됨" 알림

세션 보존:
  - 이전 세션은 삭제하지 않음 (AiSession 목록에서 열람 가능)
  - 세션 전환 시 이전 세션의 대화 이력은 읽기 전용으로 보존
```

**AI 세션 격리:** AI 호출 시 LLM에 전달되는 모든 컨텍스트는 해당 사용자의 Organization/Project 범위로 제한된다. 화면 코드는 해당 프로젝트의 파일만, 태그 목록은 해당 프로젝트의 Gateway prefix 태그만, 알람 규칙은 해당 프로젝트의 AlarmRule만, 대화 이력은 해당 Screen의 AiSession만 포함된다. 다른 조직의 데이터가 LLM 컨텍스트에 포함될 경로는 존재하지 않는다.


## 12. 태그 시뮬레이션 (테스트 모드)

Editor에서 실제 설비 없이 AI가 생성한 화면을 테스트할 수 있는 **시뮬레이션 모드**를 제공한다.

### 12.1 시뮬레이션 개요

```text
┌─ Editor 모드 ──────────────────────────────────────────────┐
│                                                            │
│  ┌─ AI Chat Panel ──┐  ┌─ Preview ───────────────────────┐  │
│  │                  │  │                                │  │
│  │  ...             │  │  SCADA 화면                     │
│  │                  │  │  (시뮬레이션 태그값으로 렌더링)   │  │
│  ├──────────────────┤  │                                │  │
│  │ Tag Panel        │  └────────────────────────────────┘  │
│  │ [태그 목록]       │                                      │
│  ├──────────────────┤  ┌─ Tag Simulator ────────────────┐  │
│  │ [시뮬레이션 모드]  │  │                                │  │
│  │  🔘 LIVE         │  │ /tr1/voltage  [__220__] V  ◀── │  │
│  │  🔘 SIMULATE ◄   │  │ /tr1/current  [__350__] A      │  │
│  │                  │  │ /cb1/status   [__1____]        │  │
│  │                  │  │ /cb1/command  [__0____]        │  │
│  │                  │  │                                │  │
│  │                  │  │ [값 입력] [슬라이더] [프리셋]    │  │
│  │                  │  │                                │  │
│  │                  │  │ 알람 테스트:                     │  │
│  │                  │  │  /tr1/current를 450으로 설정    │  │
│  │                  │  │  → "TR-1 과전류" 알람 발생 확인  │  │
│  └──────────────────┘  └────────────────────────────────┘  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 12.2 시뮬레이션 동작

```text
LIVE 모드 (기본):
  tags → TagBus Bridge에서 실시간 수신
  알람 → Data Worker가 실시간 평가
  제어 → 실제 설비로 전달

SIMULATE 모드:
  tags → Tag Simulator에서 사용자가 수동 입력
  알람 → 브라우저에서 AlarmRule 조건 로컬 평가 (표시만, DB 저장 안 함)
  제어 → setTags 호출 시 시뮬레이션 태그값만 변경 (실제 전송 안 함)
```

**시뮬레이션 기능:**

| 기능 | 설명 |
|------|------|
| 수동 값 입력 | 태그별 값을 직접 입력하여 화면 렌더링 확인 |
| 슬라이더 | 숫자 태그를 슬라이더로 연속 조절 (min/max 범위) |
| 프리셋 | "정상 상태", "과전류 상태", "비상 정지" 등 저장된 값 세트 |
| 알람 테스트 | 값 변경 시 AlarmRule 조건을 로컬 평가하여 알람 배너 표시 |
| 제어 테스트 | setTags 호출이 시뮬레이션 태그값에 반영됨을 확인 |

시뮬레이션 모드는 **Editor에서만** 사용 가능하다. Runtime은 항상 LIVE 모드로 동작한다.


## 13. 장애 대응 및 복원력 (Resilience)

SCADA는 24/7 운영이므로 각 장애 시나리오별 복구 전략을 설계한다.

### 13.1 장애 시나리오별 대응

```text
장애                    감지 방법            대응                            사용자 체감
───────────────────────────────────────────────────────────────────────────────────────
LLM API 장애            HTTP 타임아웃        재시도 (3회, exp. backoff)       "AI 응답 지연"
                                            실패 시 이용 불가 표시           기존 화면 정상

TagBus Bridge 크래시    ping 실패 (3회)      systemd 자동 재시작 (2초)       "연결 복구 중" 배너
                                            재시작 후 구독 복구             ~5초 후 복귀

Data Worker 크래시      ping 실패 (3회)      systemd 자동 재시작 (2초)       알람 일시 중단
                                            AlarmRule 재로드               모니터링은 정상

WebSocket 끊김          heartbeat 실패       자동 재접속 (exp. backoff)      "재연결 중..."
                                            재접속 후 스냅샷 전송           구독 자동 복구

esbuild 컴파일 실패     컴파일 에러           LLM에 피드백 후 재생성 (3회)     기존 화면 유지

DB 연결 장애            커넥션 풀 에러        Prisma reconnect               API 일시 에러

시계열 DB 장애          쿼리 타임아웃         차트 "조회 불가" 표시           핵심 기능 유지
```

### 13.2 Graceful Degradation 원칙

```text
핵심 (항상 동작해야 함):
  - 실시간 태그 값 표시
  - 알람 발생/표시
  - 설비 제어 (setTags)

중요 (짧은 중단 허용):
  - 태그 구독 갱신
  - 알람 확인 (Acknowledge)
  - 화면 전환

보조 (장애 시 비활성화 가능):
  - AI 화면 생성/수정
  - 차트 데이터 조회
  - 감사 로그 기록
```


## 14. 사용자 인터페이스

### 14.1 Editor 모드

```text
┌────────────────────────────────────────────────────────────┐
│ 네비게이션 바 + 알람 배너 (정형화)                            │
│  [프로젝트명]  [화면1] [화면2] [알람 에디터]    🔔 3  ⚠ 12  │
├────────────────────────────────────────────────────────────┤
│ Editor                                                     │
│                                                            │
│  ┌──────────────────┐  ┌────────────────────────────────┐  │
│  │ AI Chat Panel    │  │ Preview + Drawing Panel        │  │
│  │                  │  │                                │  │
│  │  [텍스트 입력]    │  │  ┌────────────────────────┐   │  │
│  │                  │  │  │ Drawing Layer (tldraw) │   │  │
│  │  [이미지 첨부]    │  │  ├────────────────────────┤   │  │
│  │                  │  │  │ ScreenRenderer        │   │  │
│  │  [AI Progress]   │  │  └────────────────────────┘   │  │
│  │                  │  │                                │  │
│  │                  │  │  [드로잉 모드] [전송] [캡처]    │  │
│  ├──────────────────┤  ├────────────────────────────────┤  │
│  │ Tag Panel        │  │ Tag Simulator                  │  │
│  │ [태그 목록/값]    │  │ [LIVE / SIMULATE 모드 전환]     │  │
│  │                  │  │ [태그별 값 수동 입력]            │  │
│  │ [시뮬레이션 모드]  │  │ [프리셋] [알람 테스트]          │  │
│  └──────────────────┘  └────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

| 패널 | 역할 |
|------|------|
| 네비게이션 바 | 프로젝트 내 화면 이동, 알람 카운터/배너, 알람 에디터 접근 |
| AI Chat Panel | 자연어로 화면 생성/수정 요청. 이미지 첨부 및 AI 진행 상태 표시 |
| Tag Panel | 프로젝트에 연결된 실시간 태그 목록 및 현재 값 확인. 시뮬레이션 모드 전환 |
| Preview | 생성된 화면 실시간 미리보기 (ScreenRenderer). [캡처] 버튼으로 스크린샷 다운로드 |
| Drawing Panel | Preview 위 투명 오버레이. 드로잉 모드 ON 시 어노테이션. [전송]으로 AI에 전달 |
| Tag Simulator | 시뮬레이션 모드 시 태그값 수동 입력, 프리셋, 알람 조건 테스트 |

### 14.2 Runtime 모드

배포(publish)된 SCADA 화면을 실시간 데이터와 함께 표시하는 운영 화면.

```text
┌────────────────────────────────────────────────────────────┐
│ 네비게이션 바 + 알람 배너 (정형화)                            │
│  [프로젝트명]  [화면1] [화면2]              🔔 3  ⚠ 12     │
│  ┌────────────────────────────────────────────────────┐    │
│  │ 🔴 TR-1 과전류 (450A > 400A)  14:32  [확인]       │    │
│  └────────────────────────────────────────────────────┘    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ScreenRenderer (SCADA 화면 — 전체 화면)                     │
│                                                            │
│  실시간 데이터 표시 + 제어 버튼 + 알람 연동                    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

네비게이션 메뉴를 통해 화면 간 이동. 화면 이동은 `navigate` props를 통해 AI 생성 코드에서도 트리거 가능.

**Editor / Runtime 격리:**

```text
Editor 모드:
  - 사용자는 자기 Organization의 Project만 목록에 표시
  - Project 선택 → 해당 프로젝트의 Screen만 표시
  - AI Chat은 해당 Screen의 AiSession만 로드
  - Tag Panel은 해당 Project의 Gateway 태그만 표시
  - 편집 잠금: Screen.editingBy로 동시 편집 방지
  - 시뮬레이션 모드: LIVE/SIMULATE 전환 가능

Runtime 모드:
  - URL: /runtime/{projectId}/{screenSlug}
  - projectId 접근 권한 미들웨어에서 검증
  - 번들: /api/screens/{projectId}/{slug}/bundle.js → Organization 소속 확인 후 서빙
  - WebSocket: 해당 Project Channel만 연결
  - Navigation: 해당 Project의 메뉴만 표시
  - 알람 배너: 해당 Project의 활성 알람만 표시
  - 항상 LIVE 모드 (시뮬레이션 불가)
```

### 14.3 알람 에디터 화면

프로젝트 관리 영역에서 접근하는 정형화된 알람 규칙 편집 화면:

```text
┌──────────────────────────────────────────────────────────┐
│ 알람 규칙 관리                              [+ 규칙 추가] │
├──────────────────────────────────────────────────────────┤
│ 이름         │ 태그                   │ 조건   │ 값  │ 심각도   │ 활성 │
├─────────────┼───────────────────────┼───────┼─────┼─────────┼──────┤
│ TR-1 과전류 │ /acme/plant1/tr1/curr │ > (GT)│ 400 │ CRITICAL│  ✓   │
│ CB-2 통신끊김│ /acme/plant1/cb2/comm │ ==(EQ)│ 0   │ WARNING │  ✓   │
├──────────────────────────────────────────────────────────┤
│ 규칙 편집 폼:                                            │
│  이름:     [__________________]                          │
│  태그:     [드롭다운: 프로젝트 TagConfig에서 선택]         │
│  조건:     [드롭다운: >, >=, <, <=, ==, !=, DEADBAND]    │
│  임계값:   [________]                                    │
│  데드밴드: [________] (선택)                              │
│  심각도:   [CRITICAL / WARNING / INFO]                   │
│  메시지:   [________] ({value}, {threshold} 변수 사용)    │
│                                        [저장] [취소]     │
└──────────────────────────────────────────────────────────┘
```

### 14.4 프로젝트 관리 화면

- 프로젝트 목록 / 생성 / 설정 (프로젝트별 파라미터 오버라이드)
- 게이트웨이 등록 / 상태 모니터링 / 프로젝트 할당
- 화면 목록 (그룹별) / 검색 / 정렬
- 태그 설정 관리 (TagConfig — 게이트웨이에서 자동 검색, label/unit/writable 편집)
- 알람 규칙 관리 (AlarmRule — TagConfig 선택, 조건 설정, 심각도 지정)
- 알람 이력 조회 (AlarmEvent — 필터링, 내보내기)
- 네비게이션 메뉴 편집
- 멤버 관리 (역할 할당)
- 감사 로그 조회 (AuditLog — 제어 이력, 알람 확인 이력 등)
- 시스템 설정 (SystemConfig — Owner만 접근, 전역 파라미터 관리)


## 15. 향후 확장

- 자동 P&ID (배관 계장도) 생성
- 자동 Single Line Diagram 생성
- AI 기반 알람 분석 및 이상 감지
- AI 에너지 사용 패턴 분석
- 설비 예지 보전
- 원격 Edge Gateway 지원 (eCAL 의존성 추상화)
- 알람 에스컬레이션 (미확인 알람 자동 상위 보고)
- SMS/이메일 알람 알림
- UHD/4K 대형 모니터 최적화
