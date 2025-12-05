# Oracle CLOB 타입의 JSON 직렬화 오류 해결기

> Spring Boot + MyBatis + Oracle 환경에서 CLOB 데이터 조회 시 발생하는 Jackson 직렬화 오류와 해결 방법

## 1. 문제 상황

### 1.1. 배경

메인 대시보드에 공지사항/검사착안사항 목록을 구현하고, 제목 클릭 시 상세 팝업을 띄우는 기능을 개발했다. 목록 조회는 정상 동작했지만, **상세 조회 API 호출 시 서버 에러가 발생**했다.

### 1.2. 에러 로그

```
org.springframework.http.converter.HttpMessageConversionException:
Type definition error: [simple type, class oracle.jdbc.internal.Monitor$CloseableLock]

Caused by: com.fasterxml.jackson.databind.exc.InvalidDefinitionException:
No serializer found for class oracle.jdbc.internal.Monitor$CloseableLock
and no properties discovered to create BeanSerializer
(to avoid exception, disable SerializationFeature.FAIL_ON_EMPTY_BEANS)
(through reference chain:
  com.fw.cmm.utils.SmartHashMap["CONTENT"]
  ->oracle.sql.CLOB["physicalConnection"]
  ->oracle.jdbc.driver.T4CConnection["monitorLock"])
```

### 1.3. 문제의 쿼리

```xml
<select id="getNoticeDetailData" parameterType="Hashmap" resultType="smartHashMap">
    SELECT *
      FROM R5.COLOR_BD_ITEM
     WHERE ITEMID = #{ITEMID}
</select>
```

단순해 보이는 `SELECT *` 쿼리가 왜 문제를 일으켰을까?

---

## 2. 원인 분석

### 2.1. 에러 메시지 해석

에러의 참조 체인(reference chain)을 따라가보면:

```
SmartHashMap["CONTENT"]
  → oracle.sql.CLOB["physicalConnection"]
    → oracle.jdbc.driver.T4CConnection["monitorLock"]
```

1. `SmartHashMap`의 `CONTENT` 키에 담긴 값이
2. `oracle.sql.CLOB` 객체이고
3. 이 CLOB 객체가 Oracle JDBC 내부 클래스(`T4CConnection`)에 대한 참조를 가지고 있어서
4. Jackson이 이 내부 클래스를 직렬화하려다 실패한 것

### 2.2. 근본 원인: Oracle CLOB 타입

R5 테이블의 `CONTENTS` 컬럼이 **Oracle CLOB(Character Large Object) 타입**이었다.

```
CONTENTS 컬럼 → Oracle CLOB 타입 → SELECT * 조회 시 oracle.sql.CLOB 객체로 반환
```

### 2.3. 왜 CLOB이 문제인가?

일반적인 VARCHAR2 타입은 MyBatis가 자동으로 Java String으로 변환한다. 하지만 **CLOB 타입은 `oracle.sql.CLOB` 객체로 반환**되며, 이 객체는 다음과 같은 구조를 가진다:

```java
oracle.sql.CLOB
  └─ physicalConnection (oracle.jdbc.driver.T4CConnection)
       └─ monitorLock (oracle.jdbc.internal.Monitor$CloseableLock)
            └─ ... (직렬화 불가능한 내부 클래스들)
```

Jackson은 객체를 JSON으로 변환할 때 모든 프로퍼티를 재귀적으로 탐색한다. CLOB 객체 내부의 Oracle JDBC 커넥션 관련 클래스들은 직렬화를 위한 설계가 되어있지 않아서 오류가 발생한다.

### 2.4. 데이터 흐름도

```
[Oracle DB]                    [MyBatis]                      [Spring/Jackson]
    │                              │                                │
    │  SELECT * FROM TABLE         │                                │
    │  (CLOB 컬럼 포함)             │                                │
    │─────────────────────────────>│                                │
    │                              │                                │
    │  ResultSet 반환               │                                │
    │  (CLOB = oracle.sql.CLOB)    │                                │
    │<─────────────────────────────│                                │
    │                              │                                │
    │                              │  Map<String, Object> 생성       │
    │                              │  map.put("CONTENTS", clobObj)  │
    │                              │─────────────────────────────────>│
    │                              │                                │
    │                              │                    JSON 변환 시도│
    │                              │              CLOB 내부 객체 탐색│
    │                              │                       ❌ 실패! │
```

---

## 3. 해결 방법

### 3.1. 방법 1: SQL에서 CLOB → VARCHAR2 변환 (권장)

가장 간단하고 확실한 방법. 쿼리 단계에서 CLOB을 문자열로 변환한다.

```xml
<select id="getNoticeDetailData" parameterType="Hashmap" resultType="smartHashMap">
    SELECT ITEMID
         , TITLE
         , AUTH_NAME
         , TO_CHAR( RGST_DATE, 'YYYY-MM-DD HH24:MI:SS' ) AS RGST_DATE
         , TO_CHAR( CONTENTS )                           AS CONTENTS
      FROM R5.COLOR_BD_ITEM
     WHERE ITEMID = #{ITEMID}
</select>
```

