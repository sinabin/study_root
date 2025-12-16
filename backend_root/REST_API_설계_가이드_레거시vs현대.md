---
sticker: emoji//2b50
---
# REST API 설계: 레거시 vs 현대

> "왜 모든 응답이 HTTP 200이야?" 라는 질문에서 시작해보자.

---

## Q: 왜 에러인데 HTTP 200이 나와?

```java
// 레거시 패턴: 모든 응답이 200
@PostMapping("/save.do")
public DtoAssembler save(@RequestBody ParamMap param) {
    try {
        userService.save(param);
        return createResponse("000", "저장되었습니다.");  // 성공도 200
    } catch (Exception e) {
        return createResponse("110", "저장 실패");        // 실패도 200 (?!)
    }
}
```

```javascript
// 프론트엔드에서 body 내 msgCd로 성공/실패 판단
const res = await api.postJson('/save.do', data);
if (res.message.msgCd === '000') {
    // 성공 처리
} else if (res.message.msgCd === '110') {
    // 에러 처리
}
```

---

## Q: 이게 왜 문제야?

HTTP 상태 코드는 **클라이언트-서버 간 계약**이야.

```
2xx → 성공
4xx → 클라이언트 오류
5xx → 서버 오류
```

실패해도 200을 반환하면 이 계약이 깨져.

### 인프라/모니터링이 무력화돼

| 도구 | 영향 |
|------|------|
| **APM** | 에러율 0%로 표시 (실제 에러가 있어도) |
| **로그 분석** | 4xx/5xx 필터링 불가 |
| **API Gateway** | 서킷브레이커, 재시도 정책 동작 안함 |
| **CDN/캐시** | 에러 응답도 캐시될 수 있음 |

---

## Q: Controller가 Service를 상속받는 것도 문제야?

```java
// 레거시 패턴
public class UserController extends BaseService {
    @PostMapping("/save.do")
    public DtoAssembler save(@RequestBody ParamMap param) {
        return super.saveData("insert");  // 부모 메서드 호출
    }
}
```

```java
public class BaseService {
    @Autowired
    protected DtoAssembler assembler;  // 인스턴스 변수

    protected DtoAssembler saveData(String strMethod) {
        assembler = new DtoAssembler();  // @Autowired 받은 걸 버리고 new?!
        // ...
    }
}
```

### 문제점

| 문제 | 설명 |
|------|------|
| 책임 혼재 | Controller는 HTTP 처리, Service는 비즈니스 로직 |
| 상속 오용 | "Controller **is-a** Service"는 성립 안 함 |
| 스레드 안전성 | 싱글톤에서 인스턴스 변수 공유 시 동시성 문제 |
| @Autowired + new | DI 컨테이너 관리 객체를 new로 생성하는 모순 |

---

## Q: 그럼 현대적인 방식은 어떻게 해?

### HTTP 상태 코드 활용

| 상태 코드 | 용도 | 예시 |
|-----------|------|------|
| **200 OK** | 조회/수정 성공 | GET /users/1 |
| **201 Created** | 생성 성공 | POST /users |
| **204 No Content** | 삭제 성공 | DELETE /users/1 |
| **400 Bad Request** | 잘못된 요청 | 필수 파라미터 누락 |
| **404 Not Found** | 리소스 없음 | 존재하지 않는 사용자 |
| **409 Conflict** | 충돌 | 중복 이메일 |
| **500 Internal Server Error** | 서버 오류 | 예상치 못한 예외 |

---

## Q: 코드로 보여줘

### 현대적인 Controller

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @RequestBody @Valid UserCreateRequest request) {
        UserResponse created = userService.create(request);
        URI location = URI.create("/api/users/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 전역 예외 처리

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 리소스 없음 → 404
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException e) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse.of("NOT_FOUND", e.getMessage()));
    }

    // 유효성 검증 실패 → 400
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        return ResponseEntity
            .badRequest()
            .body(ErrorResponse.of("VALIDATION_ERROR", "입력값이 올바르지 않습니다."));
    }
}
```

---

## Q: 프론트엔드는 어떻게 바뀌어?

### 레거시 (msgCd 체크 필요)

```javascript
const res = await api.postJson('/users', data);
if (res.message.msgCd === '000') { ... }
else if (res.message.msgCd === '010') { ... }
else if (res.message.msgCd === '110') { ... }
```

### 현대적 (HTTP 상태 코드로 자동 분기)

```javascript
try {
    const res = await api.post('/users', data);  // 성공 시만 여기
    showSuccess('저장 완료');
} catch (error) {
    // 에러는 인터셉터에서 처리됨
}
```

### Axios 인터셉터

```javascript
http.interceptors.response.use(
    (response) => response,
    (error) => {
        const status = error.response?.status;
        switch (status) {
            case 400: showWarning('입력 오류'); break;
            case 401: redirectToLogin(); break;
            case 404: showWarning('데이터 없음'); break;
            case 500: showError('서버 오류'); break;
        }
        return Promise.reject(error);
    }
);
```

---

## 비교 정리

|                | 레거시            | 현대적               |
| -------------- | -------------- | ----------------- |
| **HTTP 상태**    | 항상 200         | 의미에 맞는 코드         |
| **에러 구분**      | body의 msgCd    | HTTP 상태 코드        |
| **Controller** | BaseService 상속 | 상속 없음             |
| **예외 처리**      | try-catch 직접   | @ControllerAdvice |
| **모니터링**       | 불가능            | 가능                |

---

## 체크리스트

### 새 API 개발 시

- [ ] `ResponseEntity<T>`를 반환 타입으로 사용했는가?
- [ ] 적절한 HTTP 상태 코드를 사용했는가?
- [ ] `@Valid`로 요청 유효성 검증을 했는가?
- [ ] Controller에 비즈니스 로직이 없는가?

### 피해야 할 패턴

- [ ] Controller가 다른 클래스를 상속받지 않는가?
- [ ] 모든 응답을 HTTP 200으로 반환하지 않는가?
- [ ] `@Autowired` 필드를 `new`로 덮어쓰지 않는가?

---

## 이어서 읽기

- [[HTTP_STATUS_CODE_REFACTORING_PLAN]] - HTTP 상태 코드 리팩토링 상세 계획
- [[SPRING_BOOT_PROJECT_REVIEW]] - Spring Boot 프로젝트 구조 리뷰
- [[ORACLE_CLOB_JSON_SERIALIZATION_ERROR]] - CLOB 직렬화 이슈
