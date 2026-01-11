# 구현 계획

테스트를 먼저 작성하고, 단순한 해결책을 선택한다.

---

## 기술 근거

### Claude Agent SDK 환경변수 지원

SDK의 `env` 옵션을 통해 런타임에 환경변수를 주입할 수 있다.

```typescript
// ClaudeAgentOptions.env - dict[str, str]
query({
  prompt: message,
  options: {
    env: {
      ANTHROPIC_BASE_URL: "https://custom-api.example.com",
      HTTP_PROXY: "http://proxy.company.com:8080",
      HTTPS_PROXY: "http://proxy.company.com:8080",
    }
  }
});
```

**출처:**
- [Agent SDK Python Reference - ClaudeAgentOptions.env](https://platform.claude.com/docs/en/agent-sdk/python)
- [GitHub Issue #216: Custom Claude API Endpoint Support](https://github.com/anthropics/claude-code/issues/216) - Anthropic이 `ANTHROPIC_BASE_URL`, `HTTP_PROXY` 환경변수 지원을 확인함

### Claude Code 환경변수

| 환경변수 | 용도 |
|----------|------|
| `ANTHROPIC_BASE_URL` | 커스텀 API 엔드포인트 |
| `ANTHROPIC_AUTH_TOKEN` | 인증 토큰 |
| `HTTP_PROXY` | HTTP 프록시 |
| `HTTPS_PROXY` | HTTPS 프록시 |

**출처:**
- [Claude Code Configuration - Environment Variables](https://docs.anthropic.com/en/docs/claude-code/configuration#environment-variables)

### 인증

`claude login` 실행 시 OAuth 진행 후 `~/.claude/` 디렉토리에 인증 정보 저장. SDK가 자동으로 참조.

---

## Phase 1: 프로젝트 셋업

### 목표
Electron + Vite + React + TypeScript 개발 환경 구축

### 작업
1. package.json 생성
2. Vite 설정 (main/preload/renderer 빌드)
3. Electron 보안 설정 (contextIsolation, sandbox)

### 완료 조건
- `npm run dev`로 빈 창이 뜬다

---

## Phase 2: 설정 저장

### 목표
Base URL, Proxy URL을 저장하고 불러온다.

### 테스트 먼저
```typescript
test('설정을 저장하고 불러온다', () => {
  const settings = { baseUrl: 'https://api.example.com', proxyUrl: '' };
  saveSettings(settings);
  expect(loadSettings()).toEqual(settings);
});
```

### 완료 조건
- 설정 저장/로드 테스트 통과

---

## Phase 3: 설정 화면

### 목표
React로 설정 입력 폼을 만든다.

### 작업
1. IPC 브릿지 설정
2. SettingsPage 컴포넌트 (Base URL, Proxy 입력)
3. 저장 버튼

### 완료 조건
- 설정을 저장하면 앱 재시작 후에도 유지된다

---

## Phase 4: 연결 테스트

### 목표
입력한 설정으로 API 연결이 되는지 확인한다.

### 테스트 먼저
```typescript
test('올바른 설정이면 연결 성공', async () => {
  const result = await testConnection({ baseUrl: validUrl });
  expect(result.success).toBe(true);
});

test('잘못된 설정이면 연결 실패', async () => {
  const result = await testConnection({ baseUrl: invalidUrl });
  expect(result.success).toBe(false);
});
```

### 완료 조건
- 연결 테스트 버튼이 성공/실패를 표시한다

---

## Phase 5: 채팅

### 목표
Claude에게 메시지를 보내고 응답을 받는다.

### 테스트 먼저
```typescript
test('메시지를 보내면 응답을 받는다', async () => {
  const response = await sendMessage('안녕', settings);
  expect(response).toBeDefined();
});
```

### 핵심 코드
```typescript
async function sendMessage(message: string, settings: Settings) {
  return query({
    prompt: message,
    options: {
      env: {
        ANTHROPIC_BASE_URL: settings.baseUrl,
        HTTP_PROXY: settings.proxyUrl,
      }
    }
  });
}
```

### 완료 조건
- 메시지를 보내면 Claude 응답이 표시된다

---

## Phase 6: 스트리밍

### 목표
응답을 실시간으로 표시한다.

### 완료 조건
- 응답이 청크 단위로 나타난다

---

## Phase 7: 인증

### 목표
로그인 상태를 확인하고 안내한다.

### 작업
1. ~/.claude/ 존재 확인
2. 로그인 상태 표시
3. 미로그인 시 `claude login` 안내

### 완료 조건
- 로그인 상태가 표시된다

---

## 성공 기준

- [ ] Base URL을 설정하고 채팅할 수 있다
- [ ] Proxy를 설정하고 채팅할 수 있다
- [ ] 연결 테스트로 설정 오류를 확인할 수 있다
