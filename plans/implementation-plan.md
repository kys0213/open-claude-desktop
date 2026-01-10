# Open Claude Desktop - 상세 구현 계획서

## 1. 프로젝트 개요

### 1.1 목표
기업 환경(프록시 뒤)에서 Claude를 사용해야 하는 사용자들을 위한 오픈소스 데스크톱 클라이언트. 공식 Claude Desktop이 지원하지 않는 Proxy URL / API Base URL을 GUI로 간편하게 설정할 수 있도록 한다.

### 1.2 핵심 차별화
| 현재 Claude Desktop | Open Claude Desktop |
|---------------------|---------------------|
| 환경변수로만 설정 | GUI로 간편하게 설정 |
| 재시작 필요 | 런타임에 동적 변경 |
| 단일 프로필 | 멀티 프로필 지원 |
| 설정 공유 불가 | 팀 설정 Import/Export |

### 1.3 대상 사용자
- 기업 프록시 뒤에서 근무하는 개발자
- 사설 API 엔드포인트를 사용하는 팀
- 보안 정책상 직접 인터넷 접속이 불가능한 환경의 사용자

---

## 2. 기술 스택 상세

### 2.1 핵심 기술
| 카테고리 | 기술 | 버전 | 비고 |
|---------|------|------|------|
| 언어 | TypeScript | 5.x | 타입 안정성 |
| 프레임워크 | Electron | 28.x+ | 크로스플랫폼 데스크톱 |
| AI SDK | @anthropic-ai/claude-agent-sdk | latest | Claude 통합 |
| UI 프레임워크 | React | 18.x | 컴포넌트 기반 UI |
| 상태관리 | Zustand | 4.x | 경량 상태관리 |
| 빌드 도구 | Vite | 5.x | 빠른 개발 환경 |
| 패키지 매니저 | pnpm | 8.x | 효율적인 의존성 관리 |

### 2.2 보안 관련
| 카테고리 | 기술 | 용도 |
|---------|------|------|
| 키 저장 | keytar | OS keychain 통합 |
| IPC 보안 | electron-context-bridge | 안전한 IPC |

### 2.3 개발 도구
| 카테고리 | 기술 | 용도 |
|---------|------|------|
| 테스트 | Vitest | 유닛 테스트 |
| E2E 테스트 | Playwright | 통합 테스트 |
| 린터 | ESLint + Prettier | 코드 품질 |
| 빌드/배포 | electron-builder | 크로스플랫폼 배포 |

---

## 3. 아키텍처 설계

### 3.1 프로세스 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    Electron Application                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    Main Process                          │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │ │
│  │  │   Update    │  │   Proxy     │  │  Secure Storage │  │ │
│  │  │  Manager    │  │  Manager    │  │  (OS Keychain)  │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │ │
│  │  │   Config    │  │   Claude    │  │     Profile     │  │ │
│  │  │  Service    │  │   Agent     │  │     Manager     │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                                │
│                         IPC Bridge                            │
│                              │                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   Preload Script                         │ │
│  │         (Whitelisted IPC channels only)                  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                                │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                  Renderer Process                        │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐  │ │
│  │  │   Chat UI   │  │  Settings   │  │    Profile      │  │ │
│  │  │             │  │     UI      │  │   Switcher      │  │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 데이터 흐름

```
User Input → Renderer → IPC (Preload) → Main Process
                                              │
                                    ┌─────────┴─────────┐
                                    │                   │
                              Claude Agent SDK    Config Store
                                    │                   │
                                    └─────────┬─────────┘
                                              │
Main Process → IPC (Preload) → Renderer → UI Update
```

### 3.3 보안 모델

