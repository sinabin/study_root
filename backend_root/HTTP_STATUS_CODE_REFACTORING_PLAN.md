---
sticker: emoji//1f4c5
---
# HTTP Status Code 리팩토링 가이드

> "왜 모든 API가 200을 반환해?" 라는 질문에서 시작해보자.

---

## Q: 지금 뭐가 문제야?

```java
// 현재 상태: 에러가 나도 HTTP 200
@PostMapping("/save.do")
public DtoAssembler save(@RequestBody ParamMap param) {
    try {
        userService.save(param);
        return createResponse("000", "저장되었습니다.");  // 성공: 200
    } catch (Exception e) {
        return createResponse("110", "저장 실패");        // 실패도: 200 (?!)
    }
}
```

HTTP Status Code가 **의미를 잃어버렸어**.

| 도구 | 영향 |
|------|------|
| **APM/모니터링** | 에러율 0%로 표시 (실제 에러 있어도) |
| **로그 분석** | 4xx/5xx 필터링 불가 |
| **API Gateway** | 서킷브레이커, 재시도 정책 동작 안함 |

---

## Q: 어떻게 고쳐?

`DtoAssembler` 대신 `ResponseEntity<DtoAssembler>`를 반환해.

```java
// 개선된 방식
@PostMapping("/save.do")
public ResponseEntity<DtoAssembler> save(@RequestBody ParamMap param) {
    int cnt = userService.save(param);
    return super.afterInsert(cnt);  // 201 Created 반환!
}
```

---

## Q: HTTP Status Code는 언제 뭘 써?

### 조회 (SELECT)

```
성공 (데이터 있음)  → 200 OK
성공 (데이터 없음)  → 200 OK (빈 배열)
실패               → 404 Not Found
```

### 생성 (INSERT)

```
성공               → 201 Created
유효성 검증 실패    → 400 Bad Request
중복               → 409 Conflict
```

### 수정 (UPDATE)

```
성공               → 200 OK
대상 없음          → 404 Not Found
유효성 검증 실패    → 400 Bad Request
```

### 삭제 (DELETE)

```
성공               → 200 OK (또는 204 No Content)
대상 없음          → 404 Not Found
제약조건 위배       → 409 Conflict
```

---

## Q: BaseService는 어떻게 바꿔?

`BaseService` → `BaseController`로 이름을 바꾸고 `ResponseEntity`를 반환해.

```java
public class BaseController {

    // 조회 성공 → 200 OK
    protected ResponseEntity<DtoAssembler> afterSelectList(List<CmmHashMap<String, Object>> result) {
        assembler = new DtoAssembler();

        if (result != null && result.size() > 0) {
            customMessage.setMsgCd("000");
        } else {
            customMessage.setMsgCd("010");
        }

        assembler.put("list", result);
        assembler.putMessage(customMessage);

        return ResponseEntity.ok(assembler);  // 200 OK
    }

    // INSERT 성공 → 201 Created
    protected ResponseEntity<DtoAssembler> afterInsert(Integer cnt) {
        assembler = new DtoAssembler();
        customMessage.setMsgCd("100");
        assembler.putMessage(customMessage);

        return ResponseEntity.status(HttpStatus.CREATED).body(assembler);  // 201!
    }

    // UPDATE 성공 → 200 OK
    protected ResponseEntity<DtoAssembler> afterUpdate(Integer cnt) {
        return ResponseEntity.ok(assembler);
    }

    // DELETE 성공 → 200 OK
    protected ResponseEntity<DtoAssembler> afterDelete(Integer cnt) {
        return ResponseEntity.ok(assembler);
    }
}
```

---

## Q: 에러 처리는 어떻게 해?

### CustomException에 HttpStatus 추가

```java
public class CustomException extends RuntimeException {
    private HttpStatus httpStatus;  // 추가!

    // 편의 메서드들
    public static CustomException badRequest(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.BAD_REQUEST);
    }

    public static CustomException notFound(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.NOT_FOUND);
    }

    public static CustomException conflict(String code, Object[] arg) {
        return new CustomException(code, arg, "NULL", HttpStatus.CONFLICT);
    }
}
```

### 비즈니스 예외 클래스들

```java
// 400 Bad Request
public class BadRequestException extends CustomException {
    public BadRequestException(String message) {
        super("fw.cmm.036", new String[]{message}, "NULL", HttpStatus.BAD_REQUEST);
    }
}

// 404 Not Found
public class NotFoundException extends CustomException {
    public NotFoundException(String message) {
        super("fw.cmm.040", new String[]{message}, "NULL", HttpStatus.NOT_FOUND);
    }
}

// 409 Conflict
public class ConflictException extends CustomException {
    public ConflictException(String message) {
        super("fw.cmm.041", new String[]{message}, "NULL", HttpStatus.CONFLICT);
    }
}
```

---

## Q: GeneralExceptionHandler는?

