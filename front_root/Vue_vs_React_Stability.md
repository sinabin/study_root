---
sticker: emoji//2696-fe0f
---
# Vue vs React 안정성 비교

> "Vue랑 React 중에 뭐가 더 안정적이야?" 라는 질문에서 시작해보자.

---

## Q: Vue랑 React 중에 뭐가 더 안정적이야?

**결론부터**: 둘 다 프로덕션 레벨에서 충분히 안정적이야. 하지만 "안정성"의 기준에 따라 답이 달라져.

| 기준 | Vue | React |
|------|-----|-------|
| **API 변경 빈도** | Vue 3로 대규모 변경 후 안정 | 지속적 점진 변경 (Hooks 등) |
| **생태계** | 공식 라이브러리 중심 | 서드파티 중심, 선택지 많음 |
| **실수 방지** | 프레임워크가 가이드 제공 | 자유도 높음, 책임은 개발자 |

---

## Q: "API 변경 빈도"가 안정성이랑 무슨 상관이야?

한번 배운 코드 패턴이 얼마나 오래 유지되느냐의 문제야.

### Vue의 경우

```javascript
// Vue 2 (Options API)
export default {
  data() {
    return { count: 0 }
  },
  methods: {
    increment() { this.count++ }
  }
}

// Vue 3 (Composition API) - 완전히 다른 패러다임!
import { ref } from 'vue'
const count = ref(0)
const increment = () => count.value++
```

Vue 2 → Vue 3는 **큰 변화**였어. 하지만 Vue 3 내에서는 안정적.

### React의 경우

```javascript
// Class Component (옛날 방식)
class Counter extends React.Component {
  state = { count: 0 }
  increment = () => this.setState({ count: this.state.count + 1 })
}

// Hooks (현재 권장)
function Counter() {
  const [count, setCount] = useState(0)
  const increment = () => setCount(c => c + 1)
}
```

React는 Class → Hooks로 **점진적으로** 변경됐어. 두 방식 모두 여전히 지원되고.

---

## Q: 그럼 "실수하기 어려운" 프레임워크는 뭐야?

Vue가 "올바른 방법"을 더 강제하는 편이야.

### 의존성 자동 추적 (Vue vs React)

```javascript
// Vue - 의존성을 명시할 필요 없음
const total = computed(() => {
    return price.value * quantity.value  // Vue가 자동 추적
})

// React - 의존성 배열 직접 관리
const total = useMemo(() => {
    return price * quantity
}, [price, quantity])  // 빼먹으면 버그!
```

### 상태 변경 방식

```javascript
// Vue - 직접 수정 가능
items.value.push(4)        // OK
user.value.name = '홍길동'  // OK

// React - 불변성 유지 필수
items.push(4)                 // 화면 안 바뀜!
setItems([...items, 4])       // 이렇게 해야 함
setUser({ ...user, name: '홍길동' })  // 새 객체 생성 필수
```

---

## Q: React에서 흔히 하는 실수가 뭐야?

### 1. 의존성 배열 실수

```javascript
// taxRate를 빼먹으면?
const total = useMemo(() => {
    return price * quantity * (1 + taxRate)  // taxRate 사용하는데...
}, [price, quantity])  // taxRate 없음! → 세율 바뀌어도 total 안 바뀜
```

### 2. 직접 수정 실수

```javascript
const [items, setItems] = useState([1, 2, 3])
items.push(4)  // 화면 안 바뀜, 에러도 안 남!

const [form, setForm] = useState({ name: '', email: '' })
form.name = '홍길동'  // 화면 안 바뀜
```

### 3. useEffect 의존성 배열 실수

```javascript
// 빈 배열 vs 배열 없음 혼동
useEffect(() => { ... }, [])   // 마운트 시 1번만
useEffect(() => { ... })       // 매 렌더마다 실행! (위험)
```

---

## Q: Vue에서 흔히 하는 실수는?

### 1. ref와 reactive 혼동

```javascript
const state = reactive({
  count: ref(0)  // 이중 래핑 - 불필요
})

// ref 자체를 교체
let count = ref(0)
count = ref(1)      // 반응성 끊김! ref 객체 자체를 교체해버림
count.value = 1     // 이렇게 해야 함
```

### 2. 구조분해로 반응성 손실

```javascript
const state = reactive({ x: 1, y: 2 })
const { x, y } = state  // x, y는 이제 그냥 숫자! (반응성 없음)
const { x, y } = toRefs(state)  // 이렇게 해야 반응성 유지
```

---

## Q: 양방향 바인딩은 뭐가 달라?

### Vue - v-model로 간단

```html
<input v-model="name" />
<input v-model="email" />
<select v-model="country">
    <option value="kr">한국</option>
</select>
```

### React - 매번 핸들러 작성

```jsx
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
</select>
```

React에서 흔한 실수:
```jsx
<input value={name} />  // onChange 없으면 입력 불가!
```

---

## Q: 리스트 렌더링은?

### Vue - v-for 디렉티브

```html
<ul>
    <li v-for="(item, index) in items" :key="item.id">
        {{ index + 1 }}. {{ item.name }}
    </li>
</ul>
```

### React - map 함수

```jsx
<ul>
    {items.map((item, index) => (
        <li key={item.id}>
            {index + 1}. {item.name}
        </li>
    ))}
</ul>
```

React에서 흔한 실수:
```jsx
// return 누락 - 아무것도 안 나옴!
{items.map(item => {
    <li key={item.id}>{item.name}</li>
})}

// 올바른 방법
{items.map(item => (
    <li key={item.id}>{item.name}</li>
))}
```

---

## Q: 비동기 상태 업데이트는?

### Vue - 직관적

```javascript
const count = ref(0)

function increment() {
    count.value++
    count.value++
    count.value++
    console.log(count.value)  // 3 (예상대로)
}
```

### React - 배치 처리 주의

```javascript
const [count, setCount] = useState(0)

function increment() {
    setCount(count + 1)
    setCount(count + 1)
    setCount(count + 1)
    console.log(count)  // 여전히 0! (비동기 업데이트)
    // 결과적으로 count는 1이 됨 (3이 아님!)
}

// 올바른 방법: 함수형 업데이트
function incrementCorrect() {
    setCount(prev => prev + 1)
    setCount(prev => prev + 1)
    setCount(prev => prev + 1)
    // 이제 count는 3이 됨
}
```

---

## 결론: 언제 뭘 써?

| | Vue | React |
|---|---|---|
| **장점** | 실수하기 어려움, 직관적 | 유연함, 생태계 넓음 |
| **단점** | 생태계 상대적으로 작음 | 규칙 숙지 필요 |
| **추천 상황** | 주니어 많은 팀, 빠른 개발 | 대규모 앱, 다양한 요구사항 |

### Vue가 유리한 경우
- 팀 규모가 작을 때
- 빠른 프로토타이핑
- 실수 방지가 중요할 때

### React가 유리한 경우
- 대규모 앱
- 다양한 라이브러리 필요
- React Native로 모바일 확장

---

## 이어서 읽기

- [[vue-ref-vs-react-usestate]] - ref와 useState의 구체적 차이
- [[React_Vue_Reactivity_Misconceptions]] - 반응성에 대한 오해들
- [[React_Vue_SideEffects_Guide]] - 사이드 이펙트 처리 방법
