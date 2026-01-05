---
sticker: emoji//1f6e0-fe0f
---
#  nFilter(보안 키패드) 호환성 패치 내역 요약

본 가이드는 Spring Boot 3.x(Jakarta EE) 환경에서 레거시 nFilter(Java EE)를 구동하기 위한 수정 사항을 담고 있습니다.

## 1. 배경 (Why?)
- **프로젝트 환경:** Spring Boot 3.2.5 / Tomcat 10+ (**jakarta.** 패키지 사용)
- **nFilter 라이브러리:** 구형 **javax.** 패키지 기반으로 작성됨
- **문제점:** 타입 불일치로 인한 컴파일 에러 및 리소스 누락으로 인한 초기화 실패(500 Error)

## 2. 주요 패치 내역 (What & How?)

### ① Servlet API 호환성 어댑터 적용
- **파일:** `ServletApiAdapter.java` 및 `open_nFilter_keypad_manager.jsp`
- **내용:** `jakarta` 객체를 `javax` 인터페이스로 실시간 변환(Proxy)해주는 어댑터 적용
- **이유:** nFilter 내부 로직에 `javax.*` 객체를 안전하게 전달하기 위함

### ② 라이브러리 교체 (리소스 충돌 해결)
- **추가:** `tomcat-servlet-api-9.0.87.jar`
- **이유:** `GenericServlet` 초기화 시 필요한 리소스(`LocalStrings.properties`)를 확보하여 `ExceptionInInitializerError` 해결

### ③ 필수 의존성 추가 (Quartz)
- **추가:** `quartz-2.3.2.jar`
- **이유:** nFilter 내부 세션 매니저가 Quartz를 사용하므로 누락 시 `ClassNotFoundException` 발생

### ④ 보안 필터 및 CSRF 예외 처리
- **파일:** `SettingInitalizer.java`
- **내용:** `PERMIT_ALL_PATTERNS` 및 `PERMIT_CSRF_PATTERNS`에 `/nfilter/**` 추가
- **이유:** 인증되지 않은 상태에서 키패드 JSP 및 리소스에 접근할 수 있도록 허용

### ⑤ JSP 컴파일 에러 및 타입 모호성 해결
- **내용:** JSP 상단 `javax.servlet` import 제거
- **이유:** `jakarta`와 `javax` 클래스명이 중복되어 발생하는 `Ambiguous type` 에러 방지

---

## 3. ⚠️ 로컬 개발자(Windows) 주의 사항
- **현상:** 현재 프로젝트에 리눅스용(`libNSaferJNI.so`)만 있고 **윈도우용 DLL(`NSaferJNI.dll`)이 없어** 로컬에서는 키패드가 뜨지 않음.
  
- **대처:** JSP에 `try-catch` 패치를 적용해 두었으므로, **키패드만 안 뜰 뿐 로그인 기능은 정상 사용 가능**. (운영 배포 시 리눅스 서버에서는 정상 작동함)
  
- **필요 시:** `NSaferJNI.dll` 파일을 구하여 JDK bin 폴더에 넣으면 로컬에서도 키패드가 작동
