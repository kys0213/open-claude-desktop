# Open Claude Desktop - 구현 계획

## 목표

기업 환경(프록시 뒤)에서 Claude를 사용하는 사람들을 위한 데스크톱 클라이언트.
공식 Claude Desktop이 지원하지 않는 **Proxy URL / API Base URL**을 GUI로 설정한다.

## MVP 범위

1. **설정 GUI** - Base URL, Proxy 설정 입력 폼
2. **연결 테스트** - 설정이 올바른지 확인하는 버튼
3. **채팅** - Claude와 대화 (스트리밍 응답)
4. **인증** - `claude login` CLI에 위임, ~/.claude/ 자동 참조

MVP에서 제외: 멀티 프로필, Import/Export, 대화 기록 저장, 자동 업데이트

## 기술 스택

| 영역 | 선택 | 이유 |
|------|------|------|
| 프레임워크 | Electron | 크로스플랫폼 데스크톱 |
| UI | React | 익숙함, 생태계 |
| AI | Claude Agent SDK | 공식 SDK, env 옵션으로 설정 주입 가능 |
| 빌드 | Vite | 빠른 개발 환경 |

## 아키텍처

```
┌─────────────────────────────────────────┐
│              Electron                    │
├─────────────────────────────────────────┤
│  Main Process                            │
│  - 설정 저장                             │
│  - Claude SDK 호출 (env로 설정 주입)     │
├─────────────────────────────────────────┤
│  Preload (IPC Bridge)                    │
├─────────────────────────────────────────┤
│  Renderer (React)                        │
│  - 채팅 화면                             │
│  - 설정 화면                             │
└─────────────────────────────────────────┘
```

## 핵심 아이디어

SDK의 `env` 옵션으로 런타임에 Base URL과 Proxy를 동적 설정:

```typescript
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

인증은 `claude login` CLI에 위임. ~/.claude/ 자동 참조.

## 구현 순서

1. **프로젝트 셋업** - Electron + Vite + TypeScript + React
2. **설정 화면** - Base URL, Proxy 입력 폼 + 저장
3. **연결 테스트** - 설정으로 API 호출 가능한지 확인
4. **채팅 화면** - 메시지 입력 → Claude 응답 표시
5. **인증 연동** - claude login 버튼 + 상태 표시

## 성공 기준

- [ ] Base URL을 설정하고 채팅할 수 있다
- [ ] Proxy를 설정하고 채팅할 수 있다
- [ ] 연결 테스트로 설정 오류를 확인할 수 있다
