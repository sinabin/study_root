---
sticker: emoji//26a0-fe0f
---
# Oracle CLOB 타입의 JSON 직렬화 오류

> "SELECT * 했는데 왜 에러가 나지?" 라는 질문에서 시작해보자.

---

## Q: 단순한 SELECT * 인데 왜 에러가 나?

```xml
<select id="getNoticeDetailData" resultType="smartHashMap">
    SELECT * FROM R5.COLOR_BD_ITEM WHERE ITEMID = #{ITEMID}
</select>
```

이 쿼리가 아래 에러를 발생시켰어:

```
com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
No serializer found for class oracle.jdbc.internal.Monitor$CloseableLock
```

---

## Q: 에러 메시지가 뭘 말하는 거야?

에러의 **참조 체인(reference chain)**을 따라가보면:

```
SmartHashMap["CONTENT"]
  → oracle.sql.CLOB["physicalConnection"]
    → oracle.jdbc.driver.T4CConnection["monitorLock"]
```

1. `SmartHashMap`의 `CONTENT` 키에 담긴 값이
2. `oracle.sql.CLOB` 객체이고
3. 이 CLOB 객체가 Oracle JDBC 내부 클래스에 대한 참조를 가지고 있어서
4. Jackson이 이 내부 클래스를 직렬화하려다 실패한 것

---

## Q: CLOB이 뭐야?

**CLOB (Character Large Object)**: 대용량 텍스트를 저장하는 Oracle 데이터 타입이야.

```
일반 VARCHAR2   → Java String으로 자동 변환
CLOB 타입       → oracle.sql.CLOB 객체로 반환 (문제!)
```

### CLOB 객체의 내부 구조

```
oracle.sql.CLOB
  └─ physicalConnection (oracle.jdbc.driver.T4CConnection)
       └─ monitorLock (직렬화 불가능한 내부 클래스)
            └─ ...
```

Jackson이 JSON으로 변환할 때 **모든 프로퍼티를 재귀적으로 탐색**하는데, CLOB 내부의 커넥션 관련 클래스들은 직렬화 설계가 안 되어 있어서 오류가 발생해.

---

## Q: 그럼 어떻게 해결해?

### 방법 1: SQL에서 TO_CHAR() 사용 (권장)

```xml
<select id="getNoticeDetailData" resultType="smartHashMap">
    SELECT ITEMID
         , TITLE
         , AUTH_NAME
         , TO_CHAR( RGST_DATE, 'YYYY-MM-DD HH24:MI:SS' ) AS RGST_DATE
         , TO_CHAR( CONTENTS ) AS CONTENTS
      FROM R5.COLOR_BD_ITEM
     WHERE ITEMID = #{ITEMID}
</select>
```

**주의**: `TO_CHAR(CLOB)`는 최대 **4000 bytes**까지만 변환 가능!

### 방법 2: DBMS_LOB.SUBSTR 사용

```sql
-- 앞 4000자만 가져오기
SELECT DBMS_LOB.SUBSTR(CONTENTS, 4000, 1) AS CONTENTS FROM TABLE

-- 전체 내용 (여러 조각 결합)
SELECT DBMS_LOB.SUBSTR(CONTENTS, 4000, 1)
    || DBMS_LOB.SUBSTR(CONTENTS, 4000, 4001) AS CONTENTS FROM TABLE
```

### 방법 3: MyBatis TypeHandler

```java
@MappedTypes(String.class)
@MappedJdbcTypes(JdbcType.CLOB)
public class ClobTypeHandler extends BaseTypeHandler<String> {
    @Override
    public String getNullableResult(ResultSet rs, String columnName) {
        Clob clob = rs.getClob(columnName);
        return clob != null ? clob.getSubString(1, (int) clob.length()) : null;
    }
}
```

---

## Q: SELECT * 가 왜 문제야?

| 방식 | 문제점 |
|------|--------|
| `SELECT *` | 어떤 타입의 컬럼이 있는지 모름 |
| 명시적 컬럼 | CLOB 등 특수 타입 변환 가능 |

```sql
-- Bad: 어떤 타입의 컬럼이 있는지 모름
SELECT * FROM TABLE

-- Good: 필요한 컬럼만 명시적으로
SELECT COL1, COL2, TO_CHAR(CLOB_COL) AS CLOB_COL FROM TABLE
```

---

## Oracle LOB 타입 처리 방법 정리

| Oracle 타입 | 문제점 | 해결책 |
|------------|--------|--------|
| **CLOB** | oracle.sql.CLOB 객체 반환 | TO_CHAR() 또는 TypeHandler |
| **BLOB** | oracle.sql.BLOB 객체 반환 | Base64 인코딩 또는 별도 API |
| **BFILE** | 파일 포인터 반환 | 별도 파일 다운로드 API |

---

## 교훈

### 에러 메시지를 꼼꼼히 읽자!

```
(through reference chain:
  SmartHashMap["CONTENT"]->oracle.sql.CLOB["physicalConnection"]->...)
```

이 참조 체인이 **문제의 원인을 정확히 가리키고 있었어**:
- `CONTENT` 컬럼이
- `oracle.sql.CLOB` 타입이라서
- 직렬화 실패

---

## 이어서 읽기

- [[SPRING_BOOT_PROJECT_REVIEW]] - Spring Boot 프로젝트 구조 리뷰
- [[HTTP_STATUS_CODE_REFACTORING_PLAN]] - HTTP 상태 코드 리팩토링
