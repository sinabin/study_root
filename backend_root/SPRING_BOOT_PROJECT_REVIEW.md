# Spring Boot 프로젝트 구조 비판적 리뷰: 레거시 관행에서 벗어나기
> 대기업 Java 개발자 관점에서 바라본 공공기관 SI 프로젝트의 구조적 문제점과 개선 방향

---

## 들어가며

최근 한 공공기관의 Spring Boot 프로젝트를 분석할 기회가 있었다. Spring Boot라는 현대적인 프레임워크를 사용하면서도, 10년 전 개발 관행을 그대로 답습하고 있는 모습이 안타까웠다. 이 글에서는 해당 프로젝트의 구조적 문제점을 짚어보고, 네이버/카카오/쿠팡/배민 등 대기업에서 일반적으로 채택하는 방식과 비교해본다.

---

## 1. WAR 배포: Spring Boot의 철학을 거스르는 선택

### 현재 상태

```groovy
// build.gradle
plugins {
    id "java"
    id "war"  // WAR 플러그인 사용
}

war {
    archiveFileName = 'ROOT.war'
    archiveVersion = '0.0.0'
    webAppDirName = 'dummyFolder'
}
```

```java
// FwSpringBootApplication.java
public class FwSpringBootApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        // 외장 톰캣을 위한 설정
        return builder.sources(FwSpringBootApplication.class);
    }
}
```

### 문제점
**Spring Boot는 "실행 가능한 JAR 하나로 끝내자"라는 철학으로 탄생했다.** WAR 배포는 이 철학을 정면으로 거스르는 선택이다.

| 항목 | JAR (내장 톰캣) | WAR (외장 WAS) |
|------|----------------|----------------|
| **배포** | `java -jar app.jar` | WAS 설치 → 설정 → WAR 배포 |
| **환경 일관성** | 개발 = 테스트 = 운영 | WAS 버전/설정 차이로 "내 PC에선 됐는데요" 발생 |
| **컨테이너화** | Dockerfile 3줄로 끝 | 복잡한 WAS 이미지 구성 필요 |
| **CI/CD** | 단순함 | WAS 의존성으로 파이프라인 복잡 |

### 대기업은 어떻게 하나?

- **네이버, 카카오, 쿠팡, 배민**: 내장 톰캣 + 컨테이너(Docker/Kubernetes)
- **토스, 카카오뱅크 (금융권)**: 내장 톰캣 + Kubernetes

금융권도 내장 톰캣을 사용한다. "보안 때문에 외장 WAS를 써야 한다"는 것은 기술적 근거가 없는 미신에 가깝다.

### 권장 개선안

```groovy
// build.gradle - JAR 배포로 전환
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
}

// WAR 플러그인 제거, SpringBootServletInitializer 상속 제거
```

---

## 2. WEB-INF/lib: 의존성 관리 방식의 트레이드오프

### 현재 상태

```
demo1/
├── WEB-INF/
│   └── lib/
│       ├── HikariCP-5.0.1.jar
│       ├── jackson-core-2.15.4.jar
│       ├── lombok-1.18.32.jar
│       └── ... (100개 이상의 JAR 파일)
```

```groovy
// build.gradle
repositories {
    flatDir {
        dirs 'WEB-INF/lib'  // 로컬 디렉토리에서 JAR 로드
    }
}

dependencies {
    implementation fileTree(dir: '/WEB-INF/lib', include: ['*.jar'])
}
```

### 이 방식의 문제점

1. **버전 관리 불가**: 어떤 버전의 라이브러리를 쓰는지 `build.gradle`만 봐서는 알 수 없음
2. **의존성 충돌**: 전이 의존성(transitive dependency) 해결 불가
3. **보안 취약점 대응 어려움**: 취약한 라이브러리 버전을 추적하기 어려움
4. **Git 저장소 비대화**: 바이너리 파일이 Git에 포함되어 저장소 크기 급증
5. **재현성 부재**: 다른 개발자가 같은 환경을 구성하기 어려움

### 그러나 이 방식의 장점도 있다: 내부망(폐쇄망) 환경

**인터넷 연결이 차단된 내부망 환경**에서는 이 방식이 현실적인 선택일 수 있다:

- 인터넷 연결 없이 바로 빌드 가능
- 의존성을 물리적으로 보유
- 별도의 인프라 구축 없이 단순하게 동작

공공기관, 금융권, 군/국방 등 **보안상 외부 네트워크 접근이 제한된 환경**에서는 Maven Central에 접근할 수 없다. 이런 환경에서 `WEB-INF/lib` 방식은 나름의 합리성이 있다.

### 대기업은 어떻게 하나?

대기업도 내부망 환경이 있다. 하지만 **내부망에 사설 Nexus/Artifactory를 구축**하여 해결한다:

