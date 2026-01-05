---
sticker: emoji//1f48f
---
# 🔐 JWT와 OAuth2 완전 정복 가이드

## 📚 기본 개념 먼저 이해하기

### 1️⃣ OAuth2란? (구글/카카오 로그인)

**비유**: 호텔 투숙객이 프론트 데스크에 요청해서 **임시 출입 카드**를 받는 것과 같습니다.

```
사용자: "구글 계정으로 로그인하고 싶어요"
      ↓
구글: "네, 이 사용자가 맞는지 확인했습니다. 여기 정보 드릴게요!"
      ↓
우리 앱: "감사합니다. 이제 JWT 토큰을 발급해드릴게요!"
```

**OAuth2의 핵심:**
- 사용자가 **직접 비밀번호를 입력하지 않고** 구글/카카오 같은 곳에서 로그인
- 구글이 "이 사람 맞습니다"라고 확인해주면
- 우리 앱은 그 정보를 믿고 사용자를 만들거나 찾아줍니다

---

### 2️⃣ JWT란? (Json Web Token)

**비유**: 영화관 입장권 같은 것입니다.

```
JWT 토큰 내용:
{
  "사용자ID": 123,
  "이름": "홍길동",
  "이메일": "hong@example.com",
  "만료시간": "2025년 1월 1일 12시"
}
+ 암호화된 서명 (위조 방지)
```

**JWT의 특징:**
- **한번 발급하면 서버가 기억할 필요 없음** (Stateless)
- **토큰 자체에 정보가 들어있음** (사용자 ID, 이름, 권한 등)
- **서명이 있어서 위조 불가능**
- **만료시간이 있음** (보안을 위해)

**JWT 검증 원리 (왜 서버가 기억할 필요 없는가?):**
```
JWT 구조: Header.Payload.Signature
                          │
          Signature = 서버_비밀키로_서명(Header + Payload)
```

- 서버는 토큰 자체를 저장하지 않고, **비밀키(Secret Key)만** 보관
- 토큰이 들어오면 비밀키로 서명을 다시 계산 → 토큰의 서명과 비교
  - **일치** → 서버가 발급한 정상 토큰
  - **불일치** → 변조된 토큰 (거부!)
- 공격자가 Payload를 변조해도 **비밀키 없이는 올바른 서명을 만들 수 없음**

---

## 🎬 로그인 흐름

### **시나리오: 사용자가 "구글 로그인" 버튼을 누른다**

```
1단계: 구글 로그인 페이지로 이동
   ↓
2단계: 구글에서 로그인 성공
   ↓
3단계: 우리 앱으로 돌아옴 (OAuth2 콜백)
   ↓
4단계: JWT 토큰 발급
   ↓
5단계: 이후 모든 API 요청에 JWT 토큰 사용
```

---

## 🔍 코드로 보는 상세 흐름

### **1단계: SecurityConfig.kt - 보안 설정**

**파일 위치**: `src/main/kotlin/com/newdaptor/stocklens/config/SecurityConfig.kt`

```kotlin
// SecurityConfig.kt:37
.sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
```
**의미**: 세션을 사용하지 않고, JWT로만 인증하겠다는 뜻

```kotlin
// SecurityConfig.kt:69-74
.oauth2Login { oauth2 ->
    oauth2
        .authorizationEndpoint { ... }  // 구글 로그인 시작점
        .userInfoEndpoint { it.userService(oAuth2UserService) }  // 구글에서 사용자 정보 받기
        .successHandler(oAuth2SuccessHandler)  // 로그인 성공 후 처리
}
```
**의미**: 구글 OAuth2 로그인을 설정

```kotlin
// SecurityConfig.kt:75
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter::class.java)
```
**의미**: 모든 요청마다 JWT 토큰을 확인하는 필터를 추가

---

### **2단계: OAuth2UserService.kt - 구글에서 사용자 정보 받기**

**파일 위치**: `src/main/kotlin/com/newdaptor/stocklens/security/OAuth2UserService.kt`

