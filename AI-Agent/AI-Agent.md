---
sticker: emoji//1f916
banner: "https://images.unsplash.com/photo-1677442136019-21780ecad995?w=1200"
banner_y: 0.3
---

# AI Agent 학습 노트

> MCP (Model Context Protocol) 서버 개발 및 Claude Code 확장

---

## 문서 목록

### MCP 서버 개발
- [[MCP_SERVER_TEMPLATE]] - MCP 서버 기본 템플릿
- [[DB_SCHEMA MCP 서버 구축해보기]] - DB 스키마 조회 MCP 실전

---

## MCP 서버 개발 체크리스트

### 기본 구조
- [ ] `@modelcontextprotocol/sdk` 설치
- [ ] `package.json`에 `"type": "module"` 설정
- [ ] ListToolsRequestSchema 핸들러 구현
- [ ] CallToolRequestSchema 핸들러 구현

### 등록
- [ ] `.mcp.json` 설정
- [ ] `~/.claude.json`에 서버 추가
- [ ] Claude Code 재시작
- [ ] `/mcp`로 연결 확인

---

## 아이디어 목록

| 아이디어 | 설명 | 상태 |
|---------|------|------|
| DB Schema MCP | DB 테이블 구조 조회 | 완료 |
| API Docs MCP | Swagger/OpenAPI 문서 조회 | 계획 |
| Git Stats MCP | 커밋 통계, 변경 이력 | 아이디어 |
| Build Runner MCP | 빌드/테스트 실행 | 아이디어 |

---

## 참고 자료

- [MCP 공식 문서](https://modelcontextprotocol.io/)
- [MCP SDK npm](https://www.npmjs.com/package/@modelcontextprotocol/sdk)

