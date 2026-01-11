---
aliases: []
sticker: emoji//1f636-200d-1f32b-fe0f
---
# Spring Boot Classpath와 ViewResolver 정리

## 1. Spring MVC 요청 처리 흐름

```
Client Request
     ↓
┌─────────────────┐
│  Filter Chain   │  ← (Spring Security Filter 등)
└─────────────────┘
     ↓
┌─────────────────┐
│DispatcherServlet│  ← Front Controller
└─────────────────┘
     ↓
┌─────────────────────────┐
│ HandlerMapping          │  ← RequestMappingHandlerMapping
│ (컨트롤러 매핑 정보 조회)    │
└─────────────────────────┘
     ↓
┌─────────────────────────┐
│ HandlerAdapter          │  ← 실제 컨트롤러 메서드 호출
└─────────────────────────┘
     ↓
┌─────────────────┐
│   Controller    │
└─────────────────┘
     ↓
┌─────────────────┐
│  ViewResolver   │  ← (Thymeleaf 등, @ResponseBody면 생략)
└─────────────────┘
     ↓
   Response
```

---

## 2. @Controller vs @RestController

### ViewResolver를 거치는 경우 (`@Controller`)

```java
@Controller
public class BasicController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("user", user);
        return "home";  // → ViewResolver가 "home.html" 찾아서 렌더링
    }
}
```

### JSON 직접 반환 (`@ResponseBody` 또는 `@RestController`)

```java
@RestController  // = @Controller + @ResponseBody
public class ClientController {

    @PostMapping("/api/user")
    public UserDTO getUser() {
        return userDto;  // → HttpMessageConverter가 JSON으로 변환
    }
}
```

### 처리 흐름 비교

```
@Controller (뷰 반환)
Controller → ViewResolver → View(HTML) → Response

@ResponseBody (JSON 반환)
Controller → HttpMessageConverter(Jackson) → JSON → Response
```

**핵심**: `@ResponseBody`가 있으면 반환값을 HTTP Response Body에 직접 쓰고, 없으면 반환값을 뷰 이름으로 해석한다.

---

## 3. 템플릿 vs 정적 리소스 디렉토리

| 디렉토리 | 용도 | 접근 방식 |
|----------|------|-----------|
| `resources/templates/` | Thymeleaf 등 템플릿 파일 | ViewResolver가 처리 |
| `resources/static/` | CSS, JS, 이미지 등 정적 파일 | URL로 직접 접근 |

### 디렉토리 구조 예시

```
resources/
├── templates/          ← ViewResolver가 여기서 찾음
│   ├── home.html
│   └── user/
│       └── login.html
│
└── static/             ← 정적 리소스 (직접 서빙)
    ├── css/
    ├── js/
    └── images/
```

### 사용 예시

```java
@Controller
public class BasicController {

    @GetMapping("/home")
    public String home() {
        return "home";  // → templates/home.html
    }

    @GetMapping("/user/login")
    public String login() {
        return "user/login";  // → templates/user/login.html
    }
}
```

```html
<!-- static은 브라우저에서 직접 요청 -->
<link href="/css/style.css">   <!-- → static/css/style.css -->
<script src="/js/common.js">   <!-- → static/js/common.js -->
```

**차이점**:
- `static/`에 있는 파일은 Controller 없이 URL로 바로 접근
- `templates/`에 있는 파일은 반드시 Controller를 통해 ViewResolver가 렌더링

---

## 4. Classpath 개념

**Classpath**는 JVM이 클래스 파일과 리소스를 찾는 경로이다.

### Classpath 매핑

```
[개발 시 소스]                    [빌드 후 classpath]
src/main/resources/         →    target/classes/
       │                              │
       └── templates/                 └── templates/
       └── static/                    └── static/
       └── application.properties     └── application.properties
```

### 실제 경로 변환

```properties
spring.thymeleaf.prefix=classpath:/custom-views/
```

| 환경 | 실제 경로 |
|------|-----------|
| 개발 시 | `src/main/resources/custom-views/` |
| 빌드 후 | `target/classes/custom-views/` |
| JAR 실행 | JAR 내부 `/custom-views/` |

---

## 5. 템플릿 경로 변경 방법

### 방법 1: application.properties (간단한 방법)

```properties
# 기본값: classpath:/templates/
spring.thymeleaf.prefix=classpath:/custom-views/
spring.thymeleaf.suffix=.html
```

### 방법 2: @Configuration 클래스 (세밀한 제어)

```java
@Configuration
public class ThymeleafConfig {

    @Bean
    public SpringResourceTemplateResolver templateResolver() {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setPrefix("classpath:/custom-views/");
        resolver.setSuffix(".html");
        resolver.setTemplateMode(TemplateMode.HTML);
        resolver.setCharacterEncoding("UTF-8");
        resolver.setCacheable(false);  // 개발시 false
        return resolver;
    }
}
```

### 여러 경로 설정 (다중 Resolver)

```java
@Configuration
public class ThymeleafConfig {

    @Bean
    public SpringResourceTemplateResolver templateResolver1() {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setPrefix("classpath:/templates/");
        resolver.setSuffix(".html");
        resolver.setOrder(1);  // 우선순위
        resolver.setCheckExistence(true);  // 없으면 다음 resolver로
        return resolver;
    }

    @Bean
    public SpringResourceTemplateResolver templateResolver2() {
        SpringResourceTemplateResolver resolver = new SpringResourceTemplateResolver();
        resolver.setPrefix("classpath:/other-views/");
        resolver.setSuffix(".html");
        resolver.setOrder(2);
        return resolver;
    }
}
```

---

## 6. classpath vs file 비교

```properties
# classpath - JAR 내부 또는 target/classes 에서 찾음
spring.thymeleaf.prefix=classpath:/templates/

# file - 파일 시스템 절대경로에서 찾음
spring.thymeleaf.prefix=file:/opt/app/templates/

# file - 상대경로
spring.thymeleaf.prefix=file:./external-templates/
```

**`classpath:`를 쓰면 JAR로 패키징해도 내부 리소스를 찾을 수 있어서 배포에 용이하다.**