```kotlin
// OAuth2UserService.kt:16-21
override fun loadUser(userRequest: OAuth2UserRequest): OAuth2User {
    val oAuth2User = super.loadUser(userRequest)  // 구글에서 사용자 정보 가져옴
    val registrationId = userRequest.clientRegistration.registrationId  // "google"

    return processOAuth2User(registrationId, oAuth2User)
}
```

**동작:**
1. 구글에서 사용자 정보를 받아옴 (email, name, picture)
2. DB에서 이 사용자를 찾아봄
3. **없으면 새로 만들고** (회원가입), **있으면 정보 업데이트**

```kotlin
// OAuth2UserService.kt:32-45
var user = userMapper.findByProviderAndProviderId(registrationId, providerId)

if (user == null) {
    // 🆕 신규 회원가입 (1초 컷!)
    user = User(
        email = email,
        nickname = name,
        profileImage = picture,
        provider = registrationId,  // "google"
        providerId = providerId,     // 구글 고유 ID
        role = Role.USER
    )
    userMapper.insert(user)
    user = userMapper.findByProviderAndProviderId(registrationId, providerId)!!
} else {
    // 기존 사용자 정보 업데이트
    user = user.copy(
        email = email,
        nickname = name ?: user.nickname,
        profileImage = picture ?: user.profileImage
    )
    userMapper.update(user)
}
```

---

### **3단계: OAuth2SuccessHandler.kt - JWT 토큰 발급**

**파일 위치**: `src/main/kotlin/com/newdaptor/stocklens/security/OAuth2SuccessHandler.kt`

```kotlin
// OAuth2SuccessHandler.kt:34-41
val accessToken = jwtTokenProvider.createAccessToken(
    userId = principal.id,
    email = principal.email,
    name = principal.name,
    profileImage = principal.profileImage,
    role = principal.authorities.first().authority.removePrefix("ROLE_")
)
val refreshToken = jwtTokenProvider.createRefreshToken(principal.id)
```

**두 가지 토큰 발급:**
- **Access Token** (유효기간: 1시간) - 실제 API 요청에 사용
- **Refresh Token** (유효기간: 7일) - Access Token 재발급용

```kotlin
// OAuth2SuccessHandler.kt:44-53
redisTemplate.opsForValue().set(
    "SESSION:${principal.id}",
    refreshToken,
    refreshExpiration,
    TimeUnit.MILLISECONDS
)
```
**의미**: Refresh Token을 Redis에 저장 (나중에 검증용)

```kotlin
// OAuth2SuccessHandler.kt:63-67
val targetUrl = UriComponentsBuilder.fromUriString(redirectUri)
    .queryParam("token", accessToken)
    .queryParam("refresh_token", refreshToken)
    .build()
    .toUriString()
```
**의미**: 프론트엔드로 리다이렉트하면서 토큰을 URL 파라미터로 전달

**리다이렉트 예시:**
```
http://localhost:3000/oauth2/callback?token=eyJhbGc...&refresh_token=eyJhbGc...
```

---

### **4단계: JwtTokenProvider.kt - JWT 토큰 생성/검증**

**파일 위치**: `src/main/kotlin/com/newdaptor/stocklens/security/JwtTokenProvider.kt`

#### **토큰 생성 (createAccessToken)**

```kotlin
// JwtTokenProvider.kt:29-44
fun createAccessToken(...): String {
    return Jwts.builder()
        .subject(userId.toString())           // 주인공: 사용자 ID
        .claim("email", email)                // 추가 정보들
        .claim("name", name)
        .claim("profile_image", profileImage)
        .claim("role", role)
        .claim("type", "access")              // 토큰 타입 표시
        .issuedAt(now)                        // 발급 시간
        .expiration(expiry)                   // 만료 시간
        .signWith(key)                        // 🔐 암호화 서명
        .compact()
}
```

