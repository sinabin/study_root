---
sticker: emoji//1f504
---
# Vue ref vs React useState 비교

> 실제 코드에서 시작해서 "그럼 이건 뭐야?" 식으로 이해해보자.

---

## Q: `const userCanvasData = ref<UserCanvasInfo[]>` 이게 뭐야?

Vue 3에서 반응형 상태를 선언하는 방식이야.

```typescript
import { ref } from 'vue'

const userCanvasData = ref<UserCanvasInfo[]>([])
```

여기서 `ref`는 React의 `useState`랑 비슷한 역할을 해.

---

## Q: 그럼 React에서 똑같이 선언하면?

```typescript
import { useState } from 'react'

const [userCanvasData, setUserCanvasData] = useState<UserCanvasInfo[]>([])
```

배열 구조분해로 `[값, setter함수]`를 받아.

---

## Q: 값 접근하고 변경하는 게 어떻게 달라?

| 항목 | Vue `ref` | React `useState` |
|------|-----------|------------------|
| 값 접근 | `page.value` | `page` |
| 값 변경 | `page.value = newValue` | `setPage(newValue)` |
| 템플릿/JSX에서 | `.value` 자동 언래핑 | 그대로 사용 |

### Vue - 값 변경

```typescript
// 단순 할당
page.value = 5

// 배열 수정
userCanvasData.value = userCanvasData.value.map(user =>
  user.userId === id ? { ...user, enabled: !user.enabled } : user
)

// 중첩 객체 직접 수정 가능 (반응성 유지됨!)
userCanvasData.value[0].enabled = false
```

### React - 값 변경

```typescript
// setter 함수 호출 필수
setPage(5)

// 배열 수정 - prev 콜백 패턴
setUserCanvasData(prev => prev.map(user =>
  user.userId === id ? { ...user, enabled: !user.enabled } : user
))

// 중첩 객체 - 불변성 유지 필수 (새 객체 생성해야 함)
setUserCanvasData(prev => [
  { ...prev[0], enabled: false },
  ...prev.slice(1)
])
```

---

## Q: 값 변경 코드만 봐도 Vue가 훨씬 직관적인데?

맞아. 특히 중첩 객체 수정할 때 차이가 커.

```typescript
// Vue - 직접 수정 가능 (반응성 유지됨)
userCanvasData.value[0].enabled = false

// React - 불변성 유지 필수 (새 객체 생성해야 함)
setUserCanvasData(prev => [
  { ...prev[0], enabled: false },
  ...prev.slice(1)
])
```

### 그럼 왜 React는 setter를 강제해?

| | Vue | React |
|---|---|---|
| **장점** | 직관적, 코드 간결 | 변경 추적 명확, 디버깅 용이 |
| **단점** | 스크립트에서 `.value` 붙여야 함 | 보일러플레이트 많음 |
| **철학** | Proxy로 자동 감지 | 명시적 상태 변경 |

React가 setter를 강제하는 이유: **언제 상태가 바뀌었는지 명시적으로 알 수 있어서** 디버깅이나 상태 흐름 파악이 쉬워.

Vue는 Proxy로 자동 감지하니까 편하지만, 어디서 값이 바뀌었는지 추적이 상대적으로 어려울 수 있어.

---

## Q: 그럼 실제 함수로 비교하면?

### Vue

```typescript
const page = ref(1)
const canvasData = ref<Record<number, any>>({})

function handlePageUpdate(newPage: number) {
  page.value = newPage
}

function handleCanvasDataUpdate(newData: Record<number, any>) {
  canvasData.value = newData
}
```

### React

```typescript
const [page, setPage] = useState(1)
const [canvasData, setCanvasData] = useState<Record<number, any>>({})

function handlePageUpdate(newPage: number) {
  setPage(newPage)
}

function handleCanvasDataUpdate(newData: Record<number, any>) {
  setCanvasData(newData)
}
```

단순 할당은 거의 비슷한데, 상태가 많아지면 React는 `setXxx` 함수가 계속 늘어나. Vue는 `.value = ` 패턴 하나로 통일되고.

---

## Q: `function handlePageUpdate(newPage: number)` 이건 TypeScript 문법이야?

네, [[TypeScript_기초|TypeScript]] 문법이야.

```typescript
// TypeScript - 파라미터 타입 명시
function handlePageUpdate(newPage: number) {
  page.value = newPage
}

// JavaScript - 타입 없음
function handlePageUpdate(newPage) {
  page.value = newPage
}
```

`: number`는 **타입 어노테이션**이야. 이 함수에 숫자가 아닌 값을 넣으면 컴파일 에러가 나.

```typescript
handlePageUpdate(5)       // OK
handlePageUpdate("5")     // 에러: string은 number에 할당 불가
handlePageUpdate(null)    // 에러
```

### 그럼 반환 타입도 명시할 수 있어?

```typescript
// 반환값 없음
function handlePageUpdate(newPage: number): void {
  page.value = newPage
}

// 값을 반환하는 경우
function getNextPage(current: number): number {
  return current + 1
}
```

---

## Q: 그럼 언제 Vue를 쓰고 언제 React를 써?

### Vue가 유리한 경우
- 단순 CRUD 앱
- 중첩된 객체 상태가 많을 때
- 빠른 프로토타이핑

### React가 유리한 경우
- 복잡한 상태 흐름 관리
- 상태 변경 추적이 중요할 때
- 팀 내 코드 리뷰 시 명시성 필요

---

## 참고: UserCanvasInfo 타입

```typescript
interface UserCanvasInfo {
  userId: string
  userName: string
  enabled: boolean
  createdDt: string
  canvasData: Record<number, any>  // 페이지별 캔버스 데이터
}
```

---

## 이어서 읽기

- [[React_Vue_Reactivity_Misconceptions]] - Vue/React 반응성 오해와 진실
- [[React_Vue_SideEffects_Guide]] - 사이드 이펙트 처리 비교
- [[Vue_vs_React_Stability]] - 두 프레임워크의 안정성 비교
