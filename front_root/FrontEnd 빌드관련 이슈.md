## 개요  
  
이 문서는 Spring Boot + Vue.js 프로젝트에서 프론트엔드 의존성 관리와 빌드/배포 방식을 정리한 가이드입니다.  
  
---  
  
## 1. 프론트엔드 의존성 관리 방식 비교  
  
### 1.1 세 가지 방식 개요  
  
```  
┌─────────────────────────────────────────────────────────────────────────────┐  
│                        프론트엔드 라이브러리 관리 방식                          │  
├─────────────────┬─────────────────────┬─────────────────────────────────────┤  
│   CDN 방식      │    벤더링 방식        │       npm + 번들러 방식              │  
├─────────────────┼─────────────────────┼─────────────────────────────────────┤  
│ 외부 서버에서    │ 로컬에 파일 직접     │  npm install + Vite/Webpack        │  
│ 직접 로드        │ 복사하여 포함        │  으로 번들링                        │  
└─────────────────┴─────────────────────┴─────────────────────────────────────┘  
```  
  
### 1.2 CDN 방식  
  
```html  
<!-- 외부 CDN 서버에서 직접 로드 -->  
<script src="https://cdn.jsdelivr.net/npm/vue@3/dist/vue.global.js"></script>  
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>  
```  
  
| 장점 | 단점 |  
|------|------|  
| 설정 간단 | 인터넷 연결 필수 |  
| 브라우저 캐시 활용 | 외부 서버 의존성 |  
| 빌드 과정 불필요 | 내부망 환경에서 사용 불가 |  
  
### 1.3 벤더링(Vendoring) 방식  
  
```html  
<!-- 로컬에 저장된 라이브러리 파일 로드 -->  
<script src="/js/vendor/vue.global.js"></script>  
<script src="/js/vendor/axios.min.js"></script>  
```  
  
```  
프로젝트/  
└── js/  
    └── vendor/           # 라이브러리 파일 직접 포함  
        ├── vue.global.js  
        ├── axios.min.js  
        └── primevue.min.js  
```  
  
| 장점         | 단점               |
| ---------- | ---------------- |
| 인터넷 연결 불필요 | 라이브러리 업데이트 수동 관리 |
| 내부망 환경 가능  | 버전 관리 어려움        |
| 빌드 과정 불필요  | 용량이 Git에 포함됨     |
  
> **참고**: "Vendoring"은 Go 언어 등에서 `vendor/` 폴더에 의존성을 복사하는 방식을 부르는 공식 용어입니다.  
  
### 1.4 npm + 번들러 방식  
  
```bash  
npm install vue axios primevue  
npm run build  
```  
  
```  
프로젝트/  
├── package.json          # 의존성 목록  
├── node_modules/         # 다운로드된 라이브러리  
├── src/                  # 소스 코드  
└── dist/                 # 빌드 결과물  
```  
  
| 장점               | 단점            |
| ---------------- | ------------- |
| 의존성 버전 관리 용이     | 빌드 과정 필요      |
| Tree-shaking 최적화 | Node.js 환경 필요 |
| 모듈 시스템 활용        | 설정 복잡         |
  
---  
  
## 2. npm 핵심 개념  
  
### 2.1 주요 파일/폴더 역할  
  
```  
프로젝트/  
├── package.json          # 의존성 "목록" (레시피)  
├── package-lock.json     # 정확한 버전 잠금  
└── node_modules/         # 실제 다운로드된 라이브러리 (재료)  
```  
  
| 파일/폴더 | 역할 | Git 포함 여부 |  
|-----------|------|---------------|  
| `package.json` | 필요한 라이브러리 목록 정의 | O |  
| `package-lock.json` | 정확한 버전 고정 | O |  
| `node_modules/` | 실제 다운로드된 코드 (수백 MB) | 일반적으로 X |  
  
### 2.2 npm 명령어와 인터넷 연결  
  
```  
┌─────────────────────────────────────────────────────────────────┐  
│                     npm install                                  │  
│                         │                                        │  
│                         ▼                                        │  
│   registry.npmjs.org ──────────> node_modules/ 다운로드           │  
│        (인터넷 필요)                                              │  
└─────────────────────────────────────────────────────────────────┘  
  
┌─────────────────────────────────────────────────────────────────┐  
│                  npm run dev / build                             │  
│                         │                                        │  
│                         ▼                                        │  
│   node_modules/에서 로컬 실행 (인터넷 불필요)                       │  
└─────────────────────────────────────────────────────────────────┘  
```  
  
| 명령어 | 인터넷 필요 | 설명 |  
|--------|-------------|------|  
| `npm install` | O | registry.npmjs.org에서 패키지 다운로드 |  
| `npm install <패키지>` | O | 새 패키지 추가 |  
| `npm run dev` | X | 로컬 개발 서버 실행 |  
| `npm run build` | X | 프로덕션 빌드 |  
  
