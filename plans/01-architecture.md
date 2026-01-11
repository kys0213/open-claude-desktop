# 아키텍처

## 개요

Open Claude Desktop은 Electron 기반 데스크톱 애플리케이션으로, Claude Agent SDK를 통해 AI 기능을 제공한다.

---

## 프로세스 구조

```
┌─────────────────────────────────────────────────────────────┐
│  Main Process                                                │
│  - 설정 관리 (electron-store)                                │
│  - Claude SDK 호출 (SDK Adapter)                             │
│  - 파일/네트워크 접근                                         │
│  - 도구 실행 승인 (Human-in-the-loop)                        │
├─────────────────────────────────────────────────────────────┤
│  Preload (Context Bridge)                                    │
│  - IPC 채널 화이트리스트 노출                                 │
│  - 보안 격리 유지                                            │
├─────────────────────────────────────────────────────────────┤
│  Renderer (React)                                            │
│  - UI 렌더링                                                 │
│  - A2UI Engine                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## A2UI Engine

Agent 응답을 UI로 변환하는 파이프라인.

```
SDK Events → EventParser → BlockRouter → ComponentRender → React UI
```

| 컴포넌트 | 책임 |
|----------|------|
| EventParser | SDK 이벤트 스트림을 파싱하여 블록 단위로 분리 |
| BlockRouter | 블록 타입(text, code, tool_use, tool_result)에 따라 라우팅 |
| ComponentRender | 블록을 React 컴포넌트로 렌더링 |

### 블록 타입

| 타입 | 렌더링 |
|------|--------|
| `markdown` | 마크다운 렌더러 (unified + remark) |
| `code` | 코드 블록 (Shiki 하이라이팅) |
| `tool_use` | 도구 실행 승인 UI |
| `tool_result` | 도구 결과 표시 |
| `interactive` | 차트, 폼 등 인터랙티브 컴포넌트 |

---

## SDK Adapter 패턴

SDK 의존성을 격리하여 테스트와 확장을 용이하게 한다.

```typescript
// Main Process에서 SDK 호출
query({
  prompt: message,
  options: {
    env: {
      ANTHROPIC_BASE_URL: settings.baseUrl,
      HTTP_PROXY: settings.proxyUrl,
    }
  }
});
```

인증은 `claude login` CLI에 위임하며, `~/.claude/`를 자동 참조한다.

---

## 설정 위계

우선순위: 프로젝트 설정 > 사용자 설정 > 기본값

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  .claude/       │ >  │  ~/.claude/     │ >  │  Defaults       │
│  (프로젝트)     │    │  (사용자 전역)  │    │  (하드코딩)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

| 위치 | 용도 |
|------|------|
| `~/.claude/` | 인증, 기본 Base URL, Proxy |
| `.claude/` | MCP 서버, 프로젝트별 설정 |

---

## 보안 설계

### Electron 보안

| 설정 | 값 |
|------|-----|
| `contextIsolation` | `true` |
| `nodeIntegration` | `false` |
| `sandbox` | `true` |

### 도구 실행 보안 (Human-in-the-loop)

```
SDK 도구 요청 → 승인 UI 표시 → 사용자 확인 → 도구 실행 → 결과 반환
```

민감한 도구(파일 쓰기, 삭제, 쉘 실행)는 반드시 사용자 승인을 거친다.

### 설정 저장

- electron-store 암호화 옵션 사용
- API 인증은 `~/.claude/` 참조 (SDK 위임)