**생성된 JWT 토큰 예시:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjMiLCJlbWFpbCI6ImhvbmdAZXhhbXBsZS5jb20iLCJuYW1lIjoi7ZmN6ri464-ZIiwicm9sZSI6IlVTRVIiLCJ0eXBlIjoiYWNjZXNzIiwiaWF0IjoxNzAwMDAwMDAwLCJleHAiOjE3MDAwMDM2MDB9.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**토큰 구조:**
- **Header**: 알고리즘 정보 (HS256)
- **Payload**: 사용자 정보 (ID, email, name, role, ...)
- **Signature**: 위조 방지 서명

#### **토큰 검증 (validateToken)**

```kotlin
// JwtTokenProvider.kt:59-68
fun validateToken(token: String): Boolean {
    return try {
        val claims = parseClaims(token)
        !claims.expiration.before(Date())  // 만료 안됐는지 확인
    } catch (e: JwtException) {
        false  // 서명이 잘못됨
    } catch (e: IllegalArgumentException) {
        false
    }
}
```

---

### **5단계: JwtAuthenticationFilter.kt - 매 요청마다 토큰 확인**

**파일 위치**: `src/main/kotlin/com/newdaptor/stocklens/security/JwtAuthenticationFilter.kt`

**사용자가 API를 호출할 때마다 실행됩니다**

```kotlin
// JwtAuthenticationFilter.kt:16-33
override fun doFilterInternal(...) {
    try {
        val token = getTokenFromRequest(request)  // ① 토큰 추출

        if (token != null && jwtTokenProvider.validateToken(token)) {  // ② 토큰 검증
            val authentication = jwtTokenProvider.getAuthentication(token)  // ③ 인증 객체 생성
            SecurityContextHolder.getContext().authentication = authentication  // ④ 인증 완료!
        }
    } catch (e: Exception) {
        logger.error("Could not set user authentication", e)
    }

    filterChain.doFilter(request, response)  // ⑤ 다음 단계로
}
```

**① 토큰 추출:**
```kotlin
// JwtAuthenticationFilter.kt:35-43
private fun getTokenFromRequest(request: HttpServletRequest): String? {
    val bearerToken = request.getHeader("Authorization")  // "Bearer eyJhbGc..."

    return if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
        bearerToken.substring(7)  // "Bearer " 제거 → "eyJhbGc..."
    } else {
        null
    }
}
```

**프론트엔드에서 API 요청 예시:**
```javascript
fetch('/api/v1/bookmarks', {
  headers: {
    'Authorization': 'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
  }
})
```

---

### **6단계: AuthController.kt - 토큰 갱신 및 사용자 정보 조회**

**파일 위치**: `src/main/kotlin/com/newdaptor/stocklens/controller/AuthController.kt`

#### **토큰 재발급 (Refresh Token 사용)**

```kotlin
// AuthController.kt:22-26
@PostMapping("/refresh")
fun refreshToken(@RequestBody request: RefreshTokenRequest): ResponseEntity<...> {
    val result = authService.refreshToken(request.refreshToken)
    return ResponseEntity.ok(ApiResponse.success(result))
}
```

**AuthService.kt 내부 동작:**
```kotlin
// AuthService.kt:23-68
fun refreshToken(refreshToken: String): TokenResponse {
    // ① Refresh Token 검증
    if (!jwtTokenProvider.validateToken(refreshToken)) {
        throw InvalidTokenException("Invalid refresh token")
    }

    // ② Refresh Token인지 확인
    if (!jwtTokenProvider.isRefreshToken(refreshToken)) {
        throw InvalidTokenException("Not a refresh token")
    }

    // ③ Redis에 저장된 토큰과 일치하는지 확인
    val userId = jwtTokenProvider.getUserIdFromToken(refreshToken)
    val storedToken = redisTemplate.opsForValue().get("SESSION:$userId")
    if (storedToken != refreshToken) {
        throw InvalidTokenException("Refresh token not found or expired")
    }

    // ④ 새 토큰 발급
    val newAccessToken = jwtTokenProvider.createAccessToken(...)
    val newRefreshToken = jwtTokenProvider.createRefreshToken(user.id)

    // ⑤ Redis에 새 Refresh Token 저장
    redisTemplate.opsForValue().set(...)

    return TokenResponse(
        accessToken = newAccessToken,
        refreshToken = newRefreshToken,
        expiresIn = 3600
    )
}
```

