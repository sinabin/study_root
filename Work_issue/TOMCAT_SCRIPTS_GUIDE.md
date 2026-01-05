---
sticker: emoji//1f606
---
# Tomcat 스크립트 파일 역할 및 실행 흐름 가이드

## 📋 목차
1. [전체 실행 흐름](#전체-실행-흐름)
2. [각 파일별 역할](#각-파일별-역할)
3. [파일 간 관계도](#파일-간-관계도)
4. [수정 가능 여부](#수정-가능-여부)
5. [실전 예제](#실전-예제)

---

## 🔄 전체 실행 흐름

Tomcat을 시작할 때 다음 순서로 스크립트가 실행됩니다:

```
1. startup.sh 실행
   ↓
2. catalina.sh 호출
   ↓
3. setclasspath.sh 로드 (catalina.sh가 호출)
   ↓
4. setenv.sh 로드 (catalina.sh가 자동 탐색, 있으면 로드)
   ↓
5. JVM 실행 (모든 환경 변수 적용)
```

---

## 📁 각 파일별 역할

### 1️⃣ `startup.sh` - Tomcat 시작 진입점

#### **역할**
- Tomcat 시작 명령의 **진입점** (사용자가 실행하는 파일)
- `catalina.sh`를 찾아서 실행

#### **주요 동작**
```bash
# 1. JAVA_HOME 설정 (우리 프로젝트에서 직접 추가한 부분)
export JAVA_HOME=/home/tomcat/jdk-21.0.2
export PATH=$JAVA_HOME/bin:$PATH

# 2. catalina.sh 실행
exec "$PRGDIR"/"$EXECUTABLE" start "$@"
# → exec /home/tomcat/tomcat/bin/catalina.sh start
```

#### **왜 존재하는가?**
- 사용자에게 **간단한 인터페이스** 제공 (`./startup.sh`만 입력)
- 복잡한 실행 로직은 `catalina.sh`에 위임

#### **수정 여부**
- ⚠️ **가급적 수정 지양** (Tomcat 업그레이드 시 덮어씌워질 수 있음)
- 우리 프로젝트는 JAVA_HOME 설정을 위해 예외적으로 수정함

---

### 2️⃣ `catalina.sh` - Tomcat 실행 핵심 로직

#### **역할**
- Tomcat의 **실제 실행 로직**을 담당
- JVM 옵션 설정, 클래스패스 구성, 프로세스 시작

#### **주요 동작**
```bash
# 1. setclasspath.sh 로드 (Java 환경 검증)
. "$CATALINA_HOME"/bin/setclasspath.sh

# 2. setenv.sh 로드 (사용자 정의 설정, 있으면)
if [ -r "$CATALINA_BASE"/bin/setenv.sh ]; then
  . "$CATALINA_BASE"/bin/setenv.sh
fi

# 3. 클래스패스 구성
CLASSPATH="$CATALINA_HOME/bin/bootstrap.jar:$CATALINA_HOME/bin/tomcat-juli.jar"

# 4. JVM 실행
eval exec "$_RUNJAVA" $JAVA_OPTS $CATALINA_OPTS \
  -classpath "$CLASSPATH" \
  -Dcatalina.base="$CATALINA_BASE" \
  -Dcatalina.home="$CATALINA_HOME" \
  org.apache.catalina.startup.Bootstrap start
```

#### **왜 존재하는가?**
- Tomcat의 **핵심 실행 엔진**
- start, stop, run, debug 등 다양한 모드 지원

#### **수정 여부**
- ❌ **절대 수정 금지** (Tomcat 원본 파일)
- 사용자 정의는 `setenv.sh`에서만!

---

### 3️⃣ `setclasspath.sh` - Java 환경 검증

#### **역할**
- **JAVA_HOME/JRE_HOME 검증**
- Java 실행 파일 경로 설정 (`_RUNJAVA` 변수)

#### **주요 동작**
```bash
# 1. JAVA_HOME 또는 JRE_HOME 확인
if [ -z "$JAVA_HOME" ] && [ -z "$JRE_HOME" ]; then
  # 자동으로 Java 경로 탐색 (MacOS, Linux)
  JAVA_PATH=`which java 2>/dev/null`
  ...
fi

# 2. JAVA_HOME이 유효한지 검증
if [ ! -x "$JAVA_HOME"/bin/java ]; then
  echo "The JAVA_HOME environment variable is not defined correctly"
  exit 1
fi

# 3. _RUNJAVA 변수 설정
_RUNJAVA="$JRE_HOME"/bin/java
```

#### **왜 존재하는가?**
- **Java 설치 여부 확인** (없으면 친절한 에러 메시지)
- 다양한 OS 환경에서 Java 경로 자동 탐색

#### **수정 여부**
- ❌ **절대 수정 금지** (Tomcat 원본 파일)

---

### 4️⃣ `setenv.sh` - 사용자 정의 환경 변수 (새로 생성)

#### **역할**
- **사용자가 추가하는 JVM 옵션 및 환경 변수**
- `catalina.sh`가 자동으로 탐색하여 로드

#### **주요 내용** (우리 프로젝트)
```bash
#!/bin/sh
# Tomcat 환경 변수 설정 파일
# catalina.sh가 자동으로 로드하여 JAVA_OPTS에 추가함

# nFilter 네이티브 라이브러리 경로 설정
export JAVA_OPTS="$JAVA_OPTS -Djava.library.path=/home/tomcat/tomcat/webapps/ROOT/WEB-INF/native"

# LD_LIBRARY_PATH 설정 (네이티브 라이브러리 로딩용)
export LD_LIBRARY_PATH="/home/tomcat/tomcat/webapps/ROOT/WEB-INF/native:$LD_LIBRARY_PATH"
```

#### **왜 존재하는가?**
- **Tomcat 원본 파일을 수정하지 않고** 사용자 설정 추가
- 업그레이드 시에도 설정 유지

#### **수정 여부**
- ✅ **자유롭게 수정 가능** (사용자가 생성한 파일)
- Tomcat 업그레이드 영향 없음

---

## 🗺️ 파일 간 관계도

```
┌─────────────────────────────────────────────────────────────┐
│  사용자 실행: ./startup.sh                                   │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  startup.sh                                                  │
│  - JAVA_HOME 설정 (우리 프로젝트만)                          │
│  - catalina.sh 실행                                          │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  catalina.sh (Tomcat 핵심 실행 스크립트)                     │
│                                                              │
│  ┌─────────────────────────────────────┐                    │
│  │ 1. setclasspath.sh 로드              │                   │
│  │    → JAVA_HOME 검증                 │                    │
│  │    → _RUNJAVA 설정                  │                    │
│  └─────────────────────────────────────┘                    │
│                     ↓                                        │
│  ┌─────────────────────────────────────┐                    │
│  │ 2. setenv.sh 로드 (있으면)          │                    │
│  │    → JAVA_OPTS 추가                 │                    │
│  │    → LD_LIBRARY_PATH 설정           │                    │
│  └─────────────────────────────────────┘                    │
│                     ↓                                        │
│  ┌─────────────────────────────────────┐                    │
│  │ 3. JVM 실행                         │                    │
│  │    java $JAVA_OPTS ...              │                    │
│  └─────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔒 수정 가능 여부

| 파일명 | 제공자 | 수정 가능 여부 | 업그레이드 시 |
|--------|--------|----------------|---------------|
| `startup.sh` | Tomcat | ⚠️ 비추천 (예외적으로 수정함) | 덮어씌워짐 |
| `catalina.sh` | Tomcat | ❌ 절대 금지 | 덮어씌워짐 |
| `setclasspath.sh` | Tomcat | ❌ 절대 금지 | 덮어씌워짐 |
| `setenv.sh` | **사용자** | ✅ 자유롭게 수정 | **보존됨** |

**결론:** 사용자 정의 설정은 **무조건 `setenv.sh`에만** 작성해야 합니다!

---

## 💡 실전 예제

### 예제 1: JVM 힙 메모리 설정

**❌ 잘못된 방법** (startup.sh 수정)
```bash
# startup.sh에 직접 추가 (나쁜 방법!)
export JAVA_OPTS="-Xms1024m -Xmx2048m"
```

**✅ 올바른 방법** (setenv.sh 생성)
```bash
# /home/tomcat/tomcat/bin/setenv.sh 생성
export JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx2048m"
```

---

### 예제 2: 네이티브 라이브러리 경로 설정 (현재 프로젝트)

**문제:** nFilter의 `.so` 파일을 찾지 못함

**해결:** `setenv.sh` 생성
```bash
#!/bin/sh
export JAVA_OPTS="$JAVA_OPTS -Djava.library.path=/home/tomcat/tomcat/webapps/ROOT/WEB-INF/native"
export LD_LIBRARY_PATH="/home/tomcat/tomcat/webapps/ROOT/WEB-INF/native:$LD_LIBRARY_PATH"
```

---

### 예제 3: 여러 JVM 옵션 추가

```bash
#!/bin/sh
# /home/tomcat/tomcat/bin/setenv.sh

# 힙 메모리 설정
export JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx4096m"

# GC 로그 설정
export JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails -XX:+PrintGCDateStamps"

# 네이티브 라이브러리 경로
export JAVA_OPTS="$JAVA_OPTS -Djava.library.path=/path/to/native/libs"

# 타임존 설정
export JAVA_OPTS="$JAVA_OPTS -Duser.timezone=Asia/Seoul"
```

---

## 🎯 핵심 요약

1. **startup.sh**: 사용자가 실행하는 진입점, `catalina.sh` 호출
2. **catalina.sh**: Tomcat 실행 핵심 로직, `setclasspath.sh`와 `setenv.sh` 로드
3. **setclasspath.sh**: Java 환경 검증 (JAVA_HOME, JRE_HOME)
4. **setenv.sh**: 사용자 정의 JVM 옵션 (수정 가능, 업그레이드 안전)

**기억할 것:**
- Tomcat 원본 파일(startup.sh, catalina.sh, setclasspath.sh)은 수정하지 말 것!
- 사용자 설정은 **무조건 `setenv.sh`에만** 작성!

---

## 📚 참고 자료

- [Apache Tomcat 공식 문서](https://tomcat.apache.org/)
- Tomcat 설치 경로: `/home/tomcat/tomcat/`
- 스크립트 위치: `/home/tomcat/tomcat/bin/`