**주의사항:**
- `TO_CHAR(CLOB)` 함수는 최대 **4000 bytes**까지만 변환 가능
- 4000 bytes를 초과하는 긴 텍스트는 잘림

### 3.2. 방법 2: DBMS_LOB.SUBSTR 사용

긴 텍스트를 여러 조각으로 나누어 가져올 수 있다.

```sql
-- 앞 4000자만 가져오기
SELECT DBMS_LOB.SUBSTR(CONTENTS, 4000, 1) AS CONTENTS
  FROM R5.COLOR_BD_ITEM
 WHERE ITEMID = #{ITEMID}

-- 전체 내용이 필요한 경우 (여러 조각 결합)
SELECT DBMS_LOB.SUBSTR(CONTENTS, 4000, 1)
    || DBMS_LOB.SUBSTR(CONTENTS, 4000, 4001) AS CONTENTS
  FROM ...
```

### 3.3. 방법 3: MyBatis TypeHandler 사용

CLOB을 자동으로 String으로 변환하는 TypeHandler를 등록한다.

```java
@MappedTypes(String.class)
@MappedJdbcTypes(JdbcType.CLOB)
public class ClobTypeHandler extends BaseTypeHandler<String> {

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        Clob clob = rs.getClob(columnName);
        return clob != null ? clob.getSubString(1, (int) clob.length()) : null;
    }

    // ... 다른 메서드 구현
}
```

```xml
<!-- mybatis-config.xml -->
<typeHandlers>
    <typeHandler handler="com.example.ClobTypeHandler"/>
</typeHandlers>
```

### 3.4. 방법 4: Jackson 설정 변경 (비권장)

직렬화 실패 시 예외를 무시하도록 설정할 수 있지만, **근본적인 해결책이 아니다**.

```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        return mapper;
    }
}
```

---

## 4. 적용 결과

### 4.1. 수정 전

```xml
<!-- 문제의 쿼리 -->
<select id="getNoticeDetailData" ...>
    SELECT *
      FROM R5.COLOR_BD_ITEM
     WHERE ITEMID = #{ITEMID}
</select>
```

### 4.2. 수정 후

```xml
<select id="getNoticeDetailData" parameterType="Hashmap" resultType="smartHashMap">
    /* com.biz.mai.mai.mai.MainMapper.getNoticeDetailData.공지사항 상세 조회 */
    /*
     * 주의: SELECT * 사용 시 CLOB 타입 컬럼(CONTENTS)이 oracle.sql.CLOB 객체로 반환되어
     * Jackson JSON 직렬화 오류 발생 (oracle.jdbc.internal.Monitor$CloseableLock 직렬화 불가)
     * 해결: CLOB 컬럼은 TO_CHAR() 또는 DBMS_LOB.SUBSTR()로 문자열 변환 필요
     */
    SELECT ITEMID
         , TITLE
         , AUTH_NAME
         , TO_CHAR( RGST_DATE, 'YYYY-MM-DD HH24:MI:SS' ) AS RGST_DATE
         , TO_CHAR( CONTENTS )                           AS CONTENTS
      FROM R5.COLOR_BD_ITEM
     WHERE ITEMID = #{ITEMID}
</select>
```

---

## 5. 교훈 및 Best Practice

### 5.1. SELECT * 를 피하자

```sql
-- ❌ Bad: 어떤 타입의 컬럼이 있는지 모름
SELECT * FROM TABLE

-- ✅ Good: 필요한 컬럼만 명시적으로
SELECT COL1, COL2, TO_CHAR(CLOB_COL) AS CLOB_COL FROM TABLE
```

### 5.2. LOB 타입은 항상 변환 처리

| Oracle 타입 | 문제점 | 해결책 |
|------------|--------|--------|
| CLOB | oracle.sql.CLOB 객체 반환 | TO_CHAR() 또는 TypeHandler |
| BLOB | oracle.sql.BLOB 객체 반환 | Base64 인코딩 또는 별도 API |
| BFILE | 파일 포인터 반환 | 별도 파일 다운로드 API |

### 5.3. 에러 메시지를 꼼꼼히 읽자

```
(through reference chain:
  SmartHashMap["CONTENT"]->oracle.sql.CLOB["physicalConnection"]->...)
```

이 참조 체인이 **문제의 원인을 정확히 가리키고 있었다**:
- `CONTENT` 컬럼이
- `oracle.sql.CLOB` 타입이라서
- 직렬화 실패

---

## 6. 참고 자료

- [Oracle JDBC CLOB Handling](https://docs.oracle.com/en/database/oracle/oracle-database/19/jjdbc/)
- [MyBatis TypeHandler](https://mybatis.org/mybatis-3/configuration.html#typeHandlers)
- [Jackson SerializationFeature](https://fasterxml.github.io/jackson-databind/javadoc/2.9/com/fasterxml/jackson/databind/SerializationFeature.html)

---

## 7. 환경 정보

- Spring Boot 3.x
- MyBatis 3.x
- Oracle Database (R5 연계 DB)
- Jackson 2.x

---

*작성일: 2024-12-04*