#### **현재 사용자 정보 조회**

```kotlin
// AuthController.kt:36-40
@GetMapping("/me")
fun getCurrentUser(@AuthenticationPrincipal principal: UserPrincipal): ResponseEntity<...> {
    val user = authService.getUserById(principal.id)
    return ResponseEntity.ok(ApiResponse.success(UserResponse.from(user)))
}
```

**동작:**
1. JWT 토큰에서 사용자 ID 추출 (JwtAuthenticationFilter가 이미 처리)
2. `@AuthenticationPrincipal`로 현재 로그인한 사용자 정보 가져옴
3. DB에서 최신 사용자 정보 조회
4. UserResponse로 변환해서 리턴

---

## 🎯 전체 흐름 다이어그램

```
┌─────────────┐
│  사용자      │
└──────┬──────┘
       │ ① "구글 로그인" 클릭
       ▼
┌──────────────────────┐
│   SecurityConfig     │ ② OAuth2 로그인 설정 확인
│  (oauth2Login 설정)  │
└──────┬───────────────┘
       │ ③ 구글 로그인 페이지로 리다이렉트
       ▼
┌──────────────────┐
│     구글         │ ④ 사용자 로그인
│  (OAuth2 Provider)│
└──────┬───────────┘
       │ ⑤ 인증 성공! 사용자 정보 제공
       ▼
┌──────────────────────┐
│  OAuth2UserService   │ ⑥ 사용자 찾기/생성
│  (사용자 DB 저장)     │
└──────┬───────────────┘
       │ ⑦ UserPrincipal 생성
       ▼
┌──────────────────────┐
│ OAuth2SuccessHandler │ ⑧ JWT 토큰 발급
│  (JWT 생성 & 저장)    │    - Access Token (1시간)
│                      │    - Refresh Token (7일)
└──────┬───────────────┘
       │ ⑨ 프론트엔드로 리다이렉트
       │    http://localhost:3000/oauth2/callback?token=xxx&refresh_token=yyy
       ▼
┌─────────────────┐
│  프론트엔드     │ ⑩ 토큰 저장 (localStorage)
└──────┬──────────┘
       │ ⑪ API 요청 시 토큰 포함
       │    Authorization: Bearer xxx
       ▼
┌──────────────────────┐
│ JwtAuthenticationFilter│ ⑫ 매 요청마다 토큰 검증
│  (토큰 검증)          │    - 토큰 추출
│                      │    - 서명 검증
└──────┬───────────────┘    - 만료 확인
       │ ⑬ 인증 성공! SecurityContext에 저장
       ▼
┌──────────────────┐
│   Controller     │ ⑭ @AuthenticationPrincipal로 사용자 정보 사용
│  (비즈니스 로직)  │
└──────────────────┘
```

---

## 🔄 토큰 만료 시 갱신 흐름

```
Access Token 만료됨 (1시간 후)
       │
       ▼
프론트엔드: POST /api/v1/auth/refresh
       │        Body: { "refreshToken": "yyy" }
       ▼
┌──────────────────┐
│   AuthService    │ ① Refresh Token 검증
│                  │ ② Redis에서 토큰 확인
│                  │ ③ 새 Access Token 발급
│                  │ ④ 새 Refresh Token 발급
└──────┬───────────┘
       │ ⑤ 새 토큰들 리턴
       ▼
프론트엔드: 토큰 업데이트 후 재시도
```

---

## 🔐 보안 포인트

### 1. **Access Token은 짧게, Refresh Token은 길게**
```yaml
# application.yml
jwt:
  access-expiration: 3600000    # 1시간
  refresh-expiration: 604800000 # 7일
```
**이유**: Access Token이 탈취되어도 1시간만 유효