```
[인터넷] → [DMZ의 Nexus Proxy] → [내부망 Nexus Mirror]
                                        ↓
                                 [개발자 PC]
```

```groovy
// 내부망 환경에서의 의존성 관리
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.zaxxer:HikariCP:5.0.1'

    // 사내 라이브러리는 사설 Maven Repository 사용
    implementation 'com.company:internal-auth:1.2.3'
}

repositories {
    // mavenCentral() 대신 내부망 Nexus만 참조
    maven { url 'http://internal-nexus.company.local/repository/maven-public/' }
}
```

- **Nexus, Artifactory** 같은 사설 Maven Repository를 내부망에 구축
- 필요한 라이브러리를 주기적으로 내부망으로 동기화
- 모든 의존성은 `build.gradle`에 명시적으로 선언
- Dependabot, Snyk 등으로 취약점 자동 스캔

### Nexus 구축 가능 여부에 따른 판단

| 환경 | 권장 방식 |
|------|----------|
| **Nexus 구축 가능** | Maven 좌표 기반 의존성 관리 (대기업 방식) |
| **Nexus 구축 불가** | `WEB-INF/lib` 방식 유지 + 문서화 강화 |

**Nexus를 구축할 수 있다면, 무조건 대기업 방식이 더 좋다.** 버전 관리, 보안 취약점 대응, 의존성 충돌 해결 모든 면에서 우월하다.

하지만 현실적으로 Nexus 구축이 어려운 환경이라면 (예산, 인력, 조직 정책 등), `WEB-INF/lib` 방식도 차선책으로 수용할 수 있다.

### 권장 개선안

**Nexus 구축이 가능한 경우:**
1. 내부망에 Nexus 또는 Artifactory 도입
2. `WEB-INF/lib`의 모든 JAR를 Maven 좌표로 변환
3. 사내 전용 라이브러리는 사설 Repository에 배포
4. `WEB-INF/lib` 디렉토리 삭제

**Nexus 구축이 불가능한 경우:**
1. 의존성 목록 문서화 (버전, 용도, 보안점검일 기록)
2. 보안 취약점 주기적 수동 점검
3. JAR 파일명에 버전 명시 (예: `HikariCP-5.0.1.jar`)
4. 중장기적으로 Nexus 도입 검토

```markdown
# dependencies.md (Nexus 없이 관리할 경우 필수)
| 라이브러리 | 버전 | 용도 | 보안취약점 점검일 |
|-----------|------|------|------------------|
| HikariCP | 5.0.1 | DB 커넥션 풀 | 2024-01-15 |
| Jackson | 2.15.4 | JSON 처리 | 2024-01-15 |
| ... | ... | ... | ... |
```

---

## 3. 설정 파일의 하드코딩: 12-Factor App 위반

### 현재 상태

```yaml
# application.yml
spring:
  datasource:
    hikari:
      main:
        jdbc-url: jdbc:log4jdbc:mariadb://172.16.50.5:13306/smart
        username: smart
        password: Passw0rd!  # 비밀번호 평문 노출
```

```java
// FwSpringBootApplication.java
String path = "C:\\demo\\config\\config.properties";  // 경로 하드코딩

if (!osName.contains("win")) {
    path = "/home/tomcat/config/config.properties";
}
```

### 문제점

1. **비밀번호 평문 노출**: Git 히스토리에 영구 기록
2. **경로 하드코딩**: 운영 환경 변경 시 코드 수정 필요
3. **OS별 분기 처리**: 코드 복잡도 증가, 테스트 어려움
4. **환경별 설정 파일 난립**: `application_dev.yml`, `application_prod.yml` 등

### 12-Factor App 원칙

> **III. Config**: 설정은 환경 변수에 저장하라

### 대기업은 어떻게 하나?

```yaml
# application.yml - 환경 변수 참조
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

```bash
# 실행 시 환경 변수 주입 (Kubernetes 예시)
kubectl create secret generic db-credentials \
  --from-literal=DB_URL=jdbc:mariadb://db:3306/smart \
  --from-literal=DB_USERNAME=smart \
  --from-literal=DB_PASSWORD=Passw0rd!
```

- **Spring Cloud Config**: 설정 중앙화
- **HashiCorp Vault**: 시크릿 관리
- **Kubernetes Secrets**: 컨테이너 환경 시크릿 관리

### 권장 개선안

1. 모든 민감 정보를 환경 변수로 외부화
2. Spring Cloud Config 또는 AWS Secrets Manager 도입
3. 하드코딩된 경로 제거
4. `.gitignore`에 민감 설정 파일 추가

---

## 4. 전자정부 프레임워크: 2024년에 필요한가?

### 현재 상태

```java
import org.egovframe.rte.fdl.cmmn.trace.LeaveaTrace;

