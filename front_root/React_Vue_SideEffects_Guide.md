# React useEffect & Vue watch 완벽 가이드

> 언제, 왜 사용하는가?

---

## 목차

1. [핵심 개념](#1-핵심-개념)
2. [API 호출](#2-api-호출)
3. [DOM 조작 및 외부 라이브러리](#3-dom-조작-및-외부-라이브러리)
4. [구독과 해제](#4-구독과-해제)
5. [로컬 스토리지 동기화](#5-로컬-스토리지-동기화)
6. [문서 제목 변경](#6-문서-제목-변경)
7. [디바운스 검색](#7-디바운스-검색)
8. [폼 유효성 검사](#8-폼-유효성-검사)
9. [URL 파라미터 동기화](#9-url-파라미터-동기화)
10. [타이머 관리](#10-타이머-관리)
11. [정리 요약](#11-정리-요약)

---

## 1. 핵심 개념

### 부수효과(Side Effect)란?

컴포넌트의 **렌더링과 직접 관련 없는 작업**을 말합니다.

```
렌더링 = 화면에 UI를 그리는 것 (순수한 작업)
부수효과 = 그 외 모든 것 (외부와 상호작용)
```

### 왜 필요한가?

```
┌─────────────────────────────────────────────────────────┐
│  상태 변경 → 화면 업데이트 → 그 다음에 뭔가 하고 싶다!     │
│                               ↓                         │
│                         useEffect / watch               │
└─────────────────────────────────────────────────────────┘
```

### 대표적인 사용 사례

| 사례 | 설명 |
|------|------|
| API 호출 | 값이 바뀌면 서버에서 데이터 가져오기 |
| DOM 조작 | 렌더링 후 외부 라이브러리 초기화 |
| 구독/해제 | 이벤트 리스너, 웹소켓 연결 |
| 로컬 스토리지 | 상태 변경 시 저장 |
| 문서 제목 | `document.title` 업데이트 |
| 타이머 | `setInterval`, `setTimeout` 관리 |

---

## 2. API 호출

> 특정 값이 바뀌면 서버에서 데이터를 다시 가져온다

### React

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(false);

    useEffect(() => {
        // userId가 바뀔 때마다 실행
        setLoading(true);

        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => setUser(data))
            .finally(() => setLoading(false));

    }, [userId]);  // 의존성: userId

    if (loading) return <div>로딩 중...</div>;
    return <div>{user?.name}</div>;
}
```

### Vue

```javascript
setup(props) {
    const user = ref(null);
    const loading = ref(false);

    // props.userId가 바뀔 때마다 실행
    watch(() => props.userId, async (newUserId) => {
        loading.value = true;

        try {
            const response = await fetch(`/api/users/${newUserId}`);
            user.value = await response.json();
        } finally {
            loading.value = false;
        }
    }, { immediate: true });  // 최초 마운트 시에도 실행

    return { user, loading };
}
```

### 언제 사용?

```
✅ 사용자가 다른 프로필을 클릭했을 때
✅ 검색 조건이 변경되었을 때
✅ 페이지 번호가 바뀌었을 때
✅ 필터가 변경되었을 때
```

---

## 3. DOM 조작 및 외부 라이브러리

> 렌더링이 완료된 후 DOM에 직접 접근하거나 외부 라이브러리를 초기화한다

### React

```javascript
function Chart({ data }) {
    const chartRef = useRef(null);
    const chartInstance = useRef(null);

    // 마운트 시 차트 초기화
    useEffect(() => {
        // DOM이 준비된 후 실행됨
        chartInstance.current = new ChartJS(chartRef.current, {
            type: 'bar',
            data: data
        });

        // 언마운트 시 정리
        return () => {
            chartInstance.current.destroy();
        };
    }, []);  // 빈 배열: 마운트 시 1번만

    // 데이터 변경 시 차트 업데이트
    useEffect(() => {
        if (chartInstance.current) {
            chartInstance.current.data = data;
            chartInstance.current.update();
        }
    }, [data]);

    return <canvas ref={chartRef} />;
}
```

### Vue

```javascript
setup(props) {
    const chartRef = ref(null);
    let chartInstance = null;

    // 마운트 시 차트 초기화
    onMounted(() => {
        chartInstance = new ChartJS(chartRef.value, {
            type: 'bar',
            data: props.data
        });
    });

    // 언마운트 시 정리
    onBeforeUnmount(() => {
        chartInstance?.destroy();
    });

    // 데이터 변경 시 차트 업데이트
    watch(() => props.data, (newData) => {
        if (chartInstance) {
            chartInstance.data = newData;
            chartInstance.update();
        }
    });

    return { chartRef };
}
```

### 언제 사용?

```
✅ Chart.js, D3.js 등 차트 라이브러리 초기화
✅ 지도 라이브러리 (Kakao Map, Google Maps) 초기화
✅ 에디터 (CKEditor, TinyMCE) 초기화
✅ 애니메이션 라이브러리 연동
```

---

## 4. 구독과 해제

> 이벤트 리스너, 웹소켓 등을 연결하고 해제한다

### React

```javascript
function WindowSize() {
    const [size, setSize] = useState({
        width: window.innerWidth,
        height: window.innerHeight
    });

    useEffect(() => {
        // 이벤트 핸들러 정의
        function handleResize() {
            setSize({
                width: window.innerWidth,
                height: window.innerHeight
            });
        }

        // 구독 (이벤트 리스너 등록)
        window.addEventListener('resize', handleResize);

        // 해제 (클린업 함수)
        return () => {
            window.removeEventListener('resize', handleResize);
        };
    }, []);  // 마운트/언마운트 시에만

    return <div>화면 크기: {size.width} x {size.height}</div>;
}
```

### Vue

```javascript
setup() {
    const size = reactive({
        width: window.innerWidth,
        height: window.innerHeight
    });

    function handleResize() {
        size.width = window.innerWidth;
        size.height = window.innerHeight;
    }

    // 마운트 시 구독
    onMounted(() => {
        window.addEventListener('resize', handleResize);
    });

    // 언마운트 시 해제
    onBeforeUnmount(() => {
        window.removeEventListener('resize', handleResize);
    });

    return { size };
}
```

### 웹소켓 예제

#### React

```javascript
function ChatRoom({ roomId }) {
    const [messages, setMessages] = useState([]);

    useEffect(() => {
        // roomId가 바뀔 때마다 새 연결
        const socket = new WebSocket(`wss://chat.example.com/${roomId}`);

        socket.onmessage = (event) => {
            const message = JSON.parse(event.data);
            setMessages(prev => [...prev, message]);
        };

        // 클린업: 이전 연결 종료
        return () => {
            socket.close();
        };
    }, [roomId]);

    return (
        <ul>
            {messages.map(msg => <li key={msg.id}>{msg.text}</li>)}
        </ul>
    );
}
```

#### Vue

```javascript
setup(props) {
    const messages = ref([]);
    let socket = null;

    // roomId가 바뀔 때마다 새 연결
    watch(() => props.roomId, (newRoomId, oldRoomId) => {
        // 이전 연결 종료
        if (socket) {
            socket.close();
        }

        // 새 연결
        socket = new WebSocket(`wss://chat.example.com/${newRoomId}`);

        socket.onmessage = (event) => {
            const message = JSON.parse(event.data);
            messages.value.push(message);
        };
    }, { immediate: true });

    // 언마운트 시 연결 종료
    onBeforeUnmount(() => {
        socket?.close();
    });

    return { messages };
}
```

### 언제 사용?

```
✅ 윈도우 이벤트 (resize, scroll, keydown)
✅ 웹소켓 연결
✅ Server-Sent Events (SSE)
✅ 브로드캐스트 채널
✅ 인터섹션 옵저버
```

---

## 5. 로컬 스토리지 동기화

> 상태가 바뀔 때마다 로컬 스토리지에 저장한다

### React

```javascript
function ShoppingCart() {
    // 초기값: 로컬 스토리지에서 불러오기
    const [cart, setCart] = useState(() => {
        const saved = localStorage.getItem('cart');
        return saved ? JSON.parse(saved) : [];
    });

    // cart가 바뀔 때마다 저장
    useEffect(() => {
        localStorage.setItem('cart', JSON.stringify(cart));
    }, [cart]);

    function addItem(item) {
        setCart([...cart, item]);
    }

    function removeItem(id) {
        setCart(cart.filter(item => item.id !== id));
    }

    return (
        <div>
            <p>장바구니: {cart.length}개</p>
            {cart.map(item => (
                <div key={item.id}>
                    {item.name}
                    <button onClick={() => removeItem(item.id)}>삭제</button>
                </div>
            ))}
        </div>
    );
}
```

### Vue

```javascript
setup() {
    // 초기값: 로컬 스토리지에서 불러오기
    const saved = localStorage.getItem('cart');
    const cart = ref(saved ? JSON.parse(saved) : []);

    // cart가 바뀔 때마다 저장
    watch(cart, (newCart) => {
        localStorage.setItem('cart', JSON.stringify(newCart));
    }, { deep: true });  // 배열 내부 변경도 감지

    function addItem(item) {
        cart.value.push(item);
    }

    function removeItem(id) {
        const index = cart.value.findIndex(item => item.id === id);
        if (index > -1) {
            cart.value.splice(index, 1);
        }
    }

    return { cart, addItem, removeItem };
}
```

### 언제 사용?

```
✅ 장바구니 데이터 유지
✅ 사용자 설정 (다크모드, 언어)
✅ 폼 임시 저장 (작성 중 새로고침 대비)
✅ 최근 검색어 저장
```

---

## 6. 문서 제목 변경

> 페이지 내용에 따라 브라우저 탭 제목을 바꾼다

### React

```javascript
function ProductDetail({ product }) {
    // product가 바뀔 때마다 제목 변경
    useEffect(() => {
        const originalTitle = document.title;

        document.title = `${product.name} - 쇼핑몰`;

        // 언마운트 시 원래 제목으로 복원
        return () => {
            document.title = originalTitle;
        };
    }, [product.name]);

    return <div>{product.name}</div>;
}
```

### Vue

```javascript
setup(props) {
    // product.name이 바뀔 때마다 제목 변경
    watch(() => props.product.name, (newName) => {
        document.title = `${newName} - 쇼핑몰`;
    }, { immediate: true });

    // 언마운트 시 원래 제목으로 복원
    const originalTitle = document.title;
    onBeforeUnmount(() => {
        document.title = originalTitle;
    });

    return {};
}
```

### 언제 사용?

```
✅ 상품 상세 페이지에서 상품명 표시
✅ 채팅방에서 읽지 않은 메시지 수 표시: "(3) 채팅"
✅ 현재 재생 중인 음악 제목 표시
```

---

## 7. 디바운스 검색

> 사용자가 타이핑을 멈춘 후 일정 시간 뒤에 검색한다

### React

```javascript
function SearchBox() {
    const [keyword, setKeyword] = useState('');
    const [results, setResults] = useState([]);

    useEffect(() => {
        // 빈 검색어면 무시
        if (!keyword.trim()) {
            setResults([]);
            return;
        }

        // 300ms 후에 검색 실행
        const timer = setTimeout(() => {
            fetch(`/api/search?q=${keyword}`)
                .then(res => res.json())
                .then(data => setResults(data));
        }, 300);

        // 키워드가 바뀌면 이전 타이머 취소
        return () => clearTimeout(timer);

    }, [keyword]);

    return (
        <div>
            <input
                value={keyword}
                onChange={(e) => setKeyword(e.target.value)}
                placeholder="검색어 입력..."
            />
            <ul>
                {results.map(item => (
                    <li key={item.id}>{item.title}</li>
                ))}
            </ul>
        </div>
    );
}
```

### Vue

```javascript
setup() {
    const keyword = ref('');
    const results = ref([]);
    let debounceTimer = null;

    watch(keyword, (newKeyword) => {
        // 이전 타이머 취소
        if (debounceTimer) {
            clearTimeout(debounceTimer);
        }

        // 빈 검색어면 무시
        if (!newKeyword.trim()) {
            results.value = [];
            return;
        }

        // 300ms 후에 검색 실행
        debounceTimer = setTimeout(async () => {
            const response = await fetch(`/api/search?q=${newKeyword}`);
            results.value = await response.json();
        }, 300);
    });

    // 언마운트 시 타이머 정리
    onBeforeUnmount(() => {
        if (debounceTimer) {
            clearTimeout(debounceTimer);
        }
    });

    return { keyword, results };
}
```

### 언제 사용?

```
✅ 실시간 검색 자동완성
✅ 필터 조건 변경 시 목록 조회
✅ 폼 입력값 자동 저장
```

---

## 8. 폼 유효성 검사

> 입력값이 바뀔 때마다 유효성을 검사한다

### React

```javascript
function RegistrationForm() {
    const [email, setEmail] = useState('');
    const [password, setPassword] = useState('');
    const [errors, setErrors] = useState({});

    // 이메일 유효성 검사
    useEffect(() => {
        if (!email) {
            setErrors(prev => ({ ...prev, email: null }));
            return;
        }

        const isValid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
        setErrors(prev => ({
            ...prev,
            email: isValid ? null : '올바른 이메일 형식이 아닙니다'
        }));
    }, [email]);

    // 비밀번호 유효성 검사
    useEffect(() => {
        if (!password) {
            setErrors(prev => ({ ...prev, password: null }));
            return;
        }

        const isValid = password.length >= 8;
        setErrors(prev => ({
            ...prev,
            password: isValid ? null : '비밀번호는 8자 이상이어야 합니다'
        }));
    }, [password]);

    return (
        <form>
            <div>
                <input
                    type="email"
                    value={email}
                    onChange={(e) => setEmail(e.target.value)}
                    placeholder="이메일"
                />
                {errors.email && <span className="error">{errors.email}</span>}
            </div>
            <div>
                <input
                    type="password"
                    value={password}
                    onChange={(e) => setPassword(e.target.value)}
                    placeholder="비밀번호"
                />
                {errors.password && <span className="error">{errors.password}</span>}
            </div>
        </form>
    );
}
```

### Vue

```javascript
setup() {
    const email = ref('');
    const password = ref('');
    const errors = reactive({ email: null, password: null });

    // 이메일 유효성 검사
    watch(email, (newEmail) => {
        if (!newEmail) {
            errors.email = null;
            return;
        }

        const isValid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(newEmail);
        errors.email = isValid ? null : '올바른 이메일 형식이 아닙니다';
    });

    // 비밀번호 유효성 검사
    watch(password, (newPassword) => {
        if (!newPassword) {
            errors.password = null;
            return;
        }

        const isValid = newPassword.length >= 8;
        errors.password = isValid ? null : '비밀번호는 8자 이상이어야 합니다';
    });

    return { email, password, errors };
}
```

### 언제 사용?

```
✅ 실시간 폼 유효성 검사
✅ 중복 아이디 체크 (API 호출과 결합)
✅ 비밀번호 강도 표시
```

---

## 9. URL 파라미터 동기화

> 상태와 URL 쿼리 파라미터를 동기화한다

### React

```javascript
function ProductList() {
    const [searchParams, setSearchParams] = useSearchParams();
    const [filters, setFilters] = useState({
        category: searchParams.get('category') || 'all',
        sort: searchParams.get('sort') || 'latest'
    });

    // filters가 바뀌면 URL 업데이트
    useEffect(() => {
        setSearchParams({
            category: filters.category,
            sort: filters.sort
        });
    }, [filters, setSearchParams]);

    return (
        <div>
            <select
                value={filters.category}
                onChange={(e) => setFilters({...filters, category: e.target.value})}
            >
                <option value="all">전체</option>
                <option value="electronics">전자기기</option>
                <option value="clothing">의류</option>
            </select>
            <select
                value={filters.sort}
                onChange={(e) => setFilters({...filters, sort: e.target.value})}
            >
                <option value="latest">최신순</option>
                <option value="price">가격순</option>
            </select>
        </div>
    );
}
```

### Vue

```javascript
setup() {
    const route = useRoute();
    const router = useRouter();

    const filters = reactive({
        category: route.query.category || 'all',
        sort: route.query.sort || 'latest'
    });

    // filters가 바뀌면 URL 업데이트
    watch(filters, (newFilters) => {
        router.replace({
            query: {
                category: newFilters.category,
                sort: newFilters.sort
            }
        });
    }, { deep: true });

    return { filters };
}
```

### 언제 사용?

```
✅ 검색 필터를 URL에 유지 (새로고침해도 유지)
✅ 페이지네이션 상태 URL 반영
✅ 공유 가능한 링크 생성
```

---

## 10. 타이머 관리

> 컴포넌트에서 반복 작업을 수행하고 정리한다

### React

```javascript
function AutoRefreshData() {
    const [data, setData] = useState(null);
    const [lastUpdated, setLastUpdated] = useState(null);

    useEffect(() => {
        // 즉시 1번 실행
        fetchData();

        // 30초마다 반복
        const interval = setInterval(() => {
            fetchData();
        }, 30000);

        // 언마운트 시 정리 (메모리 누수 방지!)
        return () => {
            clearInterval(interval);
        };

        async function fetchData() {
            const response = await fetch('/api/dashboard');
            const result = await response.json();
            setData(result);
            setLastUpdated(new Date());
        }
    }, []);

    return (
        <div>
            <p>마지막 업데이트: {lastUpdated?.toLocaleTimeString()}</p>
            <pre>{JSON.stringify(data, null, 2)}</pre>
        </div>
    );
}
```

### Vue

```javascript
setup() {
    const data = ref(null);
    const lastUpdated = ref(null);
    let interval = null;

    async function fetchData() {
        const response = await fetch('/api/dashboard');
        data.value = await response.json();
        lastUpdated.value = new Date();
    }

    onMounted(() => {
        // 즉시 1번 실행
        fetchData();

        // 30초마다 반복
        interval = setInterval(fetchData, 30000);
    });

    // 언마운트 시 정리 (메모리 누수 방지!)
    onBeforeUnmount(() => {
        if (interval) {
            clearInterval(interval);
        }
    });

    return { data, lastUpdated };
}
```

### 언제 사용?

```
✅ 대시보드 자동 새로고침
✅ 세션 타임아웃 체크
✅ 실시간 시계 표시
✅ 알림 폴링
```

---

## 11. 정리 요약

### useEffect / watch를 쓰는 이유

```
┌─────────────────────────────────────────────────────────┐
│  렌더링 후에 "외부 세계"와 상호작용하기 위해 사용한다      │
└─────────────────────────────────────────────────────────┘
```

### 사용 사례 한눈에 보기

| 사용 사례 | 하는 일 | 정리(cleanup) 필요? |
|----------|--------|-------------------|
| API 호출 | 서버 데이터 가져오기 | △ (진행 중인 요청 취소) |
| DOM 조작 | 외부 라이브러리 초기화 | O (인스턴스 정리) |
| 구독/해제 | 이벤트 리스너, 웹소켓 | **O (필수!)** |
| 로컬 스토리지 | 상태 저장 | X |
| 문서 제목 | 탭 제목 변경 | △ (원래 제목 복원) |
| 디바운스 | 지연 실행 | O (타이머 취소) |
| 폼 유효성 | 입력값 검증 | X |
| URL 동기화 | 쿼리 파라미터 반영 | X |
| 타이머 | 주기적 실행 | **O (필수!)** |

### 클린업(정리)이 필요한 경우

```javascript
// React
useEffect(() => {
    const subscription = subscribe();

    return () => {
        subscription.unsubscribe();  // 정리!
    };
}, []);

// Vue
onMounted(() => {
    const subscription = subscribe();

    onBeforeUnmount(() => {
        subscription.unsubscribe();  // 정리!
    });
});
```

### 클린업을 안 하면?

```
❌ 메모리 누수
❌ 이벤트 중복 등록
❌ 타이머 계속 실행
❌ 언마운트된 컴포넌트에 상태 업데이트 시도 → 에러
```

### 최종 정리

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   useEffect / watch = "렌더링 후 외부와 상호작용하는 창구"     │
│                                                              │
│   ┌─────────┐      ┌─────────┐      ┌─────────────────┐     │
│   │  상태   │  →   │ 렌더링  │  →   │ useEffect/watch │     │
│   │  변경   │      │         │      │ (부수효과 실행)  │     │
│   └─────────┘      └─────────┘      └─────────────────┘     │
│       원인            결과              결과                  │
│                                                              │
│   - API 호출                                                 │
│   - DOM 조작                                                 │
│   - 이벤트 구독                                              │
│   - 외부 저장소 동기화                                        │
│   - 타이머 관리                                              │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

*작성일: 2025-12-05*