### 2. **Refresh Token은 Redis에 저장**
```kotlin
// OAuth2SuccessHandler.kt:45-50
redisTemplate.opsForValue().set(
    "SESSION:${principal.id}",
    refreshToken,
    refreshExpiration,
    TimeUnit.MILLISECONDS
)
```
**이유**: 서버에서 언제든 무효화 가능 (로그아웃 시)

### 3. **로그아웃 시 Refresh Token 삭제**
```kotlin
// AuthController.kt:28-34
@PostMapping("/logout")
fun logout(@AuthenticationPrincipal principal: UserPrincipal): ResponseEntity<...> {
    redisTemplate.delete("SESSION:${principal.id}")  // Redis에서 삭제
    return ResponseEntity.ok(ApiResponse.success("Logged out successfully"))
}
```

### 4. **CORS 설정으로 허용된 도메인만 접근**
```kotlin
// SecurityConfig.kt:82-88
allowedOriginPatterns = listOf("*")  // 개발용 (운영에서는 구체적으로 지정)
allowedMethods = listOf("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
allowCredentials = true
```

---

## 📝 핵심 파일별 역할 요약

| 파일 | 역할 | 핵심 메서드 |
|-----|------|-----------|
| **SecurityConfig.kt** | 보안 전체 설정 | `securityFilterChain()` - OAuth2, JWT 필터 설정 |
| **OAuth2UserService.kt** | 구글 로그인 후 사용자 처리 | `loadUser()` - 사용자 찾기/생성 |
| **OAuth2SuccessHandler.kt** | 로그인 성공 후 JWT 발급 | `onAuthenticationSuccess()` - 토큰 생성 |
| **JwtTokenProvider.kt** | JWT 토큰 생성/검증 | `createAccessToken()`, `validateToken()` |
| **JwtAuthenticationFilter.kt** | 매 요청마다 토큰 검증 | `doFilterInternal()` - 토큰 확인 |
| **AuthService.kt** | 토큰 갱신, 사용자 조회 | `refreshToken()` - 토큰 재발급 |
| **AuthController.kt** | 인증 API 엔드포인트 | `/refresh`, `/logout`, `/me` |

---

## 💡 프론트엔드에서 사용하는 방법

### 1. **로그인 버튼**
```javascript
// 구글 로그인으로 이동
window.location.href = 'http://localhost:8080/oauth2/authorization/google';
```

### 2. **콜백 처리**
```javascript
// http://localhost:3000/oauth2/callback?token=xxx&refresh_token=yyy
const params = new URLSearchParams(window.location.search);
const accessToken = params.get('token');
const refreshToken = params.get('refresh_token');

// localStorage에 저장
localStorage.setItem('accessToken', accessToken);
localStorage.setItem('refreshToken', refreshToken);
```

### 3. **API 요청**
```javascript
const response = await fetch('/api/v1/bookmarks', {
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('accessToken')}`
  }
});
```

### 4. **토큰 갱신 (Access Token 만료 시)**
```javascript
const response = await fetch('/api/v1/auth/refresh', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    refreshToken: localStorage.getItem('refreshToken')
  })
});

const { accessToken, refreshToken } = await response.json();
localStorage.setItem('accessToken', accessToken);
localStorage.setItem('refreshToken', refreshToken);
```

### 5. **자동 토큰 갱신 (Axios Interceptor 예시)**
```javascript
import axios from 'axios';