@Bean
public LeaveaTrace leaveaTrace() {
    return new LeaveaTrace();
}
```

### 문제점

전자정부 프레임워크는 2008년에 만들어졌다. 당시에는 의미가 있었지만, 현재는:

1. **Spring Boot와 중복**: 대부분의 기능이 Spring Boot에 이미 포함
2. **업데이트 지연**: 최신 Spring 버전 지원이 느림
3. **불필요한 의존성**: `LeaveaTrace` 같은 컴포넌트의 실제 활용도 의문
4. **인력 풀 제한**: 전자정부 프레임워크 경험자만 투입 가능

### 대기업은 어떻게 하나?

순수 Spring Boot만 사용한다. 전자정부 프레임워크를 쓰는 대기업은 없다.

### 권장 개선안

1. 전자정부 프레임워크 의존성 제거
2. 순수 Spring Boot로 마이그레이션
3. 필요한 공통 기능은 자체 라이브러리로 개발

---

## 5. 프로젝트 구조: 관심사 분리 부재

### 현재 상태

```
src/main/java/com/biz/
├── inw/inm/fnm/
│   ├── FacilityManagementController.java
│   ├── FacilityManagementMapper.java
│   └── FacilityManagementService.java
├── si/cc/cc/
│   ├── CommonCodeManagementController.java
│   ├── CommonCodeManagementMapper.java
│   └── CommonCodeManagementService.java
```

### 문제점

1. **패키지 구조 불명확**: `inw/inm/fnm`, `si/cc/cc` 같은 약어의 의미 파악 어려움
2. **계층형 아키텍처 부재**: Controller-Service-Mapper가 같은 패키지에 혼재
3. **도메인 분리 미흡**: 비즈니스 로직과 인프라 코드 결합

### 대기업은 어떻게 하나?

**Layered Architecture** 또는 **Hexagonal Architecture** 적용:

```
src/main/java/com/company/
├── domain/
│   ├── facility/
│   │   ├── Facility.java
│   │   ├── FacilityRepository.java
│   │   └── FacilityService.java
│   └── inspection/
│       └── ...
├── application/
│   └── facility/
│       └── FacilityApplicationService.java
├── infrastructure/
│   ├── persistence/
│   │   └── FacilityRepositoryImpl.java
│   └── external/
│       └── ...
└── interfaces/
    └── api/
        └── FacilityController.java
```

### 권장 개선안

1. 명확한 패키지 네이밍 컨벤션 수립
2. 계층별 패키지 분리
3. 도메인 중심 설계 도입

---

## 결론: 기술 부채를 방치하지 말자

이 프로젝트의 문제점은 "기술적으로 불가능한 것"이 아니다. 모두 개선 가능하다.

문제는 **관성**이다:
- "원래 이렇게 해왔으니까"
- "운영팀이 안 된다고 하니까"
- "바꾸면 장애 나면 누가 책임지나"

하지만 기술 부채는 복리로 쌓인다. 지금 바꾸지 않으면 3년 후에는 10배의 비용이 든다.

### 오늘 당장 할 수 있는 것

1. **새 프로젝트는 JAR 배포로 시작**
2. **application.yml의 민감 정보 환경 변수로 이동**
3. **WEB-INF/lib 의존성을 Maven 좌표로 변환 시작**

### 중기적으로 해야 할 것

1. **사설 Maven Repository(Nexus) 도입**
2. **Docker/Kubernetes 도입 검토**
3. **전자정부 프레임워크 의존성 제거**

### 장기적으로 해야 할 것

1. **아키텍처 현대화 (MSA 또는 Modular Monolith)**
2. **CI/CD 파이프라인 고도화**
3. **인프라 코드화 (IaC)**

---

## 마치며

> "그렇게 하면 보안에 문제가 있다"
> "우리 조직은 특수해서 안 된다"
> "예전부터 이렇게 해왔다"

이런 말을 들으면 한 번 더 질문해보자: **"정말 기술적으로 안 되는 건가요, 아니면 바꾸기 싫은 건가요?"**

네이버, 카카오, 쿠팡, 배민, 토스, 카카오뱅크 모두 내장 톰캣을 쓴다. 금융권도 쓴다. 보안 인증도 받는다. 대규모 트래픽도 처리한다.

**기술적으로 안 되는 게 아니라, 조직이 안 바뀌는 것이다.**

변화는 어렵다. 하지만 개발자로서 더 나은 방향을 제시하고, 설득하고, 조금씩 바꿔나가는 것이 우리의 역할이 아닐까.

---

*이 글은 특정 조직을 비난하려는 의도가 아닌, 더 나은 개발 문화를 위한 기술적 제언입니다.*
