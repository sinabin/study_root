# React & Vue 반응성 시스템: 흔한 오해와 진실

> "useEffect가 리렌더링을 발생시킨다?" - 아닙니다!

---

## 목차

1. [핵심 오해](#1-핵심-오해)
2. [실제 동작 원리](#2-실제-동작-원리)
3. [React 동작 흐름](#3-react-동작-흐름)
4. [Vue 동작 흐름](#4-vue-동작-흐름)
5. [역할별 정리](#5-역할별-정리)
6. [useEffect vs watch 비교](#6-useeffect-vs-watch-비교)
7. [실전 예제로 이해하기](#7-실전-예제로-이해하기)
8. [핵심 요약](#8-핵심-요약)

---

## 1. 핵심 오해

### 잘못된 이해

```
useEffect의 의존성 배열 값이 변경됨
    ↓
useEffect가 컴포넌트를 리렌더링시킴  ← 잘못된 이해!
    ↓
화면 업데이트
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

## 2. 실제 동작 원리

### 누가 리렌더링을 발생시키는가?

| 프레임워크 | 리렌더링 발생시키는 것 | 리렌더링 후 실행되는 것 |
|-----------|---------------------|---------------------|
| **React** | `setState` | `useEffect` |
| **Vue** | `ref`, `reactive` 값 변경 | `watch` |

### 핵심 포인트

```
┌─────────────────────────────────────────────────────────┐
│  useEffect와 watch는 리렌더링의 "원인"이 아니라 "결과"다  │
└─────────────────────────────────────────────────────────┘
```

---

## 3. React 동작 흐름

### 상태 변경부터 useEffect 실행까지

```javascript
function Counter() {
    const [count, setCount] = useState(0);  // 1️⃣ 상태 선언

    useEffect(() => {
        console.log('3️⃣ useEffect 실행! count:', count);
        document.title = `클릭 ${count}회`;
    }, [count]);

    function handleClick() {
        setCount(count + 1);  // 2️⃣ 상태 변경 → 리렌더링 트리거
    }

    console.log('렌더링 중... count:', count);

    return <button onClick={handleClick}>클릭: {count}</button>;
}
```

### 실행 순서

```
버튼 클릭
    ↓
setCount(1) 호출           ← 상태 변경 요청
    ↓
React가 리렌더링 스케줄링
    ↓
컴포넌트 함수 재실행        ← "렌더링 중... count: 1" 출력
    ↓
가상 DOM 비교 & 실제 DOM 업데이트
    ↓
화면에 "클릭: 1" 표시
    ↓
useEffect 실행             ← "3️⃣ useEffect 실행! count: 1" 출력
```

### 시각화

```
시간 →

[클릭] → [setState] → [리렌더링] → [DOM 업데이트] → [useEffect]
           원인          결과          결과            결과
```

---

## 4. Vue 동작 흐름

### 값 변경부터 watch 실행까지

```javascript
setup() {
    const count = ref(0);  // 1️⃣ 반응형 상태 선언

    watch(count, (newVal, oldVal) => {
        console.log('3️⃣ watch 실행!', oldVal, '→', newVal);
        document.title = `클릭 ${newVal}회`;
    });

    function handleClick() {
        count.value++;  // 2️⃣ 값 변경 → 리렌더링 트리거
    }

    return { count, handleClick };
}
```

### 실행 순서

```
버튼 클릭
    ↓
count.value++ 실행         ← 반응형 값 변경
    ↓
Vue 반응성 시스템이 변경 감지
    ↓
컴포넌트 리렌더링
    ↓
가상 DOM 비교 & 실제 DOM 업데이트
    ↓
화면에 "클릭: 1" 표시
    ↓
watch 콜백 실행            ← "3️⃣ watch 실행! 0 → 1" 출력
```

---

## 5. 역할별 정리

### React

| 역할 | 담당 | 설명 |
|------|------|------|
| **상태 저장** | `useState` | 컴포넌트의 데이터 저장소 |
| **리렌더링 트리거** | `setState` | 상태 변경 시 리렌더링 발생 |
| **파생 값 계산** | `useMemo` | 다른 값에서 계산되는 값 (캐싱) |
| **부수효과 처리** | `useEffect` | 렌더링 후 실행할 작업 |

```javascript
const [data, setData] = useState([]);           // 상태 저장
const total = useMemo(() => sum(data), [data]); // 파생 값

useEffect(() => {
    fetchData().then(setData);                  // 부수효과
}, []);

setData([1, 2, 3]);                             // 리렌더링 트리거
```

### Vue

| 역할 | 담당 | 설명 |
|------|------|------|
| **상태 저장** | `ref`, `reactive` | 반응형 데이터 저장소 |
| **리렌더링 트리거** | 값 변경 (자동) | `.value` 변경 시 자동 리렌더링 |
| **파생 값 계산** | `computed` | 다른 값에서 계산되는 값 (자동 캐싱) |
| **부수효과 처리** | `watch` | 값 변경 시 실행할 작업 |

```javascript
const data = ref([]);                           // 상태 저장
const total = computed(() => sum(data.value));  // 파생 값

watch(data, (newData) => {
    console.log('데이터 변경됨:', newData);     // 부수효과
});

data.value = [1, 2, 3];                         // 리렌더링 트리거 (자동)
```

---

## 6. useEffect vs watch 비교

### 기본 사용법

```javascript
// React
const [userId, setUserId] = useState(1);

useEffect(() => {
    fetchUser(userId);
}, [userId]);
```

```javascript
// Vue
const userId = ref(1);

watch(userId, (newId) => {
    fetchUser(newId);
});
```

### 차이점 비교

| 기능 | React useEffect | Vue watch |
|------|-----------------|-----------|
| 이전 값 접근 | 별도 ref 필요 | 기본 제공 |
| 최초 실행 | 기본적으로 실행됨 | `immediate: true` 필요 |
| 여러 값 감시 | 배열에 나열 | 배열로 감시 가능 |
| 깊은 감시 | 직접 구현 | `deep: true` 옵션 |
| 의존성 관리 | 수동 명시 | 자동 추적 |

### 이전 값 접근

```javascript
// React - 이전 값 접근이 번거로움
const [count, setCount] = useState(0);
const prevCountRef = useRef();

useEffect(() => {
    prevCountRef.current = count;  // 현재 값을 저장
});

const prevCount = prevCountRef.current;
console.log(`이전: ${prevCount}, 현재: ${count}`);
```

```javascript
// Vue - 이전 값이 기본 제공
const count = ref(0);

watch(count, (newVal, oldVal) => {
    console.log(`이전: ${oldVal}, 현재: ${newVal}`);
});
```

### 최초 실행

```javascript
// React - 마운트 시 기본 실행됨
useEffect(() => {
    console.log('마운트 시 + userId 변경 시 실행');
    fetchUser(userId);
}, [userId]);
```

```javascript
// Vue - 기본적으로 변경 시에만 실행
watch(userId, (newId) => {
    console.log('userId 변경 시에만 실행');
    fetchUser(newId);
});

// 마운트 시에도 실행하려면
watch(userId, (newId) => {
    fetchUser(newId);
}, { immediate: true });  // 옵션 추가
```

### 여러 값 감시

```javascript
// React
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');

useEffect(() => {
    console.log('이름 변경됨');
}, [firstName, lastName]);
```

```javascript
// Vue
const firstName = ref('');
const lastName = ref('');

watch([firstName, lastName], ([newFirst, newLast], [oldFirst, oldLast]) => {
    console.log('이름 변경됨');
    console.log(`이전: ${oldFirst} ${oldLast}`);
    console.log(`현재: ${newFirst} ${newLast}`);
});
```

### 깊은 감시 (중첩 객체)

```javascript
// React - 깊은 비교 직접 구현 필요
const [user, setUser] = useState({ profile: { name: '' } });

useEffect(() => {
    console.log('user 변경됨');
}, [JSON.stringify(user)]);  // 꼼수... 비효율적
```

```javascript
// Vue - deep 옵션 제공
const user = reactive({ profile: { name: '' } });

watch(user, (newUser) => {
    console.log('user 변경됨');
}, { deep: true });

// 또는 특정 경로만 감시
watch(
    () => user.profile.name,
    (newName) => {
        console.log('이름 변경됨:', newName);
    }
);
```

---

## 7. 실전 예제로 이해하기

### 시나리오: 사용자 검색 기능

#### React 버전

```javascript
function UserSearch() {
    // 1. 상태 (리렌더링의 원인)
    const [keyword, setKeyword] = useState('');
    const [users, setUsers] = useState([]);
    const [loading, setLoading] = useState(false);

    // 2. 파생 값 (리렌더링과 무관, 캐싱용)
    const userCount = useMemo(() => users.length, [users]);

    // 3. 부수효과 (리렌더링 "후" 실행)
    useEffect(() => {
        if (!keyword) {
            setUsers([]);
            return;
        }

        setLoading(true);

        const timer = setTimeout(() => {
            fetchUsers(keyword)
                .then(setUsers)
                .finally(() => setLoading(false));
        }, 300);  // 디바운스

        return () => clearTimeout(timer);  // 클린업
    }, [keyword]);  // keyword가 바뀔 때마다 실행

    return (
        <div>
            <input
                value={keyword}
                onChange={(e) => setKeyword(e.target.value)}
            />
            {loading ? <p>검색 중...</p> : <p>결과: {userCount}명</p>}
        </div>
    );
}
```

#### Vue 버전

```javascript
setup() {
    // 1. 상태 (리렌더링의 원인)
    const keyword = ref('');
    const users = ref([]);
    const loading = ref(false);

    // 2. 파생 값 (자동 추적)
    const userCount = computed(() => users.value.length);

    // 3. 부수효과 (값 변경 "후" 실행)
    watch(keyword, (newKeyword) => {
        if (!newKeyword) {
            users.value = [];
            return;
        }

        loading.value = true;

        // 디바운스는 watchDebounced 또는 직접 구현
        fetchUsers(newKeyword)
            .then(result => users.value = result)
            .finally(() => loading.value = false);
    });

    return { keyword, users, userCount, loading };
}
```

### 동작 흐름 비교

```
[사용자가 'kim' 입력]
        ↓
┌───────────────────────────────────────────────────────┐
│ React                      │ Vue                      │
├───────────────────────────────────────────────────────┤
│ setKeyword('kim')          │ keyword.value = 'kim'    │
│         ↓                  │         ↓                │
│ 컴포넌트 리렌더링            │ 컴포넌트 리렌더링          │
│         ↓                  │         ↓                │
│ DOM 업데이트                │ DOM 업데이트              │
│         ↓                  │         ↓                │
│ useEffect 실행             │ watch 실행               │
│         ↓                  │         ↓                │
│ API 호출                   │ API 호출                 │
│         ↓                  │         ↓                │
│ setUsers(결과)             │ users.value = 결과       │
│         ↓                  │         ↓                │
│ 컴포넌트 리렌더링            │ 컴포넌트 리렌더링          │
│         ↓                  │         ↓                │
│ 결과 화면에 표시            │ 결과 화면에 표시          │
└───────────────────────────────────────────────────────┘
```

---

## 8. 핵심 요약

### 한 줄 정리

```
┌─────────────────────────────────────────────────────────────┐
│  상태 변경 → 리렌더링 → useEffect/watch 실행                  │
│     원인        결과           결과                          │
└─────────────────────────────────────────────────────────────┘
```

### 역할 정리표

| 역할 | React | Vue | 리렌더링 발생? |
|------|-------|-----|--------------|
| 상태 저장 | `useState` | `ref`/`reactive` | - |
| 상태 변경 | `setState` | `.value = ` | **O (원인)** |
| 파생 값 | `useMemo` | `computed` | X |
| 부수효과 | `useEffect` | `watch` | X (결과) |

### 기억할 것

1. **`useEffect`와 `watch`는 리렌더링의 결과이지 원인이 아니다**
2. **리렌더링은 오직 상태 변경(`setState`, `ref.value`)에 의해 발생한다**
3. **부수효과 훅은 "렌더링이 완료된 후" 실행된다**

### 비유로 이해하기

```
상태 변경 = 주문 넣기
리렌더링 = 요리하기
useEffect/watch = 요리 완성 후 서빙하기

서빙(useEffect)이 요리(리렌더링)를 시키는 게 아니라,
주문(상태변경)이 요리(리렌더링)를 시키고,
요리가 끝나면 서빙(useEffect)이 실행된다.
```

---

*작성일: 2025-12-05*