```
┌────────────────────────────────────────┐
│            Security Layers              │
├────────────────────────────────────────┤
│  1. OS Keychain (API Keys)             │
│     - macOS: Keychain                  │
│     - Windows: DPAPI                   │
│     - Linux: libsecret                 │
├────────────────────────────────────────┤
│  2. Electron Security                   │
│     - nodeIntegration: false           │
│     - contextIsolation: true           │
│     - sandbox: true                    │
├────────────────────────────────────────┤
│  3. Content Security Policy            │
│     - script-src 'self'                │
│     - connect-src 'self' + API URLs    │
│     - No inline scripts                │
├────────────────────────────────────────┤
│  4. IPC Whitelist                      │
│     - Explicit channel enumeration     │
│     - Type-safe message passing        │
└────────────────────────────────────────┘
```

---

## 4. 디렉토리 구조

```
open-claude-desktop/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml              # CI 파이프라인
│   │   ├── release.yml         # 릴리즈 빌드
│   │   └── security-audit.yml  # 보안 감사
│   └── ISSUE_TEMPLATE/
│
├── plans/                       # 계획 문서
│   └── implementation-plan.md
│
├── src/
│   ├── main/                    # Main Process
│   │   ├── index.ts            # 엔트리포인트
│   │   ├── window.ts           # 윈도우 관리
│   │   ├── ipc/
│   │   │   ├── handlers.ts     # IPC 핸들러 등록
│   │   │   └── channels.ts     # 채널 상수 정의
│   │   ├── services/
│   │   │   ├── claude-agent.ts     # Claude SDK 래퍼
│   │   │   ├── config.ts           # 설정 관리
│   │   │   ├── secure-storage.ts   # Keychain 연동
│   │   │   ├── proxy.ts            # 프록시 설정
│   │   │   └── profile.ts          # 프로필 관리
│   │   └── utils/
│   │       ├── logger.ts
│   │       └── platform.ts
│   │
│   ├── preload/                 # Preload Scripts
│   │   ├── index.ts            # Context Bridge
│   │   └── api.ts              # Exposed APIs
│   │
│   ├── renderer/                # Renderer Process (React)
│   │   ├── index.html
│   │   ├── main.tsx            # React 엔트리
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── chat/
│   │   │   │   ├── ChatWindow.tsx
│   │   │   │   ├── MessageBubble.tsx
│   │   │   │   └── InputArea.tsx
│   │   │   ├── settings/
│   │   │   │   ├── SettingsPanel.tsx
│   │   │   │   ├── ProxySettings.tsx
│   │   │   │   ├── ApiSettings.tsx
│   │   │   │   └── ProfileManager.tsx
│   │   │   └── common/
│   │   │       ├── Button.tsx
│   │   │       └── Input.tsx
│   │   ├── hooks/
│   │   │   ├── useChat.ts
│   │   │   ├── useConfig.ts
│   │   │   └── useProfile.ts
│   │   ├── stores/
│   │   │   ├── chatStore.ts
│   │   │   └── configStore.ts
│   │   └── styles/
│   │       └── global.css
│   │
│   └── shared/                  # 공유 타입/상수
│       ├── types/
│       │   ├── config.ts
│       │   ├── message.ts
│       │   └── profile.ts
│       └── constants/
│           └── ipc-channels.ts
│
├── resources/                   # 앱 리소스
│   ├── icons/
│   └── tray/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── scripts/
│   ├── build.ts
│   └── notarize.ts             # macOS 공증
│
├── electron.vite.config.ts
├── electron-builder.config.ts
├── package.json
├── tsconfig.json
├── .eslintrc.cjs
├── .prettierrc
└── README.md
```

---

## 5. 단계별 구현 계획

### Phase 1: 프로젝트 기반 구축

#### 1.1 초기 설정
- [ ] Electron + Vite + TypeScript 프로젝트 초기화
- [ ] ESLint, Prettier 설정
- [ ] 디렉토리 구조 생성
- [ ] Git hooks 설정 (husky + lint-staged)

#### 1.2 기본 Electron 구조
- [ ] Main Process 엔트리포인트
- [ ] BrowserWindow 생성 및 설정
- [ ] Preload 스크립트 (보안 설정 포함)
- [ ] 개발 환경 HMR 설정

#### 1.3 보안 기본 설정
- [ ] nodeIntegration: false
- [ ] contextIsolation: true
- [ ] sandbox: true
- [ ] 기본 CSP 헤더