// Request Interceptor - 모든 요청에 토큰 추가
axios.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response Interceptor - 401 에러 시 자동 토큰 갱신
axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Access Token 만료 (401 에러)
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;

      try {
        // Refresh Token으로 새 토큰 발급
        const refreshToken = localStorage.getItem('refreshToken');
        const response = await axios.post('/api/v1/auth/refresh', {
          refreshToken
        });

        const { accessToken, refreshToken: newRefreshToken } = response.data.data;
        localStorage.setItem('accessToken', accessToken);
        localStorage.setItem('refreshToken', newRefreshToken);

        // 실패했던 원래 요청 재시도
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;
        return axios(originalRequest);
      } catch (refreshError) {
        // Refresh Token도 만료됨 → 로그인 페이지로
        localStorage.removeItem('accessToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
        return Promise.reject(refreshError);
      }
    }

    return Promise.reject(error);
  }
);
```

---

## 🚀 추가 학습 자료

### JWT 관련
- [JWT.io](https://jwt.io/) - JWT 토큰 디코딩 및 검증
- [RFC 7519 - JWT 스펙](https://tools.ietf.org/html/rfc7519)

### OAuth2 관련
- [OAuth 2.0 공식 문서](https://oauth.net/2/)
- [Spring Security OAuth2 공식 문서](https://docs.spring.io/spring-security/reference/servlet/oauth2/index.html)

### 보안 관련
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [JWT 보안 Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)

---

## 📌 자주 묻는 질문 (FAQ)

### Q1: Access Token과 Refresh Token을 왜 둘 다 사용하나요?
**A**: 보안과 사용자 경험의 균형을 위해서입니다.
- Access Token만 사용하면: 유효기간이 길면 위험하고, 짧으면 자주 로그인해야 함
- 두 개를 사용하면: Access Token은 짧게(1시간), Refresh Token은 길게(7일) 설정 가능

### Q2: JWT는 암호화되나요?
**A**: **서명(Signature)**은 되지만 **암호화는 안 됩니다**.
- Payload는 Base64 인코딩만 되어 있어서 누구나 디코딩 가능
- 따라서 민감한 정보(비밀번호 등)는 절대 넣으면 안 됨
- 서명이 있어서 **위조는 불가능**하지만 **내용은 볼 수 있음**

### Q3: Redis가 없으면 어떻게 하나요?
**A**: Refresh Token을 DB에 저장하거나, 메모리에 저장할 수 있습니다.
- Redis는 빠르고 자동 만료 기능이 있어서 편리
- DB 사용 시: 별도로 만료 체크 로직 필요
- 메모리 사용 시: 서버 재시작 시 모든 토큰 사라짐

### Q4: 로그아웃은 어떻게 구현하나요?
**A**: Refresh Token을 서버에서 삭제합니다.
```kotlin
@PostMapping("/logout")
fun logout(@AuthenticationPrincipal principal: UserPrincipal) {
    redisTemplate.delete("SESSION:${principal.id}")
    // Access Token은 클라이언트에서 삭제 (localStorage.removeItem)
}
```

### Q5: CSRF 공격은 괜찮나요?
**A**: JWT를 사용하면 CSRF 걱정이 적습니다.
- 쿠키를 사용하지 않고 Authorization 헤더 사용
- SecurityConfig에서 `.csrf { it.disable() }` 설정

---

## 🛠️ 트러블슈팅

### 문제 1: "401 Unauthorized" 에러
**원인**: JWT 토큰이 만료되었거나 잘못됨
**해결**:
```javascript
// 1. 토큰 확인
console.log(localStorage.getItem('accessToken'));

// 2. 토큰 갱신 시도
await fetch('/api/v1/auth/refresh', {
  method: 'POST',
  body: JSON.stringify({ refreshToken: localStorage.getItem('refreshToken') })
});

// 3. 안 되면 재로그인
```

### 문제 2: CORS 에러
**원인**: 프론트엔드 도메인이 허용 목록에 없음
**해결**:
```kotlin
// SecurityConfig.kt
allowedOriginPatterns = listOf("http://localhost:3000", "http://localhost:5173")
```

### 문제 3: Redis 연결 에러
**원인**: Redis가 실행 중이 아님
**해결**:
```bash
# Redis 실행
docker run -d -p 6379:6379 redis

# 또는 로컬 설치 후
redis-server
```

---

**작성일**: 2025-12-28
**프로젝트**: StockLens API
**Kotlin + Spring Boot + JWT + OAuth2**