### 2.3 내부망(인터넷 차단) 환경 대응  
  
**문제**: `npm install`이 외부 registry에 접속해야 함  
  
**해결 방법**:  
  
| 방법 | 설명 |  
|------|------|  
| node_modules Git 포함 | 의존성을 Git에 직접 커밋 |  
| 내부 npm 미러 서버 | Verdaccio, Nexus 등으로 사내 registry 구축 |  
| 외부망에서 빌드 후 복사 | 빌드된 결과물만 내부망으로 이동 |  
  
**node_modules를 Git에 포함하면**:  
  
```bash  
# 내부망에서 가능  
npm run dev      # 개발 서버 ✅  
npm run build    # 프로덕션 빌드 ✅  
  
# 여전히 불가  
npm install xxx  # 새 패키지 추가 ❌  
```  
  
---  
  
## 3. 빌드 결과물(dist) 관리  
  
### 3.1 dist 폴더란?  
  
```bash  
npm run build  
```  
  
```  
src/                      # 소스 코드 (개발자가 작성)  
  ├── App.vue  
  ├── main.ts  
  └── components/  
  
        │ 빌드 (Vite/Webpack)  
        ▼  
  
dist/                     # 빌드 결과물 (브라우저가 실행)  
  ├── index.html  
  └── assets/  
      ├── index.js        # 번들링된 JavaScript  
      └── index.css       # 번들링된 CSS  
```  
  
### 3.2 dist를 Git에 포함할까?  
  
| 상황 | 권장 | 이유 |  
|------|------|------|  
| CI/CD 파이프라인 있음 | Git 제외 | 빌드 서버에서 자동 생성 |  
| 내부망 환경 | **Git 포함** | 빌드 환경 구성 어려움 |  
| 수정 거의 없는 프로젝트 | **Git 포함** | 빌드 복잡성 제거 |  
  
### 3.3 .gitignore 설정  
  
**일반적인 경우 (dist 제외)**:  
```gitignore  
node_modules/  
dist/  
```  
  
**내부망 대응 (dist 포함)**:  
```gitignore  
# node_modules/  # 주석 처리하여 포함  
# dist/          # 주석 처리하여 포함  
*.log  
```  
  
---  
  
## 4. Spring Boot + 프론트엔드 통합 빌드  
### 4.1 Gradle에서 npm 빌드 통합  
```groovy  
// build.gradle  
  
plugins {  
    id 'com.github.node-gradle.node' version '3.5.1'  
}  
  
// npm install 실행  
task npmInstall(type: NpmTask) {  
    args = ['install']  
}  
  
// npm run build 실행  
task npmBuild(type: NpmTask) {  
    dependsOn npmInstall  
    args = ['run', 'build']  
}  
  
// 빌드 결과물을 Spring Boot static 폴더로 복사  
task copyFrontend(type: Copy) {  
    dependsOn npmBuild  
    from 'frontend/dist'  
    into 'src/main/resources/static'  
}  
  
// Java 빌드 전에 프론트엔드 빌드 실행  
bootJar {  
    dependsOn copyFrontend  
}  
```  
### 4.2 빌드 흐름  
  
```  
gradle build  
    │  
    ├── 1. npm install (node_modules 생성)  
    ├── 2. npm run build (dist 생성)  
    ├── 3. dist → static 폴더로 복사  
    └── 4. Java 컴파일 + JAR 패키징  
```  

### 4.3 통합 빌드 vs 벤더링 비교 

| 구분 | Gradle 통합 빌드 | 벤더링 방식 |  
|------|------------------|-------------|  
| 빌드 과정 | 복잡 (npm + Gradle) | 단순 (Gradle만) |  
| CI/CD 환경 | Node.js 필요 | Node.js 불필요 |  
| 라이브러리 업데이트 | `npm update` | 수동 파일 교체 |  
| 내부망 대응 | 추가 설정 필요 | 바로 가능 |  
  
---  
  
## 5. iframe을 활용한 프론트엔드 분리  
  
### 5.1 왜 iframe을 사용하는가?  
  
무거운 라이브러리(PDF.js, Fabric.js 등)를 사용하는 기능을 분리할 때:  
  
```  
┌─────────────────────────────────────────────────────────────────┐  
│                    메인 애플리케이션                              │  
│  ┌───────────────────────────────────────────────────────────┐  │  
│  │                      팝업 컴포넌트                         │  │  
│  │  ┌─────────────────────────────────────────────────────┐  │  │  
│  │  │                    <iframe>                         │  │  │  
│  │  │  ┌───────────────────────────────────────────────┐  │  │  │  
│  │  │  │         별도 Vue 애플리케이션 (SPA)            │  │  │  │  
│  │  │  │  ┌─────────────┐  ┌─────────────────────────┐ │  │  │  │  
│  │  │  │  │  무거운     │  │     무거운              │ │  │  │  │  
│  │  │  │  │  라이브러리1 │  │     라이브러리2         │ │  │  │  │  
│  │  │  │  └─────────────┘  └─────────────────────────┘ │  │  │  │  
│  │  │  └───────────────────────────────────────────────┘  │  │  │  
│  │  └─────────────────────────────────────────────────────┘  │  │  
│  └───────────────────────────────────────────────────────────┘  │  
└─────────────────────────────────────────────────────────────────┘  
```  
  
