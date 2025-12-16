---
sticker: emoji//1f4a1
---
# Vue/React 반응성 오해와 진실

> "useEffect가 리렌더링을 발생시킨다?" - 아니야!

---

## Q: useEffect가 리렌더링을 일으키는 거 아냐?

**아니야!** 순서가 반대야.

### 잘못된 이해

```
useEffect의 의존성 배열 값이 변경됨
    ↓
useEffect가 리렌더링을 시킴  ← 틀림!
```

### 올바른 이해

```
상태(state)가 변경됨
    ↓
컴포넌트가 리렌더링됨  ← 상태 변경이 원인!
    ↓
화면 업데이트
    ↓
useEffect 실행  ← 리렌더링 "후"에 실행됨
```

---

## Q: 그럼 누가 리렌더링을 일으키는 거야?

| 프레임워크 | 리렌더링 발생시키는 것 | 리렌더링 후 실행되는 것 |
|-----------|---------------------|---------------------|
| **React** | `setState` | `useEffect` |
| **Vue** | `ref.value` 변경 | `watch` |

핵심:
```
useEffect와 watch는 리렌더링의 "원인"이 아니라 "결과"다
```

---

## Q: 실제 동작 순서를 보여줘

### React

```javascript
function Counter() {
    const [count, setCount] = useState(0)

    useEffect(() => {
        console.log('3. useEffect 실행! count:', count)
    }, [count])

    function handleClick() {
        setCount(count + 1)  // 1. 상태 변경 → 리렌더링 트리거
    }

    console.log('2. 렌더링 중... count:', count)

    return <button onClick={handleClick}>클릭: {count}</button>
}
```

버튼 클릭 시 출력 순서:
```
2. 렌더링 중... count: 1
3. useEffect 실행! count: 1
```

### Vue

```javascript
setup() {
    const count = ref(0)

    watch(count, (newVal) => {
        console.log('3. watch 실행!', newVal)
    })

    function handleClick() {
        count.value++  // 1. 값 변경 → 리렌더링 트리거
    }

    return { count, handleClick }
}
```

---

## Q: "불변성"이 뭐야? React에서 왜 중요해?

**불변성(Immutability)**: 기존 데이터를 직접 수정하지 않고, 새 데이터를 만드는 것.

### React가 상태 변경을 감지하는 방법

```javascript
const oldState = { count: 0 }
const newState = { count: 1 }

oldState === newState  // false → 다른 객체 → 리렌더링!

// 직접 수정하면?
oldState.count = 1
oldState === oldState  // true → 같은 객체 → 리렌더링 안 함
```

React는 **참조 비교**를 해. 참조가 같으면 "변경 없음"으로 판단해버려.

### Vue는 왜 직접 수정이 가능해?

```javascript
const state = reactive({ count: 0 })
state.count = 1  // Proxy가 이 동작을 가로채서 감지함
```

Vue의 Proxy는 **속성 접근 자체를 가로채기** 때문에 직접 수정해도 감지할 수 있어.

---

## Q: computed랑 useMemo는 뭐가 달라?

둘 다 "계산된 값을 캐싱"하는 건데, 사용 방식이 달라.

### Vue computed

```javascript
const firstName = ref('Kim')
const lastName = ref('철수')

// 의존성 자동 추적
const fullName = computed(() => {
  return firstName.value + ' ' + lastName.value
})
// firstName이나 lastName이 바뀌면 자동으로 재계산
```

### React useMemo

```javascript
const [firstName, setFirstName] = useState('Kim')
const [lastName, setLastName] = useState('철수')

// 의존성 명시 필수!
const fullName = useMemo(() => {
  return firstName + ' ' + lastName
}, [firstName, lastName])  // 이 배열을 빠뜨리면 버그!
```

### 핵심 차이

| | Vue computed | React useMemo |
|---|---|---|
| 의존성 | **자동 추적** | **수동 명시** |
| 실수 가능성 | 낮음 | 높음 (의존성 누락) |

---

## Q: watch랑 useEffect는?

둘 다 "값이 바뀌면 뭔가 하기"인데, 역시 사용 방식이 달라.

### Vue watch

```javascript
const userId = ref(1)

// userId가 바뀌면 fetchUser 실행
watch(userId, async (newId, oldId) => {
  console.log(`${oldId} → ${newId}`)  // 이전 값 접근 가능!
  const user = await fetchUser(newId)
})
```

### React useEffect

```javascript
const [userId, setUserId] = useState(1)

// userId가 바뀌면 fetchUser 실행
useEffect(() => {
  fetchUser(userId).then(user => {
    console.log(user)
  })
}, [userId])  // 의존성 배열 필수!
```

### 차이점

| | watch | useEffect |
|---|---|---|
| 이전 값 접근 | 기본 제공 | 별도 ref 필요 |
| 최초 실행 | 기본 X | 기본 O |
| 의존성 | 명시 또는 자동 | 수동 명시 |

---

## Q: 그럼 어떤 방식이 더 좋아?

| 관점 | Vue (Proxy) | React (불변성) |
|------|-------------|----------------|
| **편의성** | 직관적, 코드 짧음 | 보일러플레이트 많음 |
| **디버깅** | 어디서 바뀌었는지 추적 어려움 | 명시적이라 추적 쉬움 |
| **학습 곡선** | 낮음 | 높음 (불변성 개념) |
| **대규모 앱** | 복잡한 반응성 추적 필요 | 상태 흐름이 명확 |

---

## 핵심 정리

### 비유로 이해하기

```
상태 변경 = 주문 넣기
리렌더링 = 요리하기
useEffect/watch = 요리 완성 후 서빙하기

서빙(useEffect)이 요리(리렌더링)를 시키는 게 아니라,
주문(상태변경)이 요리(리렌더링)를 시키고,
요리가 끝나면 서빙(useEffect)이 실행된다.
```

### 역할 정리표

| 역할 | React | Vue | 리렌더링 발생? |
|------|-------|-----|--------------|
| 상태 저장 | `useState` | `ref`/`reactive` | - |
| 상태 변경 | `setState` | `.value = ` | **O (원인)** |
| 파생 값 | `useMemo` | `computed` | X |
| 부수효과 | `useEffect` | `watch` | X (결과) |

---

## 이어서 읽기

- [[vue-ref-vs-react-usestate]] - ref와 useState 기본 비교
- [[React_Vue_SideEffects_Guide]] - watch와 useEffect 심화
- [[Vue_vs_React_Stability]] - 프레임워크 선택 가이드