### Phase 2: 핵심 서비스 구현

#### 2.1 설정 관리 서비스
```typescript
// src/main/services/config.ts
interface AppConfig {
  profiles: Profile[];
  activeProfileId: string;
  general: {
    autoUpdate: boolean;
    startMinimized: boolean;
    language: string;
  };
}

interface Profile {
  id: string;
  name: string;
  proxy: ProxyConfig;
  api: ApiConfig;
}

interface ProxyConfig {
  enabled: boolean;
  type: 'http' | 'https' | 'socks4' | 'socks5';
  host: string;
  port: number;
  auth?: {
    username: string;
    // password는 keychain에 저장
  };
  bypassList: string[];
}

interface ApiConfig {
  baseUrl: string;
  customCaBundle?: string;
  timeout: number;
}
```

#### 2.2 인증 서비스 (CLI 위임)
```typescript
// src/main/services/auth.ts
import { spawn, exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

/**
 * 인증은 Claude CLI에 완전히 위임
 * ~/.claude/ 디렉토리의 credentials를 SDK가 자동 참조
 */
class AuthService {
  // OAuth 로그인 (Pro/Max 구독자)
  async login(): Promise<void> {
    return new Promise((resolve, reject) => {
      const child = spawn('claude', ['login'], {
        stdio: 'inherit'  // 브라우저 열림 → OAuth 진행
      });
      child.on('close', (code) => {
        code === 0 ? resolve() : reject(new Error('Login failed'));
      });
    });
  }

  // 로그아웃
  async logout(): Promise<void> {
    await execAsync('claude logout');
  }

  // 인증 상태 확인
  async getAuthStatus(): Promise<AuthStatus> {
    try {
      const { stdout } = await execAsync('claude auth status --json');
      return JSON.parse(stdout);
    } catch {
      return { authenticated: false };
    }
  }

  // API 키로 로그인 (대안)
  async loginWithApiKey(apiKey: string): Promise<void> {
    // ANTHROPIC_API_KEY 환경변수로 설정하거나
    // claude config set apiKey 명령 사용
    await execAsync(`claude config set apiKey "${apiKey}"`);
  }
}

interface AuthStatus {
  authenticated: boolean;
  type?: 'oauth' | 'api_key' | 'bedrock' | 'vertex';
  email?: string;
  plan?: 'free' | 'pro' | 'max';
}
```

#### 2.3 프록시 비밀번호 저장 (최소화된 Keychain 사용)
```typescript
// src/main/services/secure-storage.ts
import keytar from 'keytar';

/**
 * Keychain은 프록시 비밀번호만 저장
 * API 키/OAuth 토큰은 ~/.claude/에서 CLI가 관리
 */
class SecureStorage {
  private readonly SERVICE_NAME = 'open-claude-desktop';

  async setProxyPassword(profileId: string, password: string): Promise<void> {
    await keytar.setPassword(this.SERVICE_NAME, `proxy:${profileId}`, password);
  }

  async getProxyPassword(profileId: string): Promise<string | null> {
    return keytar.getPassword(this.SERVICE_NAME, `proxy:${profileId}`);
  }

  async deleteProxyPassword(profileId: string): Promise<boolean> {
    return keytar.deletePassword(this.SERVICE_NAME, `proxy:${profileId}`);
  }
}
```

#### 2.3 프록시 관리 서비스
```typescript
// src/main/services/proxy.ts
class ProxyManager {
  async applyProxyConfig(config: ProxyConfig): Promise<void>;
  async testConnection(config: ProxyConfig): Promise<ConnectionTestResult>;
  async detectSystemProxy(): Promise<ProxyConfig | null>;
  async validatePacUrl(url: string): Promise<boolean>;
}

interface ConnectionTestResult {
  success: boolean;
  latencyMs?: number;
  error?: string;
}
```

