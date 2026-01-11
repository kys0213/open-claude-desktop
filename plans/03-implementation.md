# 구현 계획

테스트를 먼저 작성하고, 단순한 해결책을 선택한다.

---

## 기술 근거

### Claude Agent SDK 환경변수 지원

SDK의 `env` 옵션으로 런타임에 환경변수를 주입한다.

```typescript
query({
  prompt: message,
  options: {
    env: {
      ANTHROPIC_BASE_URL: "https://custom-api.example.com",
      HTTP_PROXY: "http://proxy.company.com:8080",
    }
  }
});
```

**출처:**
- [Agent SDK Python Reference](https://platform.claude.com/docs/en/agent-sdk/python)
- [GitHub Issue #216](https://github.com/anthropics/claude-code/issues/216)

### 환경변수

| 변수 | 용도 |
|------|------|
| `ANTHROPIC_BASE_URL` | 커스텀 API 엔드포인트 |
| `ANTHROPIC_AUTH_TOKEN` | 인증 토큰 |
| `HTTP_PROXY` / `HTTPS_PROXY` | 프록시 |

---

## IPC 스펙

### 채널 목록

| 채널 | 방향 | 입력 | 출력 |
|------|------|------|------|
| `settings:load` | R → M | - | `Result<Settings>` |
| `settings:save` | R → M | `Settings` | `Result<void>` |
| `settings:test` | R → M | `Settings` | `Result<{latency}>` |
| `chat:send` | R → M | `string` | `Result<void>` |
| `chat:chunk` | M → R | - | `string` |
| `chat:done` | M → R | - | - |
| `chat:error` | M → R | - | `{code, message}` |
| `auth:status` | R → M | - | `Result<AuthStatus>` |
| `auth:login` | R → M | - | `Result<void>` |
| `workspace:set` | R → M | `string` | `Result<void>` |
| `workspace:get` | R → M | - | `Result<string>` |

### 타입 정의

```typescript
interface Settings {
  baseUrl: string;
  proxyUrl: string;
}

interface AuthStatus {
  loggedIn: boolean;
  email?: string;
}

type Result<T> =
  | { success: true; data: T }
  | { success: false; error: { code: string; message: string } };
```

---

## Phase 1: 프로젝트 셋업

**목표:** Electron + Vite + React + TypeScript 환경 구축

**작업:**
1. package.json 생성
2. Vite 설정 (main/preload/renderer 빌드)
3. Electron 보안 설정

**완료 조건:** `npm run dev`로 빈 창이 뜬다

---

## Phase 2: 설정 저장

**목표:** Base URL, Proxy URL 저장/로드

**테스트 먼저:**
```typescript
test('설정을 저장하고 불러온다', () => {
  const settings = { baseUrl: 'https://api.example.com', proxyUrl: '' };
  saveSettings(settings);
  expect(loadSettings()).toEqual(settings);
});
```

**완료 조건:** 테스트 통과

---

## Phase 3: 설정 화면

**목표:** React 설정 입력 폼

**작업:**
1. IPC 브릿지 설정
2. SettingsPage 컴포넌트
3. 저장 버튼

**완료 조건:** 설정이 앱 재시작 후에도 유지

---

## Phase 4: 연결 테스트

**목표:** API 연결 확인

**테스트 먼저:**
```typescript
test('올바른 설정이면 연결 성공', async () => {
  const result = await testConnection({ baseUrl: validUrl });
  expect(result.success).toBe(true);
});
```

**완료 조건:** 연결 테스트 버튼이 성공/실패 표시

---

## Phase 5: 채팅

**목표:** Claude 메시지 송수신

**테스트 먼저:**
```typescript
test('메시지를 보내면 응답을 받는다', async () => {
  const response = await sendMessage('안녕', settings);
  expect(response).toBeDefined();
});
```

**완료 조건:** 메시지 송신 후 Claude 응답 표시

---

## Phase 6: 스트리밍

**목표:** 실시간 응답 표시

**테스트 먼저:**
```typescript
test('스트리밍 청크를 순서대로 받는다', async () => {
  const chunks: string[] = [];
  await sendMessageStream('안녕', settings, (chunk) => chunks.push(chunk));
  expect(chunks.length).toBeGreaterThan(1);
});
```

**핵심 코드:**
```typescript
// Main Process
for await (const event of query({ prompt, options: { env } })) {
  if (event.type === 'text') {
    webContents.send('chat:chunk', event.text);
  }
}
```

**완료 조건:** 응답이 청크 단위로 나타남

---

## Phase 7: 인증

**목표:** 로그인 상태 확인/안내

**작업:**
1. `~/.claude/` 존재 확인
2. 로그인 상태 표시
3. 미로그인 시 `claude login` 안내

**완료 조건:** 로그인 상태 표시

---

## Phase 8: A2UI 기본

**목표:** 마크다운 + 코드 블록 렌더링

**작업:**
1. EventParser 구현
2. BlockRouter 구현
3. MarkdownBlock, CodeBlock 컴포넌트

**완료 조건:** 마크다운과 코드가 올바르게 렌더링됨

---

## Phase 9: 도구 실행 UI

**목표:** 도구 호출 시각화 및 승인

**작업:**
1. ToolUseBlock 컴포넌트
2. ToolResultBlock 컴포넌트
3. Human-in-the-loop 승인 UI

**완료 조건:** 도구 호출이 표시되고 사용자가 승인/거부할 수 있음

---

## Phase 10: 워크스페이스

**목표:** 작업 디렉토리 설정

**작업:**
1. 디렉토리 선택 UI
2. 워크스페이스 상태 표시
3. SDK cwd 옵션 연동

**완료 조건:** 작업 디렉토리를 선택하고 해당 컨텍스트에서 대화 가능

---

## 성공 기준

- [ ] Base URL 설정 후 채팅 가능
- [ ] Proxy 설정 후 채팅 가능
- [ ] 연결 테스트로 설정 오류 확인 가능
- [ ] 마크다운/코드가 올바르게 렌더링됨
- [ ] 작업 디렉토리 설정 가능
