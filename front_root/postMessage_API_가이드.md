# postMessage API 완벽 가이드

## postMessage API란?

**서로 다른 윈도우(창) 또는 iframe 사이에서 안전하게 메시지를 주고받을 수 있게 해주는 브라우저 내장 API**입니다.

---

## 왜 필요한가?

### 일반적인 문제 상황

JavaScript는 **보안상 다른 윈도우나 iframe의 내부에 직접 접근할 수 없습니다** (Same-Origin Policy):

```javascript
// ❌ 불가능 - 보안 제약 (CORS 에러)
const iframeData = iframeRef.contentWindow.someData;
```

### postMessage 해결책

**postMessage를 사용하면 안전하게 데이터를 주고받을 수 있습니다**:

```javascript
// ✅ 가능 - postMessage 사용
iframeRef.contentWindow.postMessage({ type: 'getData' }, '*');
```

---

## 기본 문법

### 1. 메시지 보내기

```javascript
targetWindow.postMessage(message, targetOrigin);
```

**매개변수**:
- `message`: 전송할 데이터 (객체, 문자열 등)
- `targetOrigin`: 수신 대상의 origin (보안)
  - `'*'`: 모든 origin 허용 (권장하지 않음)
  - `'https://example.com'`: 특정 origin만 허용 (권장)

### 2. 메시지 받기

```javascript
window.addEventListener('message', function(event) {
    // ⚠️ 보안: 항상 origin 검증!
    if (event.origin !== 'https://trusted-site.com') {
        return;
    }

    // 메시지 처리
    console.log(event.data);
});
```

**event 객체 속성**:
- `event.data`: 전송된 메시지 데이터
- `event.origin`: 메시지를 보낸 출처 (예: `https://example.com`)
- `event.source`: 메시지를 보낸 window 객체

---

## 실전 예제: PdfViewerOverlay.js

### 전체 흐름

```
부모 창 (PdfViewerOverlay.js)          iframe (PDF Viewer)
│                                      │
│  ① sendMessageToViewer()            │
│     postMessage() ─────────────────>│
│     { type: 'LOAD_PDF_BASE64' }     │
│                                      │
│                                      │ PDF 로딩...
│                                      │
│  ② handleMessage()                  │
│   <───────────────── postMessage()  │
│     { type: 'PDF_LOADED' }          │
│                                      │
```

### 1️⃣ 메시지 보내기 (부모 → iframe)

```javascript
/**
 * iframe으로 메시지 전송
 * @param {Object} message - 전송할 메시지 객체
 */
function sendMessageToViewer(message) {
    if (!iframeRef.value || !iframeRef.value.contentWindow) {
        console.warn('iframe이 준비되지 않았습니다.');
        return;
    }

    try {
        // iframe의 contentWindow에 메시지 전송
        iframeRef.value.contentWindow.postMessage(
            message,                    // 전송할 데이터
            window.location.origin      // 보안: 현재 사이트 origin만 허용
        );
    } catch (e) {
        console.error('메시지 전송 실패:', e);
    }
}

// 실제 사용 예
sendMessageToViewer({
    type: 'LOAD_PDF_BASE64',
    data: {
        base64: pdfData.value.base64,
        fileName: 'document.pdf',
        readOnly: true
    }
});
```

### 2️⃣ 메시지 받기 (iframe → 부모)

```javascript
// 허용된 origin 목록 (보안)
const ALLOWED_ORIGINS = [
    window.location.origin  // 현재 사이트와 같은 도메인만 허용
];

/**
 * origin 검증 (postMessage 보안)
 * 악의적인 사이트에서 보낸 가짜 메시지를 차단하기 위해 메시지 출처를 검증합니다.
 *
 * @param {string} origin - 검증할 origin
 * @returns {boolean}
 */
function isAllowedOrigin(origin) {
    return ALLOWED_ORIGINS.includes(origin);
}

/**
 * 뷰어에서 받은 메시지 처리 (origin 검증 포함)
 * @param {MessageEvent} event - 메시지 이벤트
 */
function handleMessage(event) {
    // ⚠️ 보안: origin 검증 (필수!)
    if (!isAllowedOrigin(event.origin)) {
        console.warn('허용되지 않은 origin:', event.origin);
        return;
    }

    const { type, canvasData, success, message } = event.data || {};

    if (!type) return;

    // 메시지 타입별 처리
    switch (type) {
        case 'VIEWER_READY':
            // 뷰어 준비 완료
            viewerReady.value = true;
            loadPdfToViewer();
            break;

        case 'PDF_LOADED':
            // PDF 로드 완료
            loading.value = false;
            break;

        case 'CANVAS_DATA_CHANGED':
            // Canvas 데이터 변경
            if (canvasData) {
                try {
                    canvasData.value = JSON.parse(canvasData);
                    hasUnsavedChanges.value = true;
                } catch (e) {
                    console.error('Canvas 데이터 파싱 실패:', e);
                }
            }
            break;

        case 'SAVE_CANVAS_RESPONSE':
            // 저장 응답
            if (success) {
                handleSaveSuccess(canvasData);
            } else {
                ShowError({ title: '저장 실패', html: message });
            }
            break;

        default:
            // 알 수 없는 메시지 타입은 무시
            break;
    }
}

// 이벤트 리스너 등록 (컴포넌트 마운트 시)
onMounted(() => {
    window.addEventListener('message', handleMessage);
});

// 이벤트 리스너 해제 (컴포넌트 언마운트 시)
onBeforeUnmount(() => {
    window.removeEventListener('message', handleMessage);
});
```

