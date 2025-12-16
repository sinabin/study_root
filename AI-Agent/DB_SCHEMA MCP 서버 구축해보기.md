---
sticker: emoji//1f5c3-fe0f
---
# DB Schema MCP 서버 구축하기

> "Claude Code에서 DB 스키마 조회할 수 있으면 좋겠는데?" 라는 질문에서 시작해보자.

---

## Q: 뭘 만들거야?

Claude Code에서 **DB 스키마를 조회**할 수 있는 MCP 서버야.

```
사용자: "USERS 테이블 구조 알려줘"
                │
                ▼
┌──────────────────────────────────────────┐
│            Claude Code                    │
│  "db_schema 도구를 호출할게요"            │
└────────────────┬─────────────────────────┘
                 │ MCP 호출
                 ▼
┌──────────────────────────────────────────┐
│        DB Schema MCP Server              │
│  1. schema.json 읽기                     │
│  2. USERS 테이블 정보 추출               │
│  3. 결과 반환                            │
└──────────────────────────────────────────┘
```

---

## Q: 왜 DB 직접 연결 안 해?

**보안** 때문이야.

| 방식 | 장점 | 단점 |
|------|------|------|
| DB 직접 연결 | 실시간 | 보안 위험, 권한 관리 복잡 |
| **JSON 파일** | 안전, 단순 | 동기화 필요 |

회사에서 쓸 거면 **JSON 파일 방식**이 안전해.

---

## Q: 프로젝트 구조는?

```
db-schema-mcp/
├── index.js           # MCP 서버 코드
├── package.json
└── data/
    └── schema.json    # DB 스키마 정보
```

---

## Q: schema.json은 어떻게 생겼어?

```json
{
  "database": "my_project",
  "tables": {
    "USERS": {
      "description": "사용자 정보 테이블",
      "columns": [
        {
          "name": "USER_ID",
          "type": "VARCHAR2(20)",
          "nullable": false,
          "description": "사용자 ID (PK)"
        },
        {
          "name": "USER_NM",
          "type": "VARCHAR2(100)",
          "nullable": false,
          "description": "사용자명"
        },
        {
          "name": "EMAIL",
          "type": "VARCHAR2(200)",
          "nullable": true,
          "description": "이메일"
        },
        {
          "name": "RGST_DT",
          "type": "DATE",
          "nullable": false,
          "description": "등록일시"
        }
      ],
      "primaryKey": ["USER_ID"],
      "indexes": ["IDX_USERS_EMAIL"]
    },
    "ORDERS": {
      "description": "주문 정보 테이블",
      "columns": [
        {
          "name": "ORDER_ID",
          "type": "NUMBER(10)",
          "nullable": false,
          "description": "주문 ID (PK)"
        },
        {
          "name": "USER_ID",
          "type": "VARCHAR2(20)",
          "nullable": false,
          "description": "사용자 ID (FK → USERS)"
        },
        {
          "name": "ORDER_AMT",
          "type": "NUMBER(15,2)",
          "nullable": false,
          "description": "주문 금액"
        }
      ],
      "primaryKey": ["ORDER_ID"],
      "foreignKeys": [
        {
          "column": "USER_ID",
          "references": "USERS.USER_ID"
        }
      ]
    }
  }
}
```

---

## Q: index.js 코드는?

```javascript
#!/usr/bin/env node

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import fs from "fs";
import path from "path";

// ========== 설정 ==========

const SCHEMA_PATH = process.env.SCHEMA_PATH || "./data/schema.json";

// ========== 스키마 로드 ==========

let schemaData = null;

function loadSchema() {
  if (!schemaData) {
    const fullPath = path.resolve(SCHEMA_PATH);
    if (!fs.existsSync(fullPath)) {
      throw new Error(`스키마 파일을 찾을 수 없습니다: ${fullPath}`);
    }
    schemaData = JSON.parse(fs.readFileSync(fullPath, "utf-8"));
    console.error(`스키마 로드 완료: ${Object.keys(schemaData.tables).length}개 테이블`);
  }
  return schemaData;
}

// ========== 서버 초기화 ==========

const server = new Server(
  {
    name: "db-schema-mcp",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// ========== 도구 정의 ==========

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "list_tables",
        description: "데이터베이스의 모든 테이블 목록을 반환합니다",
        inputSchema: {
          type: "object",
          properties: {},
          required: []
        }
      },
      {
        name: "get_table_schema",
        description: "특정 테이블의 스키마(컬럼, PK, FK 등)를 반환합니다",
        inputSchema: {
          type: "object",
          properties: {
            tableName: {
              type: "string",
              description: "테이블명 (예: USERS)"
            }
          },
          required: ["tableName"]
        }
      },
      {
        name: "search_columns",
        description: "컬럼명으로 테이블을 검색합니다",
        inputSchema: {
          type: "object",
          properties: {
            columnName: {
              type: "string",
              description: "검색할 컬럼명 (부분 일치)"
            }
          },
          required: ["columnName"]
        }
      }
    ]
  };
});

// ========== 도구 실행 ==========

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;
  const schema = loadSchema();

  switch (name) {
    // 테이블 목록 조회
    case "list_tables": {
      const tables = Object.entries(schema.tables).map(([name, info]) => ({
        name,
        description: info.description,
        columnCount: info.columns.length
      }));

      return {
        content: [{
          type: "text",
          text: `## 테이블 목록 (${tables.length}개)\n\n` +
            tables.map(t => `- **${t.name}**: ${t.description} (${t.columnCount}개 컬럼)`).join("\n")
        }]
      };
    }

    // 테이블 스키마 조회
    case "get_table_schema": {
      const tableName = args.tableName.toUpperCase();
      const table = schema.tables[tableName];

      if (!table) {
        return {
          content: [{
            type: "text",
            text: `테이블을 찾을 수 없습니다: ${tableName}`
          }],
          isError: true
        };
      }

      let result = `## ${tableName}\n\n`;
      result += `**설명**: ${table.description}\n\n`;
      result += `### 컬럼\n\n`;
      result += `| 컬럼명 | 타입 | NULL | 설명 |\n`;
      result += `|--------|------|------|------|\n`;

      for (const col of table.columns) {
        const nullable = col.nullable ? "O" : "X";
        result += `| ${col.name} | ${col.type} | ${nullable} | ${col.description} |\n`;
      }

      if (table.primaryKey) {
        result += `\n**PK**: ${table.primaryKey.join(", ")}\n`;
      }

      if (table.foreignKeys) {
        result += `\n**FK**: \n`;
        for (const fk of table.foreignKeys) {
          result += `- ${fk.column} → ${fk.references}\n`;
        }
      }

      return {
        content: [{ type: "text", text: result }]
      };
    }

    // 컬럼 검색
    case "search_columns": {
      const searchTerm = args.columnName.toUpperCase();
      const results = [];

      for (const [tableName, table] of Object.entries(schema.tables)) {
        for (const col of table.columns) {
          if (col.name.toUpperCase().includes(searchTerm)) {
            results.push({
              table: tableName,
              column: col.name,
              type: col.type,
              description: col.description
            });
          }
        }
      }

      if (results.length === 0) {
        return {
          content: [{
            type: "text",
            text: `"${args.columnName}" 컬럼을 찾을 수 없습니다.`
          }]
        };
      }

      let result = `## "${args.columnName}" 검색 결과 (${results.length}개)\n\n`;
      result += `| 테이블 | 컬럼 | 타입 | 설명 |\n`;
      result += `|--------|------|------|------|\n`;

      for (const r of results) {
        result += `| ${r.table} | ${r.column} | ${r.type} | ${r.description} |\n`;
      }

      return {
        content: [{ type: "text", text: result }]
      };
    }

    default:
      return {
        content: [{
          type: "text",
          text: `알 수 없는 도구: ${name}`
        }],
        isError: true
      };
  }
});