#### 2.4 Claude Agent 서비스 (핵심)
```typescript
// src/main/services/claude-agent.ts
import { query } from '@anthropic-ai/claude-agent-sdk';

class ClaudeService {
  async sendMessage(content: string, profile: Profile): Promise<AsyncIterable<StreamEvent>> {
    const apiKey = await secureStorage.getApiKey(profile.id);

    return query({
      prompt: content,
      options: {
        env: {
          ANTHROPIC_BASE_URL: profile.api.baseUrl,
          ANTHROPIC_AUTH_TOKEN: apiKey,
          // 프록시 설정
          HTTP_PROXY: this.buildProxyUrl(profile.proxy),
          HTTPS_PROXY: this.buildProxyUrl(profile.proxy),
          // Custom CA
          NODE_EXTRA_CA_CERTS: profile.api.customCaBundle,
        }
      }
    });
  }

  private buildProxyUrl(config: ProxyConfig): string | undefined {
    if (!config.enabled) return undefined;
    const auth = config.auth ? `${config.auth.username}:PASSWORD@` : '';
    return `${config.type}://${auth}${config.host}:${config.port}`;
  }
}
```

### Phase 3: IPC 통신 구현

#### 3.1 IPC 채널 정의
```typescript
// src/shared/constants/ipc-channels.ts
export const IPC_CHANNELS = {
  // Config
  CONFIG_GET: 'config:get',
  CONFIG_SET: 'config:set',

  // Profile
  PROFILE_LIST: 'profile:list',
  PROFILE_CREATE: 'profile:create',
  PROFILE_UPDATE: 'profile:update',
  PROFILE_DELETE: 'profile:delete',
  PROFILE_SWITCH: 'profile:switch',
  PROFILE_EXPORT: 'profile:export',
  PROFILE_IMPORT: 'profile:import',

  // Secure Storage
  API_KEY_SET: 'secure:api-key:set',
  API_KEY_TEST: 'secure:api-key:test',

  // Proxy
  PROXY_TEST: 'proxy:test',
  PROXY_DETECT: 'proxy:detect',

  // Chat
  CHAT_SEND: 'chat:send',
  CHAT_STREAM: 'chat:stream',
  CHAT_CANCEL: 'chat:cancel',

  // App
  APP_VERSION: 'app:version',
  APP_UPDATE_CHECK: 'app:update:check',
} as const;
```

#### 3.2 Preload API
```typescript
// src/preload/api.ts
export interface ElectronAPI {
  config: {
    get: () => Promise<AppConfig>;
    set: (config: Partial<AppConfig>) => Promise<void>;
  };
  profile: {
    list: () => Promise<Profile[]>;
    create: (profile: Omit<Profile, 'id'>) => Promise<Profile>;
    update: (id: string, updates: Partial<Profile>) => Promise<Profile>;
    delete: (id: string) => Promise<void>;
    switch: (id: string) => Promise<void>;
    export: (ids: string[]) => Promise<string>; // JSON string
    import: (json: string) => Promise<Profile[]>;
  };
  secure: {
    setApiKey: (profileId: string, apiKey: string) => Promise<void>;
    testApiKey: (profileId: string) => Promise<boolean>;
  };
  proxy: {
    test: (config: ProxyConfig) => Promise<ConnectionTestResult>;
    detect: () => Promise<ProxyConfig | null>;
  };
  chat: {
    send: (message: string) => Promise<void>;
    onStream: (callback: (event: StreamEvent) => void) => () => void;
    cancel: () => Promise<void>;
  };
}
```

### Phase 4: UI 구현

#### 4.1 Chat UI
- [ ] 메시지 목록 컴포넌트
- [ ] 스트리밍 응답 표시
- [ ] Markdown 렌더링
- [ ] 코드 하이라이팅
- [ ] 메시지 복사/편집

#### 4.2 Settings UI
- [ ] 프로필 관리 패널
- [ ] 프록시 설정 폼
- [ ] API 설정 폼 (Base URL 입력)
- [ ] 연결 테스트 버튼
- [ ] Import/Export 기능

#### 4.3 UX 개선
- [ ] 로딩 상태 표시
- [ ] 에러 핸들링 및 토스트
- [ ] 키보드 단축키
- [ ] 다크/라이트 테마

### Phase 5: 기업 환경 지원

#### 5.1 Custom CA 지원
```typescript
// Custom CA 번들 지원
async setCustomCaBundle(profileId: string, caPath: string): Promise<void> {
  // 파일 존재 확인
  // 유효한 인증서인지 검증
  // 설정에 저장
}
```

#### 5.2 배포 옵션
- [ ] macOS: DMG + PKG (MDM 지원)
- [ ] Windows: NSIS + MSI (Group Policy 지원)
- [ ] Linux: AppImage + DEB + RPM

#### 5.3 관리자 설정
```json
// 관리자 배포 설정 예시 (엔터프라이즈)
{
  "autoUpdate": false,
  "lockProxySettings": true,
  "defaultProxy": {
    "type": "http",
    "host": "proxy.company.com",
    "port": 8080
  },
  "allowedBaseUrls": [
    "https://api.anthropic.com",
    "https://claude.company.com"
  ]
}
```

### Phase 6: 테스트 및 품질

#### 6.1 유닛 테스트
- [ ] 서비스 레이어 테스트
- [ ] 유틸리티 함수 테스트
- [ ] 상태 관리 테스트

#### 6.2 통합 테스트
- [ ] IPC 통신 테스트
- [ ] 설정 저장/로드 테스트
- [ ] 프록시 연결 테스트

#### 6.3 E2E 테스트
- [ ] 전체 채팅 플로우
- [ ] 프로필 전환
- [ ] 설정 변경

### Phase 7: 배포 준비

#### 7.1 빌드 설정
```typescript
// electron-builder.config.ts
export default {
  appId: 'com.open-claude-desktop.app',
  productName: 'Open Claude Desktop',

  mac: {
    target: ['dmg', 'pkg'],
    hardenedRuntime: true,
    notarize: true,
    category: 'public.app-category.productivity',
  },

  win: {
    target: ['nsis', 'msi'],
    certificateFile: process.env.WIN_CERT_FILE,
  },

  linux: {
    target: ['AppImage', 'deb', 'rpm'],
    category: 'Utility',
  },

  publish: {
    provider: 'github',
    releaseType: 'release',
  },
};
```

#### 7.2 자동 업데이트
```typescript
// 옵트아웃 가능한 자동 업데이트
class UpdateManager {
  async checkForUpdates(): Promise<UpdateInfo | null>;
  async downloadUpdate(): Promise<void>;
  async installUpdate(): Promise<void>;

