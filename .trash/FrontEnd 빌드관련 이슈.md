FrontEnd 빌드 관련 이슈

> "npm install은 뭐야?", "왜 인터넷이 필요해?" 같은 질문에서 시작해서 빌드 전체를 이해해보자.

---

## Q: 프론트엔드 라이브러리를 가져오는 방법이 뭐가 있어?

크게 3가지 방식이 있어.

```
┌─────────────────┬─────────────────────┬─────────────────────────────────────┐
│   CDN 방식      │    벤더링 방식        │       npm + 번들러 방식              │
├─────────────────┼─────────────────────┼─────────────────────────────────────┤
│ 외부 서버에서    │ 로컬에 파일 직접     │  npm install + Vite/Webpack        │
│ 직접 로드        │ 복사하여 포함        │  으로 번들링                        │
└─────────────────┴─────────────────────┴─────────────────────────────────────┘
```

---

## Q: CDN 방식이 뭐야?

외부 서버에서 라이브러리를 직접 로드하는 방식이야.

```html
<!-- 외부 CDN 서버에서 직접 로드 -->
<script src="https://cdn.jsdelivr.net/npm/vue@3/dist/vue.global.js"></script>
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
```

| 장점 | 단점 |
|------|------|
| 설정 간단 | 인터넷 연결 필수 |
| 브라우저 캐시 활용 | 외부 서버 의존성 |
| 빌드 과정 불필요 | **내부망 환경에서 사용 불가** |

---

## Q: 그럼 내부망에서는 어떻게 해?

**벤더링(Vendoring)** 방식을 써. 라이브러리 파일을 직접 프로젝트에 복사해두는 거야.

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

| 장점 | 단점 |
|------|------|
| 인터넷 연결 불필요 | 라이브러리 업데이트 수동 관리 |
| 내부망 환경 가능 | 버전 관리 어려움 |
| 빌드 과정 불필요 | 용량이 Git에 포함됨 |

> "Vendoring"은 Go 언어 등에서 `vendor/` 폴더에 의존성을 복사하는 방식을 부르는 공식 용어야.

---

## Q: npm 방식은 뭐가 달라?

`npm install`로 패키지를 다운로드하고, Vite나 Webpack으로 번들링하는 방식이야.

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

| 장점 | 단점 |
|------|------|
| 의존성 버전 관리 용이 | 빌드 과정 필요 |
| Tree-shaking 최적화 | Node.js 환경 필요 |
| 모듈 시스템 활용 | 설정 복잡 |

---

## Q: package.json, node_modules, package-lock.json 이게 다 뭐야?

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
| `node_modules/` | 실제 다운로드된 코드 (수백 MB) | **일반적으로 X** |

---

## Q: npm install이랑 npm run build의 차이는?

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
| `npm install` | **O** | registry.npmjs.org에서 패키지 다운로드 |
| `npm install <패키지>` | **O** | 새 패키지 추가 |
| `npm run dev` | X | 로컬 개발 서버 실행 |
| `npm run build` | X | 프로덕션 빌드 |

---

## Q: 그럼 내부망(인터넷 차단)에서는 npm을 못 써?

**문제**: `npm install`이 외부 registry에 접속해야 함

**해결 방법**:

| 방법 | 설명 |
|------|------|
| node_modules Git 포함 | 의존성을 Git에 직접 커밋 |
| 내부 npm 미러 서버 | Verdaccio, Nexus 등으로 사내 registry 구축 |
| 외부망에서 빌드 후 복사 | 빌드된 결과물만 내부망으로 이동 |

### node_modules를 Git에 포함하면?

```bash
# 내부망에서 가능
npm run dev      # 개발 서버
npm run build    # 프로덕션 빌드

# 여전히 불가
npm install xxx  # 새 패키지 추가
```

---

## Q: dist 폴더가 뭐야?

`npm run build`의 결과물이야. 브라우저가 실행할 수 있는 최종 파일들.

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

---

## Q: dist를 Git에 포함해야 해?

상황에 따라 달라.

| 상황 | 권장 | 이유 |
|------|------|------|
| CI/CD 파이프라인 있음 | Git 제외 | 빌드 서버에서 자동 생성 |
| **내부망 환경** | **Git 포함** | 빌드 환경 구성 어려움 |
| 수정 거의 없는 프로젝트 | **Git 포함** | 빌드 복잡성 제거 |

### .gitignore 설정

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

## Q: Spring Boot에서 프론트엔드 빌드를 통합할 수 있어?

Gradle에서 npm 빌드를 자동화할 수 있어.

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

### 빌드 흐름

```
gradle build
    │
    ├── 1. npm install (node_modules 생성)
    ├── 2. npm run build (dist 생성)
    ├── 3. dist → static 폴더로 복사
    └── 4. Java 컴파일 + JAR 패키징
```

---

## Q: 무거운 라이브러리(PDF.js 등)는 어떻게 관리해?

**iframe으로 분리**하는 방법이 있어.

```
┌─────────────────────────────────────────────────────────────────┐
│                    메인 애플리케이션                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    <iframe>                               │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │         별도 Vue 애플리케이션 (SPA)                  │  │  │
│  │  │    PDF.js, Fabric.js 등 무거운 라이브러리           │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### iframe 사용의 장점

| 장점 | 설명 |
|------|------|
| **라이브러리 격리** | 무거운 라이브러리가 메인 앱 번들에 포함되지 않음 |
| **메모리 관리** | iframe 닫으면 관련 메모리 자동 해제 |
| **장애 격리** | iframe 내 오류가 메인 앱에 영향 없음 |

### iframe 통신: postMessage

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

## 상황별 권장 방식 요약

| 상황 | 권장 방식 |
|------|-----------|
| 간단한 프로젝트, 빠른 시작 | CDN 방식 |
| 내부망 환경, 빌드 환경 없음 | 벤더링 방식 |
| 대규모 프로젝트, CI/CD 있음 | npm + 번들러 + Gradle 통합 |
| 무거운 라이브러리 격리 필요 | iframe + 별도 SPA |
| 내부망 + 별도 SPA | node_modules/dist Git 포함 |

---

## 용어 정리

| 용어 | 설명 |
|------|------|
| **CDN** | Content Delivery Network, 외부 서버에서 정적 파일 제공 |
| **Vendoring** | 외부 의존성을 프로젝트 내부에 직접 복사하여 포함 |
| **Bundling** | 여러 파일을 하나로 합치는 과정 (Vite, Webpack) |
| **Tree-shaking** | 사용하지 않는 코드를 빌드에서 제거하는 최적화 |
| **SPA** | Single Page Application, 단일 HTML에서 동적으로 화면 전환 |
| **IIFE** | Immediately Invoked Function Expression, 즉시 실행 함수 |

---

## 이어서 읽기

- [[vue-ref-vs-react-usestate]] - Vue와 React의 상태 관리 비교
- [[Vue_vs_React_Stability]] - 두 프레임워크의 안정성 비교
