---
sticker: emoji//1f300
---
# Vue/React 사이드 이펙트 처리 가이드

> "사이드 이펙트가 뭐야?"부터 시작해서 watch와 useEffect를 완전히 이해해보자.

---

## Q: 사이드 이펙트(Side Effect)가 뭐야?

**"함수의 주 목적 외에 발생하는 외부 영향"**이야.

```javascript
// 순수 함수 - 사이드 이펙트 없음
function add(a, b) {
  return a + b  // 입력만으로 출력 결정
}

// 사이드 이펙트 있는 함수
function addAndLog(a, b) {
  console.log('계산 시작')  // 콘솔 출력 = 사이드 이펙트
  return a + b
}
```

### 프론트엔드에서 흔한 사이드 이펙트

| 종류 | 예시 |
|------|------|
| **API 호출** | `fetch('/api/users')` |
| **DOM 조작** | `document.title = 'New Title'` |
| **타이머** | `setTimeout`, `setInterval` |
| **구독** | WebSocket, EventListener |
| **로컬 스토리지** | `localStorage.setItem()` |

---

## Q: 왜 사이드 이펙트를 따로 관리해야 해?

렌더링과 사이드 이펙트를 분리하지 않으면 문제가 생겨.

```javascript
// 잘못된 예 - 렌더링할 때마다 API 호출!
function UserProfile({ userId }) {
  fetch(`/api/users/${userId}`)  // 무한 요청 가능!
  return <div>...</div>
}
```

그래서:
- **Vue**는 `watch`, `watchEffect`
- **React**는 `useEffect`

를 사용해서 사이드 이펙트를 **적절한 시점에** 실행해.

---

## Q: API 호출은 어떻게 해?

### React

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null)

    useEffect(() => {
        // userId가 바뀔 때마다 실행
        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => setUser(data))
    }, [userId])  // 의존성

    return <div>{user?.name}</div>
}
```

### Vue

```javascript
setup(props) {
    const user = ref(null)

    watch(() => props.userId, async (newUserId) => {
        const res = await fetch(`/api/users/${newUserId}`)
        user.value = await res.json()
    }, { immediate: true })  // 최초에도 실행

    return { user }
}
```

---

## Q: 클린업(Cleanup)이 뭐야? 왜 필요해?

**메모리 누수**와 **의도치 않은 동작**을 방지하기 위해.

### 문제 상황

```javascript
useEffect(() => {
  // userId가 1일 때 요청 시작 (응답 3초 걸림)
  fetch(`/api/users/${userId}`)
    .then(res => res.json())
    .then(data => setUser(data))

  // 2초 후 userId가 2로 변경되면?
  // → userId 1의 응답이 나중에 도착해서 덮어씀!
}, [userId])
```

### 해결: AbortController

```javascript
useEffect(() => {
  const controller = new AbortController()

  fetch(`/api/users/${userId}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => setUser(data))

  // 클린업: 다음 effect 실행 전에 이전 요청 취소
  return () => controller.abort()
}, [userId])
```

---

## Q: 타이머는 어떻게 정리해?

### React

```javascript
useEffect(() => {
  const timerId = setInterval(() => {
    console.log('tick')
  }, 1000)

  // 클린업: 언마운트 시 타이머 정리
  return () => clearInterval(timerId)
}, [])
```

### Vue

```javascript
let timerId = null

onMounted(() => {
  timerId = setInterval(() => {
    console.log('tick')
  }, 1000)
})

onBeforeUnmount(() => {
  clearInterval(timerId)  // 정리!
})
```

---

## Q: 이벤트 리스너는?

### React

```javascript
useEffect(() => {
  const handleResize = () => {
    console.log('창 크기:', window.innerWidth)
  }

  window.addEventListener('resize', handleResize)

  // 클린업
  return () => window.removeEventListener('resize', handleResize)
}, [])
```

### Vue

```javascript
const handleResize = () => {
  console.log('창 크기:', window.innerWidth)
}

onMounted(() => {
  window.addEventListener('resize', handleResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', handleResize)  // 정리!
})
```

---

## Q: 디바운스 검색은 어떻게 해?

사용자가 타이핑을 멈춘 후 일정 시간 뒤에 검색하는 패턴이야.

### React

```javascript
const [keyword, setKeyword] = useState('')
const [results, setResults] = useState([])

useEffect(() => {
  if (!keyword.trim()) {
    setResults([])
    return
  }

  // 300ms 후에 검색 실행
  const timer = setTimeout(() => {
    fetch(`/api/search?q=${keyword}`)
      .then(res => res.json())
      .then(data => setResults(data))
  }, 300)

  // 키워드가 바뀌면 이전 타이머 취소
  return () => clearTimeout(timer)
}, [keyword])
```

### Vue

```javascript
const keyword = ref('')
const results = ref([])
let debounceTimer = null

watch(keyword, (newKeyword) => {
  // 이전 타이머 취소
  if (debounceTimer) clearTimeout(debounceTimer)

  if (!newKeyword.trim()) {
    results.value = []
    return
  }

  // 300ms 후에 검색
  debounceTimer = setTimeout(async () => {
    const res = await fetch(`/api/search?q=${newKeyword}`)
    results.value = await res.json()
  }, 300)
})
```

---

## Q: 로컬 스토리지 동기화는?

상태가 바뀔 때마다 로컬 스토리지에 저장하는 패턴이야.

### React

```javascript
// 초기값: 로컬 스토리지에서 불러오기
const [cart, setCart] = useState(() => {
  const saved = localStorage.getItem('cart')
  return saved ? JSON.parse(saved) : []
})

// cart가 바뀔 때마다 저장
useEffect(() => {
  localStorage.setItem('cart', JSON.stringify(cart))
}, [cart])
```

### Vue

```javascript
const saved = localStorage.getItem('cart')
const cart = ref(saved ? JSON.parse(saved) : [])

// cart가 바뀔 때마다 저장
watch(cart, (newCart) => {
  localStorage.setItem('cart', JSON.stringify(newCart))
}, { deep: true })  // 배열 내부 변경도 감지
```

---

## Q: 클린업을 안 하면 어떻게 돼?

```
- 메모리 누수
- 이벤트 중복 등록
- 타이머 계속 실행
- 언마운트된 컴포넌트에 상태 업데이트 시도 → 에러
```

---

## 사용 사례 정리

| 사용 사례 | 클린업 필요? |
|----------|-------------|
| API 호출 | (진행 중인 요청 취소) |
| DOM 조작 | O (인스턴스 정리) |
| 이벤트 리스너 | **O (필수!)** |
| 타이머 | **O (필수!)** |
| 로컬 스토리지 | X |
| 문서 제목 | (원래 제목 복원) |

---

## 핵심 요약

```
useEffect / watch = "렌더링 후 외부와 상호작용하는 창구"

┌─────────┐      ┌─────────┐      ┌─────────────────┐
│  상태   │  →   │ 렌더링  │  →   │ useEffect/watch │
│  변경   │      │         │      │ (부수효과 실행)  │
└─────────┘      └─────────┘      └─────────────────┘
    원인            결과              결과
```

---

## 이어서 읽기

- [[vue-ref-vs-react-usestate]] - 상태 관리 기초
- [[React_Vue_Reactivity_Misconceptions]] - 반응성 오해와 진실
- [[Vue_vs_React_Stability]] - 프레임워크 선택 가이드
- [[FrontEnd 빌드관련 이슈]] - 프론트엔드 빌드 이해하기