  // 기업 환경용
  setAutoUpdateEnabled(enabled: boolean): void;
}
```

---

## 6. 보안 고려사항

### 6.1 민감 정보 보호

| 데이터 | 저장 위치 | 보호 방법 |
|--------|----------|----------|
| API 키 | OS Keychain | keytar 라이브러리 |
| 프록시 비밀번호 | OS Keychain | keytar 라이브러리 |
| 일반 설정 | 로컬 JSON | 파일 권한 제한 |
| 채팅 기록 | 로컬 SQLite | 선택적 암호화 |

### 6.2 Electron 보안 체크리스트

- [x] nodeIntegration: false
- [x] contextIsolation: true
- [x] sandbox: true
- [x] webSecurity: true
- [x] allowRunningInsecureContent: false
- [x] experimentalFeatures: false
- [x] enableBlinkFeatures: 비활성화
- [x] 명시적 IPC 채널만 허용

### 6.3 CSP 설정

```typescript
// Content Security Policy
const CSP = `
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.anthropic.com ${customBaseUrls};
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
`.replace(/\n/g, ' ');
```

### 6.4 취약점 방지

1. **XSS 방지**: React 기본 이스케이핑 + DOMPurify (마크다운용)
2. **Prototype Pollution**: Object.freeze 사용
3. **Path Traversal**: 경로 정규화 및 검증
4. **Dependency 취약점**: dependabot + npm audit

---

## 7. 주요 의존성 목록

### 7.1 Runtime Dependencies

```json
{
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "^1.x",
    "electron-store": "^8.x",
    "keytar": "^7.x",
    "react": "^18.x",
    "react-dom": "^18.x",
    "zustand": "^4.x",
    "react-markdown": "^9.x",
    "prism-react-renderer": "^2.x",
    "dompurify": "^3.x",
    "electron-updater": "^6.x",
    "winston": "^3.x"
  }
}
```

### 7.2 Development Dependencies

```json
{
  "devDependencies": {
    "electron": "^28.x",
    "electron-builder": "^24.x",
    "electron-vite": "^2.x",
    "vite": "^5.x",
    "typescript": "^5.x",
    "@types/react": "^18.x",
    "@types/node": "^20.x",
    "vitest": "^1.x",
    "playwright": "^1.x",
    "eslint": "^8.x",
    "prettier": "^3.x",
    "husky": "^9.x",
    "lint-staged": "^15.x"
  }
}
```

---

## 8. 마일스톤 및 타임라인

| 마일스톤 | 기간 | 목표 |
|---------|------|------|
| M1: Foundation | Week 1 | 프로젝트 구조, 기본 Electron 앱 |
| M2: Core Services | Week 2-3 | 설정, 보안 저장소, 프록시 서비스 |
| M3: IPC & Integration | Week 4 | IPC 통신, Claude SDK 통합 |
| M4: UI | Week 5-6 | Chat UI, Settings UI |
| M5: Enterprise | Week 7 | 기업 환경 기능 |
| M6: QA | Week 8 | 테스트, 버그 수정 |
| M7: Release | Week 9 | 배포 준비, 문서화 |

---

## 9. 리스크 및 대응 방안

| 리스크 | 확률 | 영향 | 대응 방안 |
|--------|------|------|----------|
| Claude SDK API 변경 | 중 | 높음 | SDK 버전 고정, 래퍼 레이어 |
| OS Keychain 접근 실패 | 낮음 | 높음 | 폴백 암호화 저장소 |
| 프록시 호환성 이슈 | 중 | 중 | 광범위한 테스트, 디버그 로깅 |
| Electron 보안 취약점 | 낮음 | 높음 | 신속한 업데이트, 보안 감사 |

---

## 10. 향후 확장 계획

### 10.1 v1.1 (예정)
- 대화 기록 저장 및 검색
- 커스텀 시스템 프롬프트

### 10.2 v1.2 (예정)
- MCP (Model Context Protocol) 지원
- 플러그인 시스템

### 10.3 v2.0 (예정)
- 팀 협업 기능
- 클라우드 동기화 (선택적)

---

## 11. Critical Files for Implementation

구현 시 가장 중요한 파일들:

1. **src/main/services/claude-agent.ts** - Claude SDK 래핑 및 환경변수 주입 핵심 로직
2. **src/main/services/proxy.ts** - 프록시 설정 관리 및 연결 테스트 (프로젝트 핵심 차별화)
3. **src/main/services/secure-storage.ts** - OS Keychain 연동으로 API 키 보안 저장
4. **src/preload/api.ts** - IPC 보안 브릿지, Renderer와 Main 간 안전한 통신
5. **src/shared/types/config.ts** - 전체 앱에서 사용하는 설정/프로필 타입 정의

---

## 12. 기술적 핵심 발견사항

### Claude Agent SDK Base URL 설정 방법

SDK에 직접적인 `baseUrl` 옵션은 없지만, `env` 옵션을 통해 환경변수로 전달 가능:

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk';

for await (const message of query({
  prompt: "Hello",
  options: {
    env: {
      ANTHROPIC_BASE_URL: "https://your-proxy.example.com",
      ANTHROPIC_AUTH_TOKEN: "your-api-key",
      HTTP_PROXY: "http://proxy.company.com:8080",
      HTTPS_PROXY: "http://proxy.company.com:8080",
    }
  }
})) {
  console.log(message);
}
```

이를 통해 런타임에 동적으로 Base URL과 프록시를 변경할 수 있어, GUI 설정 기능 구현이 가능함.
