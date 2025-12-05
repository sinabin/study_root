# Vue vs React: Vue가 더 안정적인 이유

> React 경험자를 위한 Vue의 장점 분석

---

## 목차

1. [의존성 자동 추적](#1-의존성-자동-추적)
2. [반응성 시스템의 명확성](#2-반응성-시스템의-명확성)
3. [라이프사이클 관리](#3-라이프사이클-관리)
4. [상태 변경의 일관성](#4-상태-변경의-일관성)
5. [양방향 바인딩](#5-양방향-바인딩)
6. [조건부 렌더링](#6-조건부-렌더링)
7. [리스트 렌더링](#7-리스트-렌더링)
8. [Props 기본값 처리](#8-props-기본값-처리)
9. [비동기 상태 업데이트](#9-비동기-상태-업데이트)
10. [결론](#결론)

---

## 1. 의존성 자동 추적

### Vue - 자동 추적

```javascript
const price = ref(1000);
const quantity = ref(3);
const taxRate = ref(0.1);

// 의존성을 명시할 필요 없음 - Vue가 자동으로 추적
const total = computed(() => {
    const subtotal = price.value * quantity.value;
    return subtotal + (subtotal * taxRate.value);
});

// price, quantity, taxRate 중 하나라도 바뀌면 자동 재계산
```

### React - 수동 명시 필요

```javascript
const [price, setPrice] = useState(1000);
const [quantity, setQuantity] = useState(3);
const [taxRate, setTaxRate] = useState(0.1);

// 의존성 배열을 직접 관리해야 함
const total = useMemo(() => {
    const subtotal = price * quantity;
    return subtotal + (subtotal * taxRate);
}, [price, quantity, taxRate]);  // 빼먹으면 버그!
```

### 발생할 수 있는 실수 (React)

```javascript
// 실수: taxRate를 의존성 배열에서 빼먹음
const total = useMemo(() => {
    const subtotal = price * quantity;
    return subtotal + (subtotal * taxRate);  // taxRate 사용하는데...
}, [price, quantity]);  // taxRate 없음! → 세율 바뀌어도 total 안 바뀜
```

---

## 2. 반응성 시스템의 명확성

### Vue - `.value`로 명확한 구분

```javascript
const count = ref(0);
const user = reactive({ name: '홍길동', age: 25 });

// ref는 .value로 접근 (명확함)
count.value++;
console.log(count.value);

// reactive는 직접 접근
user.age = 26;
console.log(user.name);
```

### React - 규칙을 알아야 함

```javascript
const [count, setCount] = useState(0);
const [user, setUser] = useState({ name: '홍길동', age: 25 });

// 이렇게 하면 안됨 (에러는 안 나지만 동작 안 함)
count++;  // X - 화면 업데이트 안됨
user.age = 26;  // X - 화면 업데이트 안됨

// 이렇게 해야 함
setCount(count + 1);  // O
setUser({ ...user, age: 26 });  // O - 새 객체 생성 필요
```

### 발생할 수 있는 실수 (React)

```javascript
// 실수 1: 직접 변경 (초보자가 자주 하는 실수)
const [items, setItems] = useState([1, 2, 3]);
items.push(4);  // 화면 안 바뀜, 에러도 안 남

// 실수 2: 객체 뮤테이션
const [form, setForm] = useState({ name: '', email: '' });
form.name = '홍길동';  // 화면 안 바뀜

// 올바른 방법
setItems([...items, 4]);
setForm({ ...form, name: '홍길동' });
```

---

## 3. 라이프사이클 관리

### Vue - 직관적인 이름의 분리된 훅

```javascript
// 마운트 시
onMounted(() => {
    console.log('컴포넌트가 DOM에 마운트됨');
    fetchData();
});

// 언마운트 직전
onBeforeUnmount(() => {
    console.log('컴포넌트가 곧 제거됨');
    cleanup();
});

// 업데이트 후
onUpdated(() => {
    console.log('컴포넌트가 업데이트됨');
});
```

### React - useEffect 하나로 모든 것을 처리

```javascript
// 마운트 + 언마운트
useEffect(() => {
    console.log('마운트됨');
    fetchData();

    return () => {
        console.log('언마운트됨');  // cleanup 함수 (까먹기 쉬움)
        cleanup();
    };
}, []);  // 빈 배열 (빼먹으면 무한루프)

// 업데이트 감지
useEffect(() => {
    console.log('count가 바뀜');
}, [count]);  // 의존성 배열
```

### 발생할 수 있는 실수 (React)

```javascript
// 실수 1: 의존성 배열 누락 → 무한 루프
useEffect(() => {
    fetchData();
    setData(result);  // 상태 변경 → 리렌더 → useEffect 재실행 → 무한루프
});  // 의존성 배열 없음!

// 실수 2: cleanup 함수 누락 → 메모리 누수
useEffect(() => {
    const timer = setInterval(() => {
        console.log('tick');
    }, 1000);
    // return () => clearInterval(timer);  // 이거 빼먹으면 메모리 누수
}, []);

// 실수 3: 빈 배열 vs 배열 없음 혼동
useEffect(() => { ... }, []);   // 마운트 시 1번만
useEffect(() => { ... });       // 매 렌더마다 실행 (위험!)
```

---

## 4. 상태 변경의 일관성

### Vue - 직접 변경 가능

```javascript
const form = reactive({
    name: '',
    email: '',
    address: {
        city: '',
        street: ''
    }
});

// 직관적인 변경
form.name = '홍길동';
form.address.city = '서울';

// 배열도 마찬가지
const items = ref([1, 2, 3]);
items.value.push(4);  // 그냥 push 해도 됨
items.value[0] = 100; // 인덱스 접근도 반응형
```

### React - 불변성 유지 필요

```javascript
const [form, setForm] = useState({
    name: '',
    email: '',
    address: {
        city: '',
        street: ''
    }
});

// 중첩 객체 업데이트가 복잡함
setForm({
    ...form,
    address: {
        ...form.address,
        city: '서울'
    }
});

// 배열 업데이트
const [items, setItems] = useState([1, 2, 3]);
setItems([...items, 4]);  // push 대신 스프레드
setItems(items.map((item, i) => i === 0 ? 100 : item));  // 인덱스 변경
```

### 발생할 수 있는 실수 (React)

```javascript
// 실수: 중첩 객체에서 스프레드 누락
setForm({
    ...form,
    address: {
        city: '서울'  // street이 사라짐!
    }
});

// 올바른 방법
setForm({
    ...form,
    address: {
        ...form.address,  // 기존 값 유지
        city: '서울'
    }
});
```

---

## 5. 양방향 바인딩

### Vue - v-model로 간단하게

```html
<template>
    <input v-model="name" />
    <input v-model="email" />
    <select v-model="country">
        <option value="kr">한국</option>
        <option value="us">미국</option>
    </select>
    <input type="checkbox" v-model="agreed" />
</template>

<script setup>
const name = ref('');
const email = ref('');
const country = ref('kr');
const agreed = ref(false);
</script>
```

### React - 매번 핸들러 작성

```jsx
function Form() {
    const [name, setName] = useState('');
    const [email, setEmail] = useState('');
    const [country, setCountry] = useState('kr');
    const [agreed, setAgreed] = useState(false);

    return (
        <>
            <input
                value={name}
                onChange={(e) => setName(e.target.value)}
            />
            <input
                value={email}
                onChange={(e) => setEmail(e.target.value)}
            />
            <select
                value={country}
                onChange={(e) => setCountry(e.target.value)}
            >
                <option value="kr">한국</option>
                <option value="us">미국</option>
            </select>
            <input
                type="checkbox"
                checked={agreed}
                onChange={(e) => setAgreed(e.target.checked)}
            />
        </>
    );
}
```

### 발생할 수 있는 실수 (React)

```jsx
// 실수 1: value만 있고 onChange 없음 → 입력 불가
<input value={name} />  // 읽기 전용이 됨

// 실수 2: checkbox에서 value와 checked 혼동
<input type="checkbox" value={agreed} />  // X
<input type="checkbox" checked={agreed} />  // O

// 실수 3: onChange 핸들러 오타
<input value={name} onChange={(e) => setName(e.target.vlaue)} />
//                                              ↑ 오타! (value → vlaue)
```

---

## 6. 조건부 렌더링

### Vue - 디렉티브로 명확하게

```html
<template>
    <!-- 조건부 렌더링 -->
    <div v-if="status === 'loading'">로딩 중...</div>
    <div v-else-if="status === 'error'">에러 발생</div>
    <div v-else>{{ data }}</div>

    <!-- 표시/숨김 (DOM 유지) -->
    <div v-show="isVisible">이 요소는 숨겨질 수 있음</div>
</template>
```

### React - JSX 내에서 JavaScript 표현식

```jsx
function Component() {
    return (
        <>
            {/* 삼항 연산자 중첩 - 가독성 저하 */}
            {status === 'loading'
                ? <div>로딩 중...</div>
                : status === 'error'
                    ? <div>에러 발생</div>
                    : <div>{data}</div>
            }

            {/* 또는 && 연산자 */}
            {isVisible && <div>이 요소는 숨겨질 수 있음</div>}
        </>
    );
}
```

### 발생할 수 있는 실수 (React)

```jsx
// 실수 1: && 연산자에서 0이 렌더링됨
const count = 0;
{count && <div>카운트: {count}</div>}  // "0"이 화면에 표시됨!

// 올바른 방법
{count > 0 && <div>카운트: {count}</div>}
{count ? <div>카운트: {count}</div> : null}

// 실수 2: 복잡한 조건에서 괄호 누락
{isLoggedIn && isAdmin && hasPermission && <AdminPanel />}  // 읽기 어려움
```

---

## 7. 리스트 렌더링

### Vue - v-for 디렉티브

```html
<template>
    <ul>
        <li v-for="(item, index) in items" :key="item.id">
            {{ index + 1 }}. {{ item.name }}
        </li>
    </ul>

    <!-- 객체 순회 -->
    <div v-for="(value, key) in object" :key="key">
        {{ key }}: {{ value }}
    </div>

    <!-- 범위 순회 -->
    <span v-for="n in 10" :key="n">{{ n }}</span>
</template>
```

### React - map 함수 사용

```jsx
function Component() {
    return (
        <ul>
            {items.map((item, index) => (
                <li key={item.id}>
                    {index + 1}. {item.name}
                </li>
            ))}
        </ul>

        {/* 객체 순회 */}
        {Object.entries(object).map(([key, value]) => (
            <div key={key}>
                {key}: {value}
            </div>
        ))}

        {/* 범위 순회 - 배열 생성 필요 */}
        {Array.from({ length: 10 }, (_, i) => i + 1).map(n => (
            <span key={n}>{n}</span>
        ))}
    );
}
```

### 발생할 수 있는 실수 (React)

```jsx
// 실수 1: key 누락 → 경고 + 성능 문제
{items.map(item => <li>{item.name}</li>)}  // key 없음!

// 실수 2: index를 key로 사용 → 리스트 변경 시 버그
{items.map((item, index) => <li key={index}>{item.name}</li>)}  // 위험!

// 실수 3: map 콜백에서 return 누락
{items.map(item => {
    <li key={item.id}>{item.name}</li>  // return 없음! 아무것도 안 나옴
})}

// 올바른 방법
{items.map(item => {
    return <li key={item.id}>{item.name}</li>;
})}
// 또는
{items.map(item => (
    <li key={item.id}>{item.name}</li>
))}
```

---

## 8. Props 기본값 처리

### Vue - 선언적으로 명확하게

```javascript
props: {
    title: {
        type: String,
        required: true
    },
    count: {
        type: Number,
        default: 0
    },
    items: {
        type: Array,
        default: () => []  // 팩토리 함수
    },
    config: {
        type: Object,
        default: () => ({ theme: 'light', size: 'medium' })
    },
    onSubmit: {
        type: Function,
        default: () => {}
    }
}
```

### React - 여러 방법이 혼재

```jsx
// 방법 1: defaultProps (클래스형, deprecated 예정)
Component.defaultProps = {
    count: 0,
    items: []
};

// 방법 2: 구조분해 기본값
function Component({
    title,  // required는 PropTypes로 별도 체크
    count = 0,
    items = [],
    config = { theme: 'light', size: 'medium' },
    onSubmit = () => {}
}) {
    // ...
}

// 방법 3: PropTypes (별도 라이브러리)
import PropTypes from 'prop-types';
Component.propTypes = {
    title: PropTypes.string.isRequired,
    count: PropTypes.number
};
```

### 발생할 수 있는 실수 (React)

```jsx
// 실수 1: 객체/배열 기본값을 리터럴로 선언 → 매 렌더마다 새 참조
function Component({ items = [] }) {
    useEffect(() => {
        console.log('items changed');
    }, [items]);  // 매번 새 배열이라 항상 실행됨!
}

// 실수 2: required 체크 누락
function Component({ data }) {
    return <div>{data.name}</div>;  // data가 undefined면 에러!
}

// 실수 3: PropTypes와 TypeScript 혼용으로 인한 혼란
```

---

## 9. 비동기 상태 업데이트

### Vue - 직관적인 동작

```javascript
const count = ref(0);

async function increment() {
    count.value++;
    count.value++;
    count.value++;

    console.log(count.value);  // 3 (예상대로)
}
```

### React - 배치 처리로 인한 주의 필요

```javascript
const [count, setCount] = useState(0);

async function increment() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);

    console.log(count);  // 여전히 0! (비동기 업데이트)
    // 결과적으로 count는 1이 됨 (3이 아님!)
}

// 올바른 방법: 함수형 업데이트
async function incrementCorrect() {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    // 이제 count는 3이 됨
}
```

### 발생할 수 있는 실수 (React)

```javascript
// 실수 1: 연속 업데이트에서 이전 값 사용
function handleClick() {
    setCount(count + 1);  // count = 0 기준
    setCount(count + 1);  // count = 0 기준 (같은 값!)
    // 결과: 1 (예상: 2)
}

// 실수 2: 비동기 콜백에서 stale closure
function handleClick() {
    setTimeout(() => {
        setCount(count + 1);  // 클릭 시점의 count 값 사용
    }, 1000);
}
// 빠르게 3번 클릭하면? 결과는 1 (예상: 3)

// 올바른 방법
function handleClick() {
    setTimeout(() => {
        setCount(prev => prev + 1);  // 최신 값 사용
    }, 1000);
}
```

---

## 결론

### Vue의 장점 요약

| 항목 | Vue | React |
|------|-----|-------|
| 의존성 관리 | 자동 추적 | 수동 명시 |
| 반응성 | `.value`로 명확 | 규칙 숙지 필요 |
| 라이프사이클 | 분리된 훅 | useEffect 통합 |
| 상태 변경 | 직접 변경 가능 | 불변성 유지 |
| 폼 바인딩 | v-model | 핸들러 작성 |
| 조건부 렌더링 | v-if/v-else | 삼항 연산자 |
| 리스트 렌더링 | v-for | map 함수 |
| Props 기본값 | 선언적 | 여러 방법 혼재 |
| 비동기 업데이트 | 직관적 | 배치 처리 주의 |

### 결론

- **Vue**: 프레임워크가 가이드레일을 제공 → **실수하기 어려움**
- **React**: 자유도가 높음 → **유연하지만 책임은 개발자에게**

Vue는 "올바른 방법"을 강제하는 경향이 있고,
React는 "여러 방법 중 선택"하게 하는 경향이 있습니다.

팀 프로젝트나 주니어 개발자가 많은 환경에서는 Vue의 명확한 규칙이 장점이 될 수 있습니다.

---
