---
sticker: emoji//1f527
---
# MCP 서버 만들기 가이드

> "Claude Code에 내가 만든 도구를 추가하고 싶은데 어떻게 해?" 라는 질문에서 시작해보자.

---

## Q: MCP가 뭐야?

**MCP (Model Context Protocol)**: Claude가 외부 도구를 호출할 수 있게 해주는 프로토콜이야.

```
┌──────────────┐      요청       ┌──────────────┐
│  Claude Code │  ─────────────▶ │  MCP Server  │
│              │  ◀─────────────  │  (내가 만든) │
└──────────────┘      응답       └──────────────┘
```

예를 들어:
- DB 스키마 조회
- 파일 시스템 탐색
- 외부 API 호출
- 빌드/테스트 실행

---

## Q: 어떻게 시작해?

```bash
mkdir my-mcp-server
cd my-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk
```

---

## Q: package.json은 어떻게 설정해?

```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  }
}
```

> **핵심**: `"type": "module"` 필수! ES Module import를 쓰려면.

---

## Q: 기본 템플릿 코드는?

```javascript
#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// ========== 1. 서버 초기화 ==========

const server = new Server(
  {
    name: "my-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},  // 도구 제공 기능 활성화
    },
  }
);

// ========== 2. 도구 목록 정의 ==========

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "hello",
        description: "이름을 받아서 인사를 반환합니다",
        inputSchema: {
          type: "object",
          properties: {
            name: {
              type: "string",
              description: "이름"
            }
          },
          required: ["name"]
        }
      }
    ]
  };
});

// ========== 3. 도구 실행 처리 ==========

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "hello": {
      return {
        content: [{
          type: "text",
          text: `안녕하세요, ${args.name}님!`
        }]
      };
    }

    default:
      return {
        content: [{
          type: "text",
          text: `알 수 없는 도구: ${name}`
        }]
      };
  }
});

// ========== 4. 서버 시작 ==========

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP Server running");  // 로그는 stderr로!
}

main().catch(console.error);
```

---

## Q: 도구 정의는 어떻게 해?

### inputSchema 구조

```javascript
{
  name: "도구_이름",
  description: "Claude가 도구를 선택할 때 참고하는 설명",
  inputSchema: {
    type: "object",
    properties: {
      param1: { type: "string", description: "설명" },
      param2: { type: "number", description: "설명" }
    },
    required: ["param1"]  // 필수 파라미터
  }
}
```

### 타입별 예시

```javascript
// 문자열
{ type: "string", description: "테이블명" }

// 숫자
{ type: "number", description: "개수" }

// 불리언
{ type: "boolean", description: "활성화 여부" }

// 열거형 (선택지)
{ type: "string", enum: ["option1", "option2"], description: "옵션" }

// 배열
{ type: "array", items: { type: "string" }, description: "목록" }
```

---

## Q: 결과는 어떻게 반환해?

### 텍스트 반환

```javascript
return {
  content: [{
    type: "text",
    text: "결과 문자열"
  }]
};
```

### 여러 결과 반환

```javascript
return {
  content: [
    { type: "text", text: "첫 번째 결과" },
    { type: "text", text: "두 번째 결과" }
  ]
};
```

### 에러 반환

```javascript
return {
  content: [{
    type: "text",
    text: "오류: 테이블을 찾을 수 없습니다"
  }],
  isError: true  // 에러임을 표시
};
```

---

## Q: 환경변수는 어떻게 써?

### index.js에서

```javascript
const DB_PATH = process.env.DB_PATH || "/default/path";
const API_KEY = process.env.API_KEY;
```

### .mcp.json에서

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "cmd",
      "args": ["/c", "node", "C:/path/to/index.js"],
      "env": {
        "DB_PATH": "C:/my/database",
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

---

## Q: 디버깅은 어떻게 해?

### 중요: stdout 절대 사용 금지!

```javascript
// stdout은 MCP 통신용 → 사용하면 프로토콜 깨짐!
console.log("xxx");  // 절대 금지!

// stderr로 로그 출력
console.error("디버그:", someValue);
```

### 수동 테스트

```bash
cd my-mcp-server
node index.js
# "MCP Server running" 출력되면 정상
# Ctrl+C로 종료
```

---

## Q: Claude Code에 어떻게 등록해?

### 1단계: .mcp.json 생성

프로젝트 루트 또는 `~/.mcp.json`에:

```json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "cmd",
      "args": ["/c", "node", "C:/full/path/to/index.js"]
    }
  }
}
```

> **Windows 주의**: `cmd /c` 필수!

### 2단계: Claude 설정에 추가

`~/.claude.json`에:

```json
{
  "enabledMcpjsonServers": ["my-server"]
}
```

### 3단계: Claude Code 재시작

```bash
# 재시작 후 확인
/mcp
# → my-server: connected 확인
```

---

## Q: 자주 쓰는 패턴은?

### 파일 읽기

```javascript
import fs from "fs";
import path from "path";

function readConfig() {
  const filePath = path.join(process.env.CONFIG_DIR, "config.json");
  if (!fs.existsSync(filePath)) {
    return null;
  }
  return JSON.parse(fs.readFileSync(filePath, "utf-8"));
}
```

### 캐싱

```javascript
let cache = null;

function getData() {
  if (!cache) {
    cache = loadExpensiveData();  // 최초 1회만 실행
  }
  return cache;
}
```

### 외부 API 호출

```javascript
async function fetchData(url) {
  const response = await fetch(url);
  return await response.json();
}

// 도구에서 사용
case "fetch_api": {
  const data = await fetchData("https://api.example.com/data");
  return {
    content: [{ type: "text", text: JSON.stringify(data, null, 2) }]
  };
}
```

---

## 체크리스트

### 개발 완료 후

- [ ] `npm install` 완료
- [ ] `node index.js` 실행 시 에러 없음
- [ ] `console.log` 사용하지 않음 (console.error만 사용)

### 등록 시

- [ ] `.mcp.json`에 서버 등록 (Windows: `cmd /c` 포함)
- [ ] `~/.claude.json`의 `enabledMcpjsonServers`에 서버명 추가
- [ ] Claude Code 재시작
- [ ] `/mcp`에서 `connected` 확인

---

## 흔한 문제와 해결

| 문제 | 원인 | 해결 |
|------|------|------|
| 서버 안 뜸 | cmd /c 누락 | args에 `"/c"` 추가 |
| 연결 끊김 | console.log 사용 | console.error로 변경 |
| 도구 안 보임 | enabledMcpjsonServers 누락 | ~/.claude.json에 추가 |
| 환경변수 안 읽힘 | env 설정 안 함 | .mcp.json에 env 추가 |

---

## 참고

- [MCP SDK npm](https://www.npmjs.com/package/@modelcontextprotocol/sdk)
- [MCP 공식 문서](https://modelcontextprotocol.io/)

---

## 이어서 읽기

- [[DB_SCHEMA MCP 서버 구축해보기]] - 실전 예제: DB 스키마 조회 MCP 서버