// ========== 서버 시작 ==========

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("DB Schema MCP Server running");
}

main().catch(console.error);
```

---

## Q: package.json은?

```json
{
  "name": "db-schema-mcp",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  }
}
```

---

## Q: 어떻게 등록해?

### .mcp.json

```json
{
  "mcpServers": {
    "db-schema": {
      "type": "stdio",
      "command": "cmd",
      "args": ["/c", "node", "C:/path/to/db-schema-mcp/index.js"],
      "env": {
        "SCHEMA_PATH": "C:/path/to/db-schema-mcp/data/schema.json"
      }
    }
  }
}
```

### ~/.claude.json

```json
{
  "enabledMcpjsonServers": ["db-schema"]
}
```

---

## Q: 어떻게 쓰면 돼?

Claude Code에서 이렇게 물어보면:

```
"USERS 테이블 구조 알려줘"
```

Claude가 자동으로:

```
db_schema MCP 서버의 get_table_schema 도구를 호출합니다.

## USERS

**설명**: 사용자 정보 테이블

### 컬럼

| 컬럼명 | 타입 | NULL | 설명 |
|--------|------|------|------|
| USER_ID | VARCHAR2(20) | X | 사용자 ID (PK) |
| USER_NM | VARCHAR2(100) | X | 사용자명 |
| EMAIL | VARCHAR2(200) | O | 이메일 |
| RGST_DT | DATE | X | 등록일시 |

**PK**: USER_ID
```

---

## Q: schema.json 자동 생성은?

실제 DB에서 스키마를 추출하는 스크립트 예시:

### Oracle

```sql
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    DATA_TYPE,
    DATA_LENGTH,
    NULLABLE,
    COMMENTS
FROM USER_TAB_COLUMNS c
LEFT JOIN USER_COL_COMMENTS cc USING (TABLE_NAME, COLUMN_NAME)
ORDER BY TABLE_NAME, COLUMN_ID;
```

### MySQL

```sql
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    COLUMN_TYPE,
    IS_NULLABLE,
    COLUMN_COMMENT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

결과를 JSON으로 변환해서 `schema.json`에 저장하면 돼.

---

## Q: 확장 아이디어는?

### 1. 관계 조회 도구 추가

```javascript
{
  name: "get_table_relations",
  description: "테이블의 FK 관계를 조회합니다"
}
```

### 2. ERD 텍스트 생성

```javascript
case "generate_erd": {
  // 테이블 간 관계를 텍스트 ERD로 표현
  // USERS ──< ORDERS (1:N)
}
```

### 3. 샘플 쿼리 생성

```javascript
case "generate_sample_query": {
  // SELECT * FROM USERS WHERE ... 형태의 샘플 생성
}
```

---

## 체크리스트

### 구축 시

- [ ] Node.js 설치
- [ ] `npm install` 실행
- [ ] `data/schema.json` 생성
- [ ] `node index.js` 테스트

### 등록 시

- [ ] `.mcp.json`에 서버 등록
- [ ] `SCHEMA_PATH` 환경변수 설정
- [ ] `~/.claude.json`에 서버명 추가
- [ ] Claude Code 재시작
- [ ] `/mcp`에서 `connected` 확인

---

## 이어서 읽기

- [[MCP_SERVER_TEMPLATE]] - MCP 서버 기본 템플릿