```java
@RestControllerAdvice
public class GeneralExceptionHandler {

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<Map<String, Object>> handleCustomException(CustomException ex) {
        HttpStatus status = ex.getHttpStatus() != null
            ? ex.getHttpStatus()
            : HttpStatus.BAD_REQUEST;

        Map<String, Object> errorResponse = new HashMap<>();
        CustomMessage customMessage = new CustomMessage();
        customMessage.setMsgCd("110");
        customMessage.setDetailMsg(getMessage(ex));
        errorResponse.put("message", customMessage);

        return ResponseEntity.status(status).body(errorResponse);  // 적절한 HTTP Status!
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneralException(Exception ex) {
        // 예상치 못한 에러 → 500 Internal Server Error
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(errorResponse);
    }
}
```

---

## Q: Controller는 어떻게 바뀌어?

### Before (레거시)

```java
@RestController
public class UserController extends BaseService {

    @PostMapping("/insertUser.do")
    public DtoAssembler insertUser(@RequestBody ParamMap param) {
        int cnt = userService.insert(param);
        return super.saveData("insert", cnt);  // 항상 200
    }
}
```

### After (개선)

```java
@RestController
public class UserController extends BaseController {

    @PostMapping("/insertUser.do")
    public ResponseEntity<DtoAssembler> insertUser(@RequestBody ParamMap param) {

        // 유효성 검증
        if (!param.containsKey("userId")) {
            throw new BadRequestException("userId는 필수입니다");  // 400
        }

        // 중복 체크
        if (userService.exists(param.getString("userId"))) {
            throw new ConflictException("이미 존재하는 사용자입니다");  // 409
        }

        int cnt = userService.insert(param);
        return super.afterInsert(cnt);  // 201 Created!
    }

    @PostMapping("/updateUser.do")
    public ResponseEntity<DtoAssembler> updateUser(@RequestBody ParamMap param) {
        int cnt = userService.update(param);

        if (cnt == 0) {
            throw new NotFoundException("수정할 사용자가 없습니다");  // 404
        }

        return super.afterUpdate(cnt);  // 200 OK
    }
}
```

---

## Q: 프론트엔드는 어떻게 바뀌어?

### Before (msgCd로 분기)

```javascript
const res = await api.post('/insertUser.do', data);
if (res.data.message.msgCd === '000') {
    // 성공
} else if (res.data.message.msgCd === '110') {
    // 실패
}
```

### After (HTTP Status로 분기)

```javascript
try {
    const res = await api.post('/insertUser.do', data);
    // 성공 (200, 201)
    showSuccess('저장 완료');
} catch (error) {
    // 에러 (400, 404, 409, 500)
    // 인터셉터에서 자동 처리됨
}
```

### Axios 인터셉터 설정

```javascript
axios.interceptors.response.use(
    response => response,
    error => {
        const status = error.response?.status;
        const message = error.response?.data?.message?.detailMsg;

        switch (status) {
            case 400: showWarning(message || '잘못된 요청'); break;
            case 404: showWarning(message || '데이터 없음'); break;
            case 409: showWarning(message || '중복 데이터'); break;
            case 500: showError('서버 오류'); break;
        }

        return Promise.reject(error);
    }
);
```

---

## Q: 마이그레이션은 어떻게 해?

### Phase 1: 인프라 구축

- [ ] `BaseController` 클래스 생성
- [ ] Exception 클래스들 생성
- [ ] `GeneralExceptionHandler` 수정
- [ ] 메시지 코드 등록

### Phase 2: 파일럿 적용

- [ ] 1~2개 Controller에 적용
- [ ] 프론트엔드 테스트
- [ ] 문제점 파악

### Phase 3: 점진적 확대

- [ ] 모듈별 순차 적용
- [ ] 회귀 테스트

### Phase 4: 정리

- [ ] `BaseService` deprecated 처리
- [ ] 문서화

---

## Q: 하위 호환성은?

기존 코드가 깨지지 않게 `BaseService`를 별칭으로 유지해.

```java
@Deprecated
public class BaseService extends BaseController {
    // 기존 코드 그대로 동작
}
```

기존 `msgCd`도 그대로 유지하면서 HTTP Status만 추가하면 돼.

---

## 비교 정리

| | Before | After |
|---|---|---|
| **HTTP Status** | 항상 200 | 의미에 맞게 |
| **에러 구분** | body의 msgCd | HTTP Status Code |
| **반환 타입** | `DtoAssembler` | `ResponseEntity<DtoAssembler>` |
| **예외 처리** | try-catch | `@ControllerAdvice` |
| **모니터링** | 불가능 | 가능 |

---

## 체크리스트

### 새 API 개발 시

- [ ] `ResponseEntity<T>`를 반환 타입으로 사용
- [ ] 적절한 HTTP Status Code 반환
- [ ] 비즈니스 예외는 `CustomException` 상속 클래스 사용
- [ ] Controller에 비즈니스 로직 없음

---

## 이어서 읽기

- [[REST_API_설계_가이드_레거시vs현대]] - REST API 설계 패턴
- [[SPRING_BOOT_PROJECT_REVIEW]] - Spring Boot 프로젝트 구조
- [[ORACLE_CLOB_JSON_SERIALIZATION_ERROR]] - CLOB 직렬화 이슈

