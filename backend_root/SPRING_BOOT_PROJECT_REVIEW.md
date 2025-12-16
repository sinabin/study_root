---
sticker: emoji//1f343
---
# Spring Boot 프로젝트 구조 리뷰

> "왜 대기업은 JAR 배포를 하는데 공공기관은 WAR를 써?" 라는 질문에서 시작해보자.

---

## Q: Spring Boot인데 왜 WAR로 배포해?

Spring Boot는 **"실행 가능한 JAR 하나로 끝내자"**라는 철학으로 만들어졌어.

```java
// WAR 배포를 위한 레거시 코드
public class FwSpringBootApplication extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(FwSpringBootApplication.class);
    }
}
```

하지만 WAR 배포는 이 철학을 거스르는 거야.

---

## Q: JAR랑 WAR가 뭐가 달라?

| 항목 | JAR (내장 톰캣) | WAR (외장 WAS) |
|------|----------------|----------------|
| **배포** | `java -jar app.jar` | WAS 설치 → 설정 → WAR 배포 |
| **환경 일관성** | 개발 = 테스트 = 운영 | "내 PC에선 됐는데요" 발생 |
| **컨테이너화** | Dockerfile 3줄로 끝 | 복잡한 WAS 이미지 필요 |
| **CI/CD** | 단순 | WAS 의존성으로 복잡 |

---

## Q: 그럼 대기업은 뭘 써?

```
네이버, 카카오, 쿠팡, 배민 → 내장 톰캣 + Docker/Kubernetes
토스, 카카오뱅크 (금융권) → 역시 내장 톰캣 + Kubernetes
```

**"보안 때문에 외장 WAS를 써야 한다"는 건 기술적 근거 없는 미신이야.** 금융권도 내장 톰캣 쓰거든.

---

## Q: WEB-INF/lib에 JAR 파일 직접 넣는 건 뭐야?

```
demo1/
├── WEB-INF/
│   └── lib/
│       ├── HikariCP-5.0.1.jar
│       ├── jackson-core-2.15.4.jar
│       └── ... (100개 이상의 JAR)
```

이건 **Vendoring** 방식이야. 라이브러리를 직접 프로젝트에 복사해두는 거지.

---

## Q: 왜 그렇게 해? npm install처럼 하면 안 돼?

**인터넷 연결이 차단된 내부망 환경**에서는 Maven Central에 접근할 수 없거든.

```groovy
// 내부망에서의 레거시 방식
repositories {
    flatDir { dirs 'WEB-INF/lib' }
}
dependencies {
    implementation fileTree(dir: '/WEB-INF/lib', include: ['*.jar'])
}
```

### 이 방식의 문제점

| 문제 | 설명 |
|------|------|
| 버전 관리 불가 | build.gradle만 봐서는 버전을 알 수 없음 |
| 의존성 충돌 | 전이 의존성(transitive dependency) 해결 불가 |
| 보안 취약점 대응 어려움 | 취약한 라이브러리 추적이 어려움 |

---

## Q: 그럼 대기업 내부망에서는 어떻게 해?

**사설 Nexus/Artifactory**를 구축해.

```
[인터넷] → [DMZ의 Nexus Proxy] → [내부망 Nexus Mirror]
                                        ↓
                                 [개발자 PC]
```

```groovy
// 대기업 방식
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'com.zaxxer:HikariCP:5.0.1'
}

repositories {
    maven { url 'http://internal-nexus.company.local/repository/maven-public/' }
}
```

**Nexus 구축이 가능하면 무조건 이 방식이 더 좋아.** 버전 관리, 보안 취약점 대응 모든 면에서.

---

## Q: application.yml에 비밀번호 평문으로 넣어도 돼?

```yaml
# 현재 상태 - 절대 하면 안 되는 방식
spring:
  datasource:
    hikari:
      jdbc-url: jdbc:mariadb://172.16.50.5:13306/smart
      password: Passw0rd!  # Git 히스토리에 영구 기록됨!
```

**12-Factor App 원칙 위반이야.** 설정은 환경 변수에 저장해야 해.

### 올바른 방식

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

```bash
# 실행 시 환경 변수 주입 (Kubernetes)
kubectl create secret generic db-credentials \
  --from-literal=DB_URL=jdbc:mariadb://db:3306/smart \
  --from-literal=DB_PASSWORD=Passw0rd!
```

---

## Q: 전자정부 프레임워크는 왜 쓰는 거야?

```java
import org.egovframe.rte.fdl.cmmn.trace.LeaveaTrace;

@Bean
public LeaveaTrace leaveaTrace() {
    return new LeaveaTrace();
}
```

전자정부 프레임워크는 **2008년에 만들어졌어.** 당시에는 의미가 있었지만...

| 문제 | 설명 |
|------|------|
| Spring Boot와 중복 | 대부분 기능이 Spring Boot에 이미 포함 |
| 업데이트 지연 | 최신 Spring 버전 지원이 느림 |
| 인력 풀 제한 | 전자정부 프레임워크 경험자만 투입 가능 |

**대기업은 전자정부 프레임워크를 안 써.** 순수 Spring Boot만 사용해.

---

## Q: 그럼 패키지 구조는 어떻게 해야 해?

### 레거시 방식 (피해야 함)

```
com/biz/inw/inm/fnm/
├── FacilityManagementController.java
├── FacilityManagementMapper.java
└── FacilityManagementService.java  # 다 같은 폴더에...
```

`inw/inm/fnm` 같은 약어는 의미 파악이 어려워.

### 대기업 방식 (권장)

```
com/company/
├── domain/
│   └── facility/
│       ├── Facility.java
│       ├── FacilityRepository.java
│       └── FacilityService.java
├── application/
│   └── facility/
│       └── FacilityApplicationService.java
├── infrastructure/
│   └── persistence/
│       └── FacilityRepositoryImpl.java
└── interfaces/
    └── api/
        └── FacilityController.java
```

**계층별로 분리**하고 **명확한 네이밍**을 사용해.

---

## 핵심 요약

| 레거시 | 현대적 |
|--------|--------|
| WAR 배포 | JAR 배포 (내장 톰캣) |
| WEB-INF/lib | Maven/Gradle + Nexus |
| 비밀번호 평문 | 환경 변수 |
| 전자정부 프레임워크 | 순수 Spring Boot |
| 약어 패키지 | 도메인 중심 설계 |

---

## 오늘 당장 할 수 있는 것

1. **새 프로젝트는 JAR 배포로 시작**
2. **application.yml의 민감 정보 환경 변수로 이동**
3. **WEB-INF/lib 의존성을 Maven 좌표로 변환 시작**

---

## 이어서 읽기

- [[REST_API_설계_가이드_레거시vs현대]] - REST API 설계 패턴
- [[HTTP_STATUS_CODE_REFACTORING_PLAN]] - HTTP 상태 코드 적용
- [[ORACLE_CLOB_JSON_SERIALIZATION_ERROR]] - CLOB 직렬화 이슈