---

## 보안 주의사항

### ⚠️ 절대 하지 말아야 할 것

```javascript
// ❌ 나쁜 예: origin 검증 없이 모든 메시지 처리
window.addEventListener('message', function(event) {
    eval(event.data.code);  // 위험! 악의적인 코드 실행 가능
});

// ❌ 나쁜 예: 모든 origin 허용
targetWindow.postMessage(data, '*');
```

### ✅ 반드시 해야 할 것

```javascript
// ✅ 좋은 예: origin 검증 + 안전한 처리
window.addEventListener('message', function(event) {
    // 1. origin 검증 (필수!)
    if (event.origin !== 'https://trusted-site.com') {
        return;
    }

    // 2. 데이터 검증
    if (!event.data || typeof event.data.type !== 'string') {
        return;
    }

    // 3. 안전한 데이터 처리
    processData(event.data);
});

// ✅ 좋은 예: 특정 origin만 허용
targetWindow.postMessage(data, 'https://trusted-site.com');
```

---

## 보안 시나리오

### ❌ 공격 시나리오 (origin 검증 없을 때)

1. 악의적인 사이트(`https://evil.com`)가 귀하의 사이트를 iframe으로 로드
2. 악의적인 JavaScript가 postMessage로 가짜 데이터 전송
3. 검증 없이 받으면 → **데이터 조작, XSS 공격 가능**

```javascript
// evil.com의 악의적인 코드
targetIframe.postMessage({
    type: 'ADMIN_COMMAND',
    action: 'deleteAllData'
}, '*');
```

### ✅ 보호 시나리오 (origin 검증 있을 때)

1. 메시지 수신 시 `event.origin` 확인
2. `https://your-site.com`이 아니면 거부
3. **신뢰할 수 있는 출처의 메시지만 처리**

```javascript
function handleMessage(event) {
    // origin 검증으로 evil.com의 메시지 차단
    if (event.origin !== 'https://your-site.com') {
        console.warn('차단된 origin:', event.origin);
        return;  // 거부!
    }

    // 안전한 메시지만 처리
    processMessage(event.data);
}
```

---

## 실용적인 사용 사례

### 1. 부모 ↔ iframe 통신
```javascript
// 부모 창
iframe.contentWindow.postMessage({ action: 'getData' }, origin);

// iframe
window.addEventListener('message', (e) => {
    if (e.data.action === 'getData') {
        e.source.postMessage({ result: data }, e.origin);
    }
});
```

### 2. 팝업 ↔ 부모 창 통신
```javascript
// 팝업 오픈
const popup = window.open('https://example.com/popup');
popup.postMessage({ userId: 123 }, 'https://example.com');

// 팝업에서 응답
window.opener.postMessage({ success: true }, origin);
```

### 3. Worker 통신 (Web Worker)
```javascript
// 메인 스레드
const worker = new Worker('worker.js');
worker.postMessage({ task: 'calculate', data: [1, 2, 3] });
worker.onmessage = (e) => console.log(e.data);

// worker.js
self.onmessage = (e) => {
    const result = calculate(e.data.data);
    self.postMessage({ result });
};
```

---

## 디버깅 팁

### 메시지 로깅

```javascript
window.addEventListener('message', (event) => {
    console.group('📨 postMessage 수신');
    console.log('Origin:', event.origin);
    console.log('Data:', event.data);
    console.log('Source:', event.source);
    console.groupEnd();

    // 실제 처리...
});
```

### Chrome DevTools에서 확인

1. **Console**: postMessage 이벤트 로그 확인
2. **Network > WS**: WebSocket과 달리 postMessage는 네트워크 탭에 안 보임
3. **Sources > Event Listener Breakpoints**: `message` 이벤트에 브레이크포인트 설정

---

## 참고 자료

- [MDN: Window.postMessage()](https://developer.mozilla.org/ko/docs/Web/API/Window/postMessage)
- [OWASP: HTML5 Security - postMessage](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#postmessage)
- [Can I Use: postMessage](https://caniuse.com/mdn-api_window_postmessage)

---

## 요약 체크리스트

- [ ] **postMessage**는 윈도우/iframe 간 안전한 통신을 위한 브라우저 API
- [ ] **보내기**: `window.postMessage(data, targetOrigin)`
- [ ] **받기**: `window.addEventListener('message', handler)`
- [ ] **보안**: 항상 `event.origin` 검증 필수!
- [ ] **targetOrigin**: 가능한 한 구체적으로 지정 (`'*'` 지양)
- [ ] **데이터 검증**: 받은 데이터의 타입과 내용 검증
- [ ] **리스너 정리**: 컴포넌트 언마운트 시 `removeEventListener` 호출

---

## 간단 비유

- **일반 함수 호출**: 같은 집 안에서 직접 대화
- **postMessage**: 서로 다른 집(윈도우/iframe)에 사는 사람들이 편지(메시지)를 주고받음
  - 우체국(브라우저)이 안전하게 배달
  - 받는 사람은 보낸 사람 주소(origin)를 확인해야 함 (보안!)