### 5.2 iframe 사용의 장점  
  
| 장점 | 설명 |  
|------|------|  
| **라이브러리 격리** | 무거운 라이브러리가 메인 앱 번들에 포함되지 않음 |  
| **메모리 관리** | iframe 닫으면 관련 메모리 자동 해제 |  
| **장애 격리** | iframe 내 오류가 메인 앱에 영향 없음 |  
| **독립 배포** | iframe 앱만 별도로 수정/배포 가능 |  
| **기술 스택 분리** | 다른 빌드 시스템 사용 가능 |  
  
### 5.3 iframe 통신: postMessage  
  
```javascript  
// 부모 → iframe  
iframe.contentWindow.postMessage({ type: 'loadData', data: {...} }, '*');  
  
// iframe → 부모  
window.parent.postMessage({ type: 'saveComplete', success: true }, '*');  
  
// 메시지 수신  
window.addEventListener('message', (event) => {  
    // origin 검증 (보안)  
    if (event.origin !== 'https://trusted-domain.com') return;  
  
    const { type, data } = event.data;  
    // 처리...  
});  
```  
  
---  
  
## 6. 실제 적용 사례: demo1 프로젝트  
### 6.1 현재 구조  
  
```  
demo1/  
├── build.gradle                    # Java/Spring만 빌드  
├── src/main/webapp/  
│   ├── js/  
│   │   ├── vendor/                 # 벤더링 방식 (Vue, PrimeVue 등)  
│   │   ├── util/                   # 공통 유틸리티  
│   │   ├── app/                    # 앱 코어  
│   │   └── biz/                    # 업무 컴포넌트 (IIFE 패턴)  
│   │  
│   └── pdf-viewer/                 # 별도 Vue SPA (iframe으로 로드)  
│       ├── src/                    # 소스 코드  
│       ├── node_modules/           # 의존성 (Git 포함 - 내부망 대응)  
│       ├── dist/                   # 빌드 결과물 (Git 포함)  
│       └── package.json  
```  
  
### 6.2 왜 이렇게 구성했는가?  
  
| 결정 | 이유 |  
|------|------|  
| 메인 앱은 벤더링 방식 | 빌드 과정 없이 단순하게 운영 |  
| pdf-viewer는 별도 SPA | PDF.js, Fabric.js 등 무거운 라이브러리 격리 |  
| iframe으로 통합 | 필요할 때만 로드, 메모리 관리 용이 |  
| node_modules Git 포함 | 내부망에서도 빌드 가능 |  
| dist Git 포함 | Gradle에 npm 빌드 통합 불필요 |  
  
### 6.3 수정 시 작업 흐름  
  
```bash  
# pdf-viewer 소스 수정 후  
cd src/main/webapp/pdf-viewer  
  
# 빌드 (node_modules 이미 있으므로 npm install 불필요)  
npm run build  
  
# 커밋  
git add .  
git commit -m "pdf-viewer 수정 및 빌드"  
```  
  
---  
  
## 7. 요약: 상황별 권장 방식  
  
| 상황 | 권장 방식 |  
|------|-----------|  
| 간단한 프로젝트, 빠른 시작 | CDN 방식 |  
| 내부망 환경, 빌드 환경 없음 | 벤더링 방식 |  
| 대규모 프로젝트, CI/CD 있음 | npm + 번들러 + Gradle 통합 |  
| 무거운 라이브러리 격리 필요 | iframe + 별도 SPA |  
| 내부망 + 별도 SPA | node_modules/dist Git 포함 |  
  
---  
  
## 8. 용어 정리  
  
| 용어 | 설명 |  
|------|------|  
| **CDN** | Content Delivery Network, 외부 서버에서 정적 파일 제공 |  
| **Vendoring** | 외부 의존성을 프로젝트 내부에 직접 복사하여 포함 |  
| **Bundling** | 여러 파일을 하나로 합치는 과정 (Vite, Webpack) |  
| **Tree-shaking** | 사용하지 않는 코드를 빌드에서 제거하는 최적화 |  
| **SPA** | Single Page Application, 단일 HTML에서 동적으로 화면 전환 |  
| **IIFE** | Immediately Invoked Function Expression, 즉시 실행 함수 |  
| **node_modules** | npm install로 다운로드된 라이브러리 저장 폴더 |  
| **dist** | distribution, 빌드된 배포용 결과물 폴더 |  
| **registry** | npm 패키지 저장소 (registry.npmjs.org) |  
  
---