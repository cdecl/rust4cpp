# C++ 개발자를 위한 Rust 안내서

> 기준: Rust 2024 Edition · Stable Rust · C++17/20/23 관점

이 문서는 모던 C++ 개발자가 Rust를 학습할 때 필요한 개념과 설계 차이를 설명합니다. 문법을 일대일로 번역하는 데 그치지 않고, C++에서 익숙한 RAII, 값 의미론, 참조, 스마트 포인터, 템플릿, 예외 처리 등이 Rust에서는 어떤 모델로 표현되는지 살펴봅니다.

두 언어 모두 가비지 컬렉터 없이 자원과 메모리를 관리하고 높은 성능을 목표로 합니다. 핵심 차이는 안전성을 확보하는 방식입니다. C++은 개발자에게 폭넓은 제어권을 주는 대신 규칙 준수를 요구하고, Rust는 소유권·대여·타입 시스템을 통해 안전한 코드의 상당 부분을 컴파일 타임에 검증합니다.

이 문서는 다음 원칙으로 구성합니다.

- **대응 관계는 참고용이다.** C++의 기능과 Rust의 기능이 비슷해 보여도 의미와 보장 수준이 다를 수 있습니다.
- **안전성과 동작을 구분한다.** 안전한 Rust의 보장과 `unsafe`, FFI, 런타임 패닉이 개입하는 경우를 구분합니다.
- **실무에서 필요한 흐름을 다룬다.** 소유권뿐 아니라 테스트, 동시성, 비동기, FFI, 프로젝트 구성까지 연결합니다.
- **예제는 개념 중심이다.** 각 코드 블록은 설명을 위한 예제이며, 여러 블록을 하나의 파일에 그대로 붙여 실행하는 프로젝트 예제와는 구분합니다.

## 목차

| 장 | 제목 | C++ 대응 |
| :--- | :--- | :--- |
| 1 | 두 언어의 성격 차이 | RAII와 예외 안전성 |
| 2 | 개발 환경: Cargo | CMake · Conan · vcpkg |
| 3 | 변수, 함수, 표현식 | `auto` · 함수 · `?:` |
| 4 | 타입과 컬렉션 | `std::string` · `vector` · `span` |
| 5 | 소유권과 대여 검사기 | 이동 의미론 · 참조 · 수명 |
| 6 | 포인터와 스마트 포인터 | `unique_ptr` · `shared_ptr` · `weak_ptr` |
| 7 | 데이터 모델과 패턴 매칭 | `class` · `enum` · `variant` |
| 8 | 트레이트와 다형성 | 템플릿 · Concepts · `virtual` |
| 9 | 표준 트레이트 | 특수 멤버 함수 · 연산자 오버로딩 |
| 10 | 모듈과 패키지 | 헤더 · namespace · CMake |
| 11 | 에러 처리 | 예외 · `std::error_code` · `std::expected` |
| 12 | C++ 개발자의 전환 가이드 | 모던 C++ 설계 관점 |
| 13 | 테스트와 품질 도구 | CTest · GoogleTest · 정적 분석 |
| 14 | 동시성 프로그래밍 | `std::thread` · `mutex` · atomic |
| 15 | 비동기 프로그래밍 | `std::future` · coroutine |
| 16 | C/C++ 연동과 `unsafe` | C ABI · 포인터 · FFI |
| 17 | 실무 학습 로드맵 | 프로젝트 전환 |
| 부록 A | 기존 내용에서 바로잡은 핵심 개념 | 개념 교정 |
| 부록 B | 참고할 공식 문서 | 공식 북, 레퍼런스 등 |
| 부록 C | [🦀 핵심 문법 & 메모리 치트시트](#부록-c-🦀-rust-핵심-문법--메모리-치트시트-c--python-3자-비교) | C++ / Python 3자 비교 |

**가장 중요한 장은 5장(소유권과 대여)입니다.** 다만 6장(스마트 포인터), 8장(트레이트), 11장(`Result`)까지 연결해서 읽어야 실제 설계에 적용할 수 있습니다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 1. 두 언어의 성격 차이 (C++: RAII와 예외 안전성 문화)

**한 줄 요약: C++은 "알아서 조심"을, Rust는 "대신 막아줌"을 선택했습니다.**

비유하자면 C++은 트랙에 올려둔 스포츠카입니다. 핸들도 브레이크도 전부 개발자 손에 있고, 한계를 넘으면 그냥 미끄러집니다. Rust는 같은 엔진에 안전장치를 잔뜩 단 차입니다. 코너에 너무 빠르게 들어가려 하면 차가 알아서 속도를 줄입니다. 답답할 때가 있지만, 그 차가 전복되는 일은 없습니다.

C++ 진영의 조언은 대부분 "절대 하지 마라" 형태입니다. *Effective C++*가 쌓아온 규칙들이 그렇습니다. 댕글링 포인터를 만들지 마라, 소유권 없는 포인터를 삭제하지 마라, 자기 대입을 조심하라. 좋은 규칙이지만 지키는 건 사람이고, 사람은 코드 리뷰에서도 놓칩니다.

Rust는 같은 항목을 컴파일러가 직접 검사합니다. 금지 사항을 외울 필요가 없습니다. 대신 타입 시스템과 관용구를 이해해야 하고, 그게 이 문서의 내용입니다.

| 비교 항목 | C++ (17/20/23) | Rust (2021/2024) |
| :--- | :--- | :--- |
| **메모리 안전성** | 개발자 주의 + 리뷰 + 정적 분석 도구 | 컴파일러의 대여 검사기가 강제 |
| **자원 관리** | RAII, 소멸자 `~T()` | 소유권 + `Drop` 트레이트 |
| **기본 가변성** | 기본 가변, 필요하면 `const` | 기본 불변, 필요하면 `let mut` |
| **동시성 안전성** | 런타임 동기화(`std::mutex` 등)에 의존 | `Send` / `Sync`로 컴파일 타임에 데이터 경합 차단 |

> **C++ 감각으로는** 자유도가 낮아진 게 아니라, 실수할 수 있는 지점이 줄어든 것입니다. 안전한 Rust 코드만으로 구성된 부분에서는 데이터 경합과 댕글링 참조 같은 핵심 메모리 안전성 문제가 방지됩니다. `unsafe` 구현, FFI, 잘못된 외부 계약은 별도 검토가 필요합니다. 물론 `unsafe` 블록을 쓰면 그 보장은 잠시 사라집니다.

> **팁** 처음 며칠은 컴파일 에러가 C++보다 훨씬 많이 나옵니다. 정상입니다. 그 에러들은 원래 C++에서 런타임에 터졌을 사고들입니다.

**한 줄 정리**
- Rust의 제약은 벌칙이 아니라 사고 예방이다.
- 안전한 Rust 코드의 메모리 안전성 규칙을 컴파일러가 검사했다고 이해하는 편이 정확하다. `unsafe`, FFI, 외부 라이브러리의 계약 위반까지 자동으로 증명하는 것은 아니다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 2. 개발 환경: CMake 대신 Cargo (C++: `CMake` · `vcpkg` · `Makefile`)

**한 줄 요약: 빌드 시스템, 패키지 관리자, 테스트 러너가 `cargo` 하나에 들어 있습니다.**

C++ 프로젝트를 새로 시작할 때를 떠올려 보세요. CMake를 쓸지, Make를 쓸지 정해야 하고, 의존성은 Conan인지 vcpkg인지 Git Submodule인지 골라야 하고, IDE마다 프로젝트 파일을 다시 만들어야 합니다. 코드를 한 줄도 안 썼는데 반나절이 갑니다.

Rust는 레고 정품 세트입니다. 상자를 열면 필요한 게 다 들어 있고, 설명서 순서대로 조립하면 됩니다. 부품이 안 맞는 일은 없습니다.

* **`rustup`**: 툴체인 관리자입니다. Stable/Beta/Nightly 채널 전환, 크로스 컴파일 타겟 추가를 명령 한 줄로 처리합니다.
* **`cargo`**: 빌드 시스템 + 패키지 관리자입니다. `Cargo.toml`에 의존성을 적으면 `crates.io`에서 알아서 받아 빌드합니다.
* **`cargo check`**: 링크를 생략하고 타입 검사와 대여 검사만 수행합니다. C++ 템플릿 프로젝트의 긴 컴파일 시간에 지친 사람에게 특히 반가운 기능입니다. 저장하고 `cargo check`만 돌리는 루프가 Rust 개발의 기본 리듬입니다.

```bash
# 프로젝트 생성부터 실행까지
cargo new my_project --bin
cd my_project

cargo check   # 타입 + 소유권 검사만 (빠름)
cargo build   # 실제 바이너리 생성
cargo run     # 빌드하고 실행
cargo test    # 테스트 실행
```

> **C++ 감각으로는** `Cargo.toml`은 패키지 메타데이터와 의존성·타깃 설정의 중심 파일입니다. 다만 CMake의 모든 기능을 대체하는 단일 파일은 아닙니다. 복잡한 네이티브 빌드에는 `build.rs`, `.cargo/config.toml`, workspace 설정 등이 함께 쓰입니다. `#include` 경로 설정, 링크 순서, 라이브러리 탐색 실패를 고민할 일이 거의 없습니다.

> **팁** `cargo check`를 습관으로 만드세요. 빌드보다 몇 배 빠르고, 소유권 에러는 99% 여기서 잡힙니다.

**한 줄 정리**
- `rustup`은 툴체인, `cargo`는 빌드와 의존성, `cargo check`는 빠른 피드백.
- 새 프로젝트는 `cargo new`로 시작하고, 패키지 메타데이터와 의존성은 `Cargo.toml`에서 관리한다.

#### Edition과 컴파일러 버전

Rust의 Edition은 언어 문법과 의미의 호환성 경계이며, 컴파일러 버전과는 다른 개념입니다. 2024 Edition은 Rust 1.85부터 안정화되었습니다. 새 프로젝트는 특별한 호환성 요구가 없다면 `edition = "2024"`를 기준으로 시작할 수 있습니다.

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2024"
rust-version = "1.85" # 프로젝트가 요구하는 최소 Rust 버전의 예
```

`rust-version`은 최소 지원 컴파일러 버전을 선언하는 필드입니다. 실제 프로젝트에서는 사용하는 의존성과 CI 환경을 고려해 결정하세요.

---

<div style="break-before: page; page-break-before: always;"></div>

## 3. 변수, 함수, 그리고 값이 되는 문장 (C++: `auto` · 함수 선언 · `?:`)

**한 줄 요약: Rust 변수는 기본이 불변이고, 바꾸려면 `mut`을 붙입니다.**

`let`은 인쇄된 포스터, `let mut`은 화이트보드라고 생각하세요. 포스터에 내용을 고치려면 새로 인쇄해야 합니다(섀도잉). 화이트보드는 지우고 다시 쓰면 됩니다(재할당). C++은 반대입니다. 기본이 화이트보드이고, 내용을 못 바꾸게 하려면 `const`를 붙여야 합니다.

| 선언 | 가변성 | 타입 추론 | 초기화 시점 | 비고 |
| :--- | :--- | :--- | :--- | :--- |
| `T x` | 가변 | 일부(`auto`) | 런타임 가능 | C++ 기본 상태 |
| `const T x` | 불변(런타임) | 일부 | 런타임 가능 | `const auto` |
| `constexpr T x` | 불변(컴파일 타임) | 일부 | 컴파일 타임 필수 | C++11 이상 |
| `let x` | 불변 | 지원 | 런타임 가능 | Rust 기본, 섀도잉 가능 |
| `let mut x` | 가변 | 지원 | 런타임 가능 | 값만 변경(타입 고정) |
| `const X: T` | 불변 상수 | 타입 필수 | 상수 표현식 필요 | 값은 컴파일 시 평가되며 일반 변수처럼 저장 위치를 보장하지 않음 |
| `static X: T` | 기본 불변 | 타입 필수 | 정적 수명 | 고정된 메모리 위치를 가짐. `static mut` 접근은 최신 Rust에서 엄격한 `unsafe` 규칙 적용 |

### 3.1. `let`과 `let mut` (C++: `auto` · `const`)

```cpp
// C++: 기본은 가변, 고정하려면 const
const int x = 10;
// x = 20; // 컴파일 에러: read-only

int y = 30;
y = 40; // 당연히 됨
```

```rust
// Rust: 기본은 불변, 바꾸려면 mut
let x = 10;
// x = 20; // 컴파일 에러: cannot assign twice to immutable variable

let mut y = 30;
y = 40; // mut을 붙였으니 가능
```

여기서 흔히 나오는 질문이 "`const`를 똑같이 쓰면 되지 않나"입니다. 아닙니다. Rust의 `const`는 컴파일 타임 상수라서 함수 호출 결과나 런타임 값을 담을 수 없습니다(타입도 반드시 명시해야 합니다). 런타임에 정해지지만 이후 절대 바뀌지 않는 값에는 `let`을 씁니다.

```rust
const MAX_RETRY: u32 = 3;        // 컴파일 타임 상수, 타입 필수
static GREETING: &str = "안녕";   // 고정 메모리 주소

fn main() {
    let limit = 10 * 2;          // 런타임 계산 결과를 담는 불변 변수
    println!("{} {} {}", MAX_RETRY, GREETING, limit);
}
```

### 3.2. 섀도잉 (Shadowing) (C++: 내부 스코프 변수 가리기)

Rust는 같은 스코프에서 `let`을 다시 쓰면 같은 이름의 **새 변수**가 만들어집니다. 기존 값을 덮어쓰는 게 아니라, 새 상자가 앞 상자를 가립니다. C++에서는 같은 스코프 재선언이 컴파일 에러이고, 중첩 블록에서 이름을 가리는 건 `-Wshadow` 경고 대상입니다. Rust에서는 이게 관용구입니다.

섀도잉을 쓰는 이유는 세 가지입니다.

1. **타입을 바꿀 수 있다.** `let mut`은 값만 바꿀 수 있고 타입은 못 바꿉니다. 섀도잉은 새 변수라서 타입이 자유롭습니다.
2. **가변성을 잠글 수 있다.** 잠깐 `mut`으로 조립한 뒤 같은 이름으로 불변 섀도잉하면, 이후 코드에서 실수로 바뀌지 않습니다.
3. **이름을 아낄 수 있다.** `spaces_len`, `data_sorted`, `x_inner` 같은 꼬리표 이름이 사라집니다. 블록을 벗어나면 가려졌던 바깥 변수가 다시 보입니다.

```cpp
// C++: 이름을 새로 지을 수밖에 없다
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

int main() {
    std::string spaces = "   ";
    // size_t spaces = spaces.size(); // 컴파일 에러: 재선언
    size_t spaces_len = spaces.size(); // 이름을 하나 더 만든다

    std::vector<int> data = {3, 1, 2};
    std::sort(data.begin(), data.end());
    // 같은 이름 'data'로 불변 재선언은 불가능하다

    int x = 5;
    x = x + 1;
    {
        int inner_x = x * 2; // 충돌을 피하려고 다른 이름
        std::cout << inner_x << "\n";
    }
    std::cout << x << "\n";
}
```

```rust
fn main() {
    // 1. 타입 변경: &str -> usize
    let spaces = "   ";
    let spaces = spaces.len();
    println!("공백 길이: {}", spaces);

    // 2. 가변 -> 불변 전환: 이후 push가 막힌다
    let mut data = vec![3, 1, 2];
    data.sort();
    let data = data;
    // data.push(4); // 컴파일 에러!

    // 3. 중첩 스코프: 블록을 벗어나면 바깥 x가 다시 보인다
    let x = 5;
    let x = x + 1; // 6
    {
        let x = x * 2; // 12, 이 블록 안에서만
        println!("안쪽 x: {}", x);
    }
    println!("바깥 x: {}", x); // 6
}
```

> **주의** 섀도잉은 이전 변수를 즉시 해제하지 않습니다. 이름만 가려질 뿐이고, 실제 해제는 그 스코프가 끝날 때 일어납니다. 큰 버퍼를 섀도잉했다고 메모리가 바로 줄어드는 건 아닙니다. 조기 해제가 필요하면 `drop(x)`를 씁니다.

**한 줄 정리**
- 기본은 불변, 바꿀 때만 `let mut`. 컴파일 타임 상수는 `const`.
- 섀도잉은 타입 변경과 불변성 잠금을 위한 도구다.

---

### 3.3. 함수 정의와 반환값 (C++: `auto f() -> T` · `void`)

**한 줄 요약: Rust 함수는 `fn 이름(인자: 타입) -> 반환타입` 순서로 쓰고, 마지막 표현식이 곧 반환값입니다.**

함수는 재료를 넣으면 결과가 나오는 자판기입니다. 시그니처는 그 자판기 앞면에 붙은 "투입구 모양" 표시입니다. C++은 타입을 이름 앞에 두고, Rust는 이름 뒤에 콜론으로 붙입니다. 같은 라벨을 왼쪽에 붙였느냐 오른쪽에 붙였느냐의 차이입니다.

```cpp
// C++: 타입이 이름 앞에 온다
#include <iostream>
#include <string>

int add(int a, int b) { return a + b; }

void greet(const std::string& name) {
    std::cout << "Hello, " << name << "\n";
}
```

```rust
// Rust: 이름 뒤에 콜론으로 타입을 붙인다
fn add(a: i32, b: i32) -> i32 {
    a + b          // 세미콜론 없음 = 이 값이 반환값
}

fn greet(name: &str) {
    println!("Hello, {}", name);   // 반환 타입 생략 = ()
}
```

| C++ 표기 | Rust 표기 | 비고 |
| :--- | :--- | :--- |
| `int f(int a)` | `fn f(a: i32) -> i32` | 타입이 이름 뒤로 이동 |
| `void f()` | `fn f()` | 반환 타입을 생략하면 `()` |
| `return x;` (본문 끝) | `x` (마지막 줄, 세미콜론 없음) | 이게 기본형 |
| `return x;` (중간 탈출) | `return x;` | 조기 반환에만 씀 |
| `[[noreturn]]` | `-> !` | never 타입 |

`()`는 "돌려줄 게 없다"는 뜻의 진짜 타입입니다. C++ `void`를 타입 자리에 쓸 수 있게 만든 것이라고 보면 됩니다. `-> ()`라고 명시해도 되지만 보통 생략합니다.

> **주의** 마지막 줄에 세미콜론을 붙이면 반환값이 사라집니다. `a + b;`는 "계산하고 버린다"는 뜻이 되어 `()`가 되고, `-> i32`와 맞지 않아 컴파일 에러가 납니다. 초보자가 가장 많이 만나는 에러 중 하나입니다.

> **팁** 파라미터도 기본이 불변입니다. `fn f(mut x: i32)`처럼 `mut`을 붙일 수는 있지만, 관용적인 코드는 파라미터를 건드리지 않고 `let mut x = x;`로 새 바인딩을 만듭니다.

> **C++ 감각으로는** `void f()`가 `fn f()`가 되고, 함수 포인터 `int(*)(int)`가 `fn(i32) -> i32`가 됩니다. C++의 `int main()`은 `fn main()`이고, 반환값을 쓰지 않으면 `()`입니다.

**한 줄 정리**
- 타입은 이름 뒤에 온다. `x: T`.
- 마지막 표현식이 반환값이다. `return`은 조기 반환용이다.
- `void`는 `()`이고, "절대 돌아오지 않음"은 `!`다.

### 3.4. 문장이 아니라 값: `if`, `match`, 블록 (C++: `?:` 연산자 · 문장식)

**한 줄 요약: Rust의 `if`, `match`, `{}` 블록은 문장이 아니라 값을 내는 표현식입니다. 그래서 삼항연산자가 없습니다.**

C++의 `if`는 "이쪽 길로 가라"는 표지판입니다. Rust의 `if`는 "두 상자 중 하나를 골라 건네는 직원"입니다. 표지판은 길만 알려주지만, 직원은 물건을 손에 쥐여 줍니다. 값이 필요하면 직원에게 물으면 되므로, 굳이 `?:`라는 별도 문법을 둘 이유가 없습니다.

```cpp
// C++: 값이 필요하면 삼항연산자를 쓴다
int x = flag ? 10 : 20;
```

```rust
// Rust: if 자체가 값을 낸다
let x = if flag { 10 } else { 20 };
```

블록도 값입니다. 중괄호로 감싼 마지막 표현식이 블록의 값이 됩니다.

```rust
let area = {
    let w = 10;
    let h = 5;
    w * h          // 이 블록의 값
};
```

`match`도 마찬가지입니다. 7장에서 자세히 다루지만, 형태만 먼저 봅니다.

```rust
let name = match code {
    200 => "OK",
    404 => "Not Found",
    _ => "Unknown",
};
```

| C++ | Rust | 비고 |
| :--- | :--- | :--- |
| `cond ? a : b` | `if cond { a } else { b }` | 삼항연산자 없음 |
| `switch` | `match` | fallthrough 없음, 완전성 검사 |
| `do { } while (c);` | `loop { ... if !c { break; } }` | `do..while` 없음 |
| `x++` / `x--` | `x += 1` / `x -= 1` | 증감 연산자 없음 |

반복은 세 가지입니다.

```rust
// while: 조건이 참인 동안
let mut n = 3;
while n > 0 {
    n -= 1;
}

// loop: 무한 반복. break로 값을 내보낼 수 있다
let mut count = 0;
let result = loop {
    count += 1;
    if count == 5 {
        break count * 2;   // break에 값을 실어 보냄
    }
};

// for: 이터레이터를 순회 (8.4절에서 자세히)
for i in 0..3 {
    println!("{}", i);     // 0, 1, 2
}
```

조건에 괄호를 쓰지 않고, 중괄호는 반드시 씁니다. `else if` 체인은 그대로 쓸 수 있습니다.

선언과 대입을 나눠 두는 문법도 있습니다. C++에서 흔한 "조건마다 한 번씩만 초기화"가 그대로 됩니다.

```rust
let x;
if flag {
    x = 1;      // 여기서 초기화
} else {
    x = 2;      // 또는 여기서. 두 경로 모두 한 번씩이면 OK
}
```

> **주의** `if`를 값으로 쓰려면 `else`가 반드시 있어야 합니다. `let x = if c { 1 };`은 "조건이 거짓일 때 값이 없다"는 뜻이라 컴파일되지 않습니다. C++ 삼항연산자에서 두 갈래를 반드시 쓰던 것과 같은 이유입니다.

> **주의** `switch`에 대응하는 `match`에는 fallthrough가 없습니다. 각 갈래는 `=>` 뒤의 표현식 하나로 끝나고, 다음 갈래로 흘러가지 않습니다. 대신 모든 경우를 처리했는지 컴파일러가 검사합니다.

**한 줄 정리**
- `if`, `match`, 블록은 값을 낸다. 삼항연산자가 없는 이유다.
- `if`를 값으로 쓰려면 `else`가 필요하다.
- `while` / `loop` / `for` 세 가지. `do..while`과 `++`는 없다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 4. 타입과 문자열 (C++: `char` · `std::string` · `std::string_view`)

**한 줄 요약: 숫자 타입은 거의 1:1로 대응되지만, `char`는 4바이트이고 문자열은 "내 것"과 "빌린 것"으로 나뉩니다.**

C++에서 문자열을 다룰 때 `std::string`과 `std::string_view`를 구분해서 썼다면, Rust의 `String`과 `&str`은 같은 구분입니다. 내가 산 책과, 도서관 책의 특정 페이지를 가리키는 메모. 메모만 들고는 책을 고칠 수 없고, 책이 반납되면 메모는 의미를 잃습니다. Rust는 그 "의미를 잃는 순간"을 컴파일 타임에 잡아냅니다.

### 4.1. 스칼라 타입 매핑 (C++: `int8_t` · `size_t` · `bool`)

Rust의 정수와 부동소수점은 C++11의 고정 크기 타입과 사실상 같습니다. 다른 건 `char`와 포인터 크기 타입 정도입니다.

| C++ 타입 | Rust 타입 | 설명 |
| :--- | :--- | :--- |
| `int8_t` / `uint8_t` | `i8` / `u8` | 8비트 부호 있음/없음 |
| `int32_t` / `uint32_t` | `i32` / `u32` | 32비트. `i32`가 Rust의 기본 정수 |
| `int64_t` / `uint64_t` | `i64` / `u64` | 64비트 |
| `std::size_t` | `usize` | 포인터 크기 부호 없는 정수 |
| `double` / `float` | `f64` / `f32` | IEEE 754 64비트 / 32비트 |
| `char` (1바이트) | `u8` 또는 `char` | C++ `char`는 1바이트, Rust `char`는 4바이트 유니코드 |

> **주의** Rust의 `char`는 UTF-32 인코딩 단위라기보다 **Unicode scalar value 하나**를 나타내는 4바이트 타입입니다. `'A'`와 `'한'` 모두 `char` 하나지만, 사용자에게 보이는 글자(grapheme cluster)는 여러 `char`로 구성될 수 있습니다. UTF-8 바이트 단위가 필요하면 `u8`을 사용하세요. C++ 감각으로 `char`를 쓰면 크기에서 어긋납니다.

### 4.2. `String`과 `&str` (C++: `std::string` 소유 · `string_view` 대여)

C++20의 `std::string` / `std::string_view` 구도와 같습니다. Rust 문자열은 항상 유효한 UTF-8입니다.

* **`String`**: 힙에 할당된 소유 문자열. 포인터 + 용량 + 길이. (`std::string` 대응)
* **`&str`**: 문자열 일부를 가리키는 불변 뷰. 포인터 + 길이. (`std::string_view` 대응)

```cpp
// C++: 소유 문자열과 뷰
#include <iostream>
#include <string>
#include <string_view>

void print_view(std::string_view sv) {
    std::cout << "View: " << sv << "\n";
}

int main() {
    std::string s = "Hello";
    s.append(" World");

    std::string_view sv = s;   // 빌린 뷰, s가 살아있는 동안만 유효
    print_view(sv);
    print_view("Literal String"); // 리터럴도 string_view로 전달 가능
}
```

```rust
// Rust: String과 &str
fn print_view(sv: &str) {
    println!("View: {}", sv);
}

fn main() {
    let mut s: String = String::from("Hello");
    s.push_str(" World");

    let sv: &str = &s;         // s를 빌린 뷰
    print_view(sv);
    print_view("Literal String"); // 문자열 리터럴의 타입이 &'static str
}
```

> **C++ 감각으로는** `std::string_view`가 "원본이 살아있는지 확인 안 해주는" 물건이라면, `&str`은 대여 검사기가 원본 수명을 추적해 주는 물건입니다. 뷰가 원본보다 오래 살면 컴파일이 안 됩니다.

#### 멀티라인 문자열과 Raw String

정규식, JSON, SQL처럼 이스케이프가 많은 문자열을 쓸 때 C++은 `R"(...)"`를, Rust는 `r#"..."#`를 씁니다. Rust는 일반 `"..."` 리터럴도 줄바꿈을 그대로 허용합니다.

```cpp
// C++: 줄마다 따옴표, 또는 Raw String Literal
#include <iostream>

int main() {
    const char* multi = "첫 번째 줄\n"
                        "두 번째 줄";

    // 이스케이프 없이 그대로
    const char* raw_json = R"({
  "name": "Alice",
  "path": "C:\Users\cdecl"
})";

    std::cout << multi << "\n" << raw_json << "\n";
}
```

```rust
fn main() {
    // 일반 리터럴에 줄바꿈을 그대로 넣을 수 있다
    let multi = "첫 번째 줄
두 번째 줄";

    // 줄 끝에 \를 붙이면 개행과 앞쪽 공백이 사라진다
    let joined = "한 줄로 \
                  이어 붙이기";

    // Raw String: # 개수를 늘리면 내부에 "# 문자열도 넣을 수 있다
    let raw_json = r#"{
  "name": "Alice",
  "path": "C:\Users\cdecl"
}"#;

    println!("{}\n{}\n{}", multi, joined, raw_json);
}
```

### 4.3. 출력 포맷팅 (C++: `iostream` · `std::format` · `printf`)

`printf`의 타입 불일치, `std::cout`의 긴 `<<` 체인 대신 Rust는 `println!`, `print!`, `format!` 매크로를 씁니다. 서식 문자열은 **컴파일 타임에 검증**되므로 인자 개수나 타입이 틀리면 빌드가 안 됩니다.

`{}`는 `Display` 트레이트(사용자에게 보여줄 형태), `{:?}`는 `Debug` 트레이트(개발자가 볼 구조)입니다.

```cpp
// C++20: std::format
#include <format>
#include <iostream>
#include <vector>

int main() {
    const char* name = "Alice";
    int age = 30;

    std::string msg = std::format("Name: {}, Age: {}", name, age);
    std::cout << msg << "\n";

    std::vector<int> numbers = {1, 2, 3};
    std::cout << "Debug View: [" << numbers[0] << ", "
              << numbers[1] << ", " << numbers[2] << "]\n";
}
```

```rust
fn main() {
    let name = "Alice";
    let age = 30;

    let msg: String = format!("Name: {}, Age: {}", name, age);
    println!("{}", msg);

    // 컨테이너는 {:?}로 한 번에 출력
    let numbers = vec![1, 2, 3];
    println!("Debug View: {:?}", numbers);
}
```

> **C++ 감각으로는** C++20에서는 `std::format`으로 문자열을 만든 뒤 `std::cout`에 넘겨야 했지만, C++23부터는 `<print>` 헤더의 `std::println("{}", x)`가 도입되어 Rust의 `println!("{}", x)`와 거의 같은 문법으로 바로 출력할 수 있습니다.

> **팁** 구조체를 `{:?}`로 찍고 싶으면 `#[derive(Debug)]` 한 줄만 붙이면 됩니다. C++에서 `operator<<`를 직접 쓰던 수고가 사라집니다.

### 4.4. 타입 변환 (C++: `static_cast` 계열)

**한 줄 요약: Rust는 원시 타입끼리 암묵적 변환을 허용하지 않습니다. `as`, `From`, `TryFrom` 셋 중 하나를 골라 명시해야 합니다.**

C++은 `int`를 `double`에 넣으면 알아서 해줍니다. Rust는 안 해줍니다. 귀찮아 보이지만, 조용히 일어나던 손실이 전부 눈에 보이게 됩니다.

| 하고 싶은 일 | Rust | 특징 |
| :--- | :--- | :--- |
| 정수끼리 강제 변환 | `x as T` | 범위 검사 없음. 넘치면 상위 비트가 잘린다 |
| float → 정수 변환 | `x as T` | 0 방향 절단. 범위를 벗어나면 포화(최소/최대), NaN은 0 |
| 실패하지 않는 변환 | `T::from(x)` / `x.into()` | `From`은 실패하지 않는 변환 계약이지, 모든 도메인에서 수학적으로 무손실임을 보장하는 것은 아님 |
| 실패할 수 있는 변환 | `T::try_from(x)` / `x.try_into()` | 실패 시 `Err` 반환 |
| 포인터 변환 | `ptr as *const T` | 역참조는 `unsafe` 필요 |

```cpp
// C++: static_cast로 명시, 위험은 개발자가 감수
#include <cstdint>
#include <iostream>

int main() {
    int32_t a = 300;
    int64_t b = static_cast<int64_t>(a); // 300
    uint8_t c = static_cast<uint8_t>(a); // 44 (상위 비트 잘림)

    double pi = 3.99;
    int32_t truncated = static_cast<int32_t>(pi); // 3

    uint8_t code = 65;
    char ch = static_cast<char>(code); // 'A'
}
```

```rust
fn main() {
    // 1. as: 비트 강제 변환. 잘림을 막아주지 않는다
    let a: i32 = 300;
    let b: i64 = a as i64;  // 300
    let c: u8 = a as u8;    // 44 (300 = 256 + 44)

    // 2. float -> 정수: 0 방향으로 절단
    let pi: f64 = 3.99;
    let truncated: i32 = pi as i32;       // 3
    let negative: i32 = (-3.99) as i32;   // -3 (내림이 아니라 0 방향)

    // 범위를 벗어나면 포화: 최소/최대에서 멈춘다 (Rust 1.45+)
    let big: f64 = 1e30;
    let saturated: i32 = big as i32;      // 2147483647
    let nan: i32 = f64::NAN as i32;       // 0

    // 3. u8 -> char
    let code: u8 = 65;
    let ch: char = code as char;    // 'A'

    // 4. C 스타일 enum -> 정수
    #[repr(i32)]
    enum Status { Ok = 200, NotFound = 404 }
    let status_code = Status::NotFound as i32; // 404

    println!("{} {} {} {} {}", b, c, truncated, ch, status_code);
}
```

`as`는 C++의 `static_cast`와 비슷합니다. 편하지만 300을 `u8`로 바꿀 때 44가 되는 걸 막아주지 않습니다. 값이 실제로 안전한지 확인하고 싶다면 `TryFrom`을 씁니다.

```rust
use std::convert::{TryFrom, TryInto};

fn main() {
    // 손실 없는 변환: From / Into (컴파일 타임에 안전성 보장)
    let small: i32 = 42;
    let big: i64 = i64::from(small);
    let big2: i64 = small.into();       // 같은 일
    // let err = i32::from(big);        // 컴파일 에러! 손실 가능성 때문에 From이 없다

    // 실패 가능한 변환: TryFrom / TryInto
    let large: i32 = 300;
    match u8::try_from(large) {
        Ok(val) => println!("변환 성공: {}", val),
        Err(e) => println!("범위 초과: {}", e), // 실행됨
    }

    let valid: Result<u8, _> = 100_i32.try_into();
    assert_eq!(valid, Ok(100));
}
```

> **C++ 감각으로는** `static_cast`는 그대로 있고(=`as`), `std::in_range` 체크를 직접 쓰던 부분이 `TryFrom` 하나로 타입 시스템에 들어왔다고 보면 됩니다.

> **주의** 원시 포인터 캐스팅(`const_cast` + `reinterpret_cast` 자리)은 `as`로 가능하지만, 역참조는 반드시 `unsafe` 블록 안에서 해야 합니다. 이 문서에서는 다루지 않습니다. 안전한 Rust에서는 raw pointer를 쓸 일이 거의 없습니다.

### 4.5. 배열, `Vec`, 슬라이스 (C++: `std::array` · `std::vector` · `std::span`)

`std::array` / `std::vector` / `std::span` 구도가 그대로 있습니다. 문자열에서 배운 "내 것 vs 빌린 것"이 컬렉션에도 적용됩니다.

| 구분 | C++ | Rust | 특징 |
| :--- | :--- | :--- | :--- |
| 고정 크기 배열 | `std::array<T, N>` | `[T; N]` | 스택 할당, 길이가 타입의 일부 |
| 동적 배열 | `std::vector<T>` | `Vec<T>` | 힙 할당, 크기 가변, 소유권 보유 |
| 불변 슬라이스 | `std::span<const T>` (C++20) | `&[T]` | 포인터 + 길이, 소유권 없음 |
| 가변 슬라이스 | `std::span<T>` (C++20) | `&mut [T]` | 빌린 구간을 제자리 수정 |

C++과 갈리는 지점은 두 가지입니다.

1. **포인터 붕괴가 없습니다.** C++의 `int arr[5]`는 함수에 넘기면 `int*`가 되면서 길이를 잃습니다. Rust의 `&[T]`는 항상 `.len()`을 들고 다닙니다.
2. **인덱싱이 기본으로 경계 검사를 합니다.** C++ `vec[i]`는 넘치면 UB, Rust `vec[i]`는 넘치면 즉시 `panic!`입니다. 패닉 없이 처리하려면 `.get(i)`로 `Option<&T>`를 받습니다.

```cpp
// C++20: array, vector, span
#include <array>
#include <iostream>
#include <span>
#include <vector>

void print_slice(std::span<const int> s) {
    std::cout << "len=" << s.size() << ": ";
    for (int v : s) std::cout << v << " ";
    std::cout << "\n";
}

void double_in_place(std::span<int> s) {
    for (int& v : s) v *= 2;
}

int main() {
    std::array<int, 3> arr = {10, 20, 30};
    std::array<int, 4> zeros{};           // 모두 0

    std::vector<int> vec = {1, 2, 3, 4, 5};
    vec.push_back(6);

    print_slice(std::span(arr));
    print_slice(std::span(vec).subspan(1, 3));     // [2, 3, 4]
    double_in_place(std::span(vec).subspan(0, 2)); // 앞 2개 2배

    if (10 < vec.size()) {
        std::cout << vec[10] << "\n";
    } else {
        std::cout << "인덱스 범위 초과\n";
    }
}
```

```rust
fn print_slice(s: &[i32]) {
    println!("len={}: {:?}", s.len(), s);
}

fn double_in_place(s: &mut [i32]) {
    for v in s.iter_mut() {
        *v *= 2;
    }
}

fn main() {
    let arr: [i32; 3] = [10, 20, 30];
    let zeros = [0; 4];                 // [0, 0, 0, 0]

    let mut vec = vec![1, 2, 3, 4, 5];
    vec.push(6);

    print_slice(&arr);                  // [T; N] -> &[T] 자동 강제
    print_slice(&vec[1..4]);            // [2, 3, 4]
    double_in_place(&mut vec[0..2]);    // 앞 2개 2배

    match vec.get(10) {
        Some(v) => println!("{}", v),
        None => println!("인덱스 범위 초과"),
    }
}
```

슬라이스에는 C++에서 직접 루프를 돌려야 했던 작업이 메서드로 들어 있습니다.

* `slice.chunks(2)`: 2개씩 겹치지 않게 나눠 순회
* `slice.windows(2)`: 크기 2의 슬라이딩 윈도우 (인접 원소 비교)
* `slice.split_at(mid)` / `split_at_mut(mid)`: 한 지점에서 두 조각으로 안전하게 분리

```rust
fn main() {
    let v = [1, 2, 3, 4, 5];

    for pair in v.chunks(2) {
        println!("chunk: {:?}", pair);   // [1,2] [3,4] [5]
    }
    for w in v.windows(2) {
        println!("{} -> {}", w[0], w[1]); // 인접 쌍
    }

    let (left, right) = v.split_at(2);
    println!("{:?} | {:?}", left, right); // [1,2] | [3,4,5]
}
```

> **팁** `.get()`은 실패를 값으로 돌려주므로 서버 코드에서 특히 유용합니다. 인덱스는 프로그래머가 확신할 때만, `.get()`은 입력이 외부에서 올 때 쓰면 됩니다.

**한 줄 정리**
- `char`는 4바이트 유니코드, 문자열은 `String`(소유)과 `&str`(빌림)으로 나뉜다.
- 암묵적 타입 변환은 없다. `as`는 강제, `From`은 안전, `TryFrom`은 실패 가능.
- 슬라이스는 길이를 항상 들고 다니고, 인덱싱은 기본으로 검사한다.

---

### 4.6. 숫자 리터럴과 연산자 (C++: 접미사 `u` · `++` · 오버플로 UB)

**한 줄 요약: 리터럴에 타입을 붙여 쓸 수 있고(`34u8`), 진법 표기와 자릿수 구분자를 지원합니다. 대신 `++`와 `--`는 없습니다.**

숫자 리터럴은 "이름표가 붙은 택배 상자"입니다. 대부분은 내용물을 보고 이름표를 추측할 수 있지만(정수면 `i32`, 소수점이 있으면 `f64`), 헷갈릴 때는 이름표를 직접 붙입니다. C++은 `34u`처럼 붙였고, Rust는 `34u8`처럼 크기까지 붙입니다.

```cpp
// C++14: 진법, 자릿수 구분자, 접미사
uint8_t mask = 0b1010'1010;
int big = 0x1F'FFFF;
unsigned int u = 34u;
```

```rust
// Rust: 구분자는 _ , 접미사는 타입 이름 전체
let mask: u8 = 0b1010_1010;
let big: u32 = 0x1F_FFFF;
let octal = 0o755;
let u = 34u8;                 // 타입을 리터럴에 직접 붙임

let default_int = 42;         // i32로 추론
let default_float = 3.14;     // f64로 추론
```

추론 기본값은 `i32`와 `f64`입니다. 둘 다 C++의 `int`/`double`과 같은 크기라, 대부분은 이름표 없이 넘어갑니다.

연산자는 C++과 거의 같지만 제약이 하나 늘어납니다. **비트 연산은 정수에만, 논리 연산은 `bool`에만** 쓸 수 있습니다.

```rust
let a: u8 = 0b1100;
let b: u8 = 0b1010;

println!("{}", a & b);   // 8   (0b1000)
println!("{}", a | b);   // 14  (0b1110)
println!("{}", a ^ b);   // 6   (0b0110)
println!("{}", !a);      // 243 (0b0011) — C++의 ~a

let t = true;
let f = false;
println!("{}", t && f);  // false
// println!("{}", a && t);  // 컴파일 에러: &&는 bool 전용
```

`++`와 `--`는 없습니다. `x += 1`을 씁니다. 3.4절에서 본 대로 이 언어는 "값을 내는 표현식"을 선호하고, 부수 효과만 있는 증감 연산자는 그 철학과 맞지 않습니다.

오버플로 동작은 빌드 모드에 따라 다릅니다. 이건 C++과 크게 다르므로 한 번은 짚고 갑니다.

```rust
// 디버그 빌드: 넘치면 panic (원인을 바로 알려줌)
// 기본 릴리스 프로파일(오버플로 검사 비활성): 랩어라운드
let mut v: u8 = 250;
v += 10;

// 의도를 명시하고 싶을 때
let w = 250u8.wrapping_add(10);     // 4   (랩어라운드 강제)
let c = 250u8.checked_add(10);      // None (실패를 값으로)
let s = 250u8.saturating_add(10);   // 255 (최대값에서 멈춤)
```

> **주의** C++의 unsigned 정수 오버플로는 모듈러 연산으로 정의되고 signed 정수 오버플로는 UB입니다. Rust의 일반 정수 연산은 오버플로 검사 설정에 따라 달라집니다. 기본 디버그 프로파일은 보통 panic, 기본 릴리스 프로파일은 래핑 동작을 사용하지만 `overflow-checks` 설정으로 바꿀 수 있습니다. "릴리스에서만 도는 코드"를 디버그로 시험하면 여기서 갈립니다. 오버플로가 정상 동작인 코드라면 `wrapping_add`로 의도를 남기세요.

> **C++ 감각으로는** `unsigned`의 조용한 랩어라운드에 의존하던 코드는 Rust에서 디버그 빌드가 즉시 터집니다. 좋은 의미의 차이입니다. 조용한 버그가 시끄러운 버그로 바뀌는 것뿐입니다.

**한 줄 정리**
- 리터럴에 타입을 붙일 수 있다. `34u8`, `0b1010_1010`.
- 추론 기본값은 `i32`, `f64`.
- `++`/`--`는 없다. `+= 1`을 쓴다.
- 오버플로 동작은 `overflow-checks` 설정에 좌우된다. 기본 디버그는 검사하고 기본 릴리스는 래핑한다.

### 4.7. 슬라이스의 실체와 성능 이야기 (C++: `std::span` · 포인터 + 길이)

**한 줄 요약: `&[T]`는 주소 하나가 아니라 "주소 + 길이" 두 워드입니다. 그래서 경계 검사가 가능하고, 그래서 크기가 정해지지 않은 타입이 존재합니다.**

슬라이스는 도서관 책의 페이지 범위를 적은 메모입니다. 메모에는 "몇 쪽부터"와 "몇 장"이 같이 적혀 있습니다. 주소만 있으면 어디서 끝나는지 알 수 없으니, 두 가지를 함께 들고 다닙니다.

```rust
let v = vec![10, 20, 30, 40, 50];
let s: &[i32] = &v[1..4];          // 20, 30, 40

println!("{}", s.len());           // 3  — 길이를 따로 들고 있으므로 O(1)
println!("{}", s[0]);              // 20 — 경계를 검사할 수 있다
```

| 참조 | 크기 | 구성 |
| :--- | :--- | :--- |
| `&i32` | 1워드 | 주소 |
| `&[i32]` | 2워드 | 주소 + 길이 |
| `&dyn Trait` | 2워드 | 주소 + vtable 포인터 |

주소와 길이를 함께 들고 다니는 참조를 **팻 포인터**(fat pointer)라고 부릅니다. 그리고 `[i32]`, `str`, `dyn Trait`처럼 길이나 구현이 런타임에 정해지는 타입을 **크기가 정해지지 않은 타입**(DST)이라고 합니다. DST는 변수에 직접 담을 수 없고 반드시 참조나 `Box` 뒤에 옵니다.

```rust
let s: str = "hello";              // 컴파일 에러: 크기를 알 수 없음
let s: &str = "hello";             // OK: 팻 포인터
let boxed: Box<str> = "hello".into();  // OK: 힙에 두고 팻 포인터로
```

`Vec`은 재할당 비용이 있습니다. C++에서 `reserve`를 쓰던 자리에 `with_capacity`가 있습니다.

```cpp
// C++: 미리 공간을 잡아 재할당을 막는다
std::vector<int> v;
v.reserve(1000);
for (int i = 0; i < 1000; ++i) v.push_back(i);
```

```rust
// Rust: 같은 역할
let mut v = Vec::with_capacity(1000);
for i in 0..1000 {
    v.push(i);                     // 재할당 없음
}
```

경계 검사를 끄고 싶을 때는 `unsafe`가 필요합니다. 탈출구는 있지만, 마지막 수단입니다.

```rust
let v = [10, 20, 30];
let x = unsafe { *v.get_unchecked(1) };   // 경계 검사 생략
```

> **주의** 경계 검사 비용은 대부분의 루프에서 최적화 단계에 사라집니다. "검사 때문에 느리다"고 느끼기 전에 프로파일하세요. `get_unchecked`는 인덱스가 범위를 벗어날 때 UB이고, 그 UB는 컴파일러가 막아주지 않습니다.

`[]` 문법은 `Index` 트레이트로 열려 있습니다. C++의 `operator[]`와 같습니다.

```rust
use std::ops::Index;

struct Grid {
    cells: Vec<i32>,
    width: usize,
}

impl Index<(usize, usize)> for Grid {
    type Output = i32;

    fn index(&self, (x, y): (usize, usize)) -> &i32 {
        &self.cells[y * self.width + x]
    }
}

let g = Grid { cells: vec![1, 2, 3, 4], width: 2 };
println!("{}", g[(1, 1)]);   // 3  (x=1, y=1 → cells[1 * 2 + 1])
```

마지막으로, 배열은 **값 타입**입니다. `let b = a;`는 포인터 복사가 아니라 내용 전체 복사입니다.

```rust
let a = [0u8; 1024];
let b = a;              // 1KB 복사. 힙 할당 없음
```

> **주의** 큰 배열을 스택에 두면 스택 오버플로가 납니다. 수 MB짜리 버퍼는 `vec![0u8; 4 * 1024 * 1024]`로 힙에 두세요. C++에서 `std::array`를 지역 변수로 두던 습관이 그대로 오면 여기서 걸립니다. 참고로 `Box::new([0u8; 4 * 1024 * 1024])`는 배열을 **먼저 스택에 만든 뒤** 옮기므로, 큰 크기에서는 해결책이 되지 못합니다.

**한 줄 정리**
- `&[T]`는 주소 + 길이, 2워드다. 그래서 경계 검사가 가능하다.
- 재할당을 막으려면 `Vec::with_capacity`.
- 경계 검사 생략은 `unsafe { get_unchecked }`, 마지막 수단이다.
- 배열은 값 타입이다. 큰 배열은 힙에 둔다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 5. 소유권과 대여 검사기 (C++: 소멸자 · `T&` · 댕글링 포인터)

**한 줄 요약: 값마다 주인이 하나뿐이고, 주인이 사라지면 값도 사라집니다. 빌려 쓰는 동안에는 규칙이 붙습니다.**

이 절이 Rust가 C++과 가장 크게 갈리는 지점입니다. C++의 이동 후 객체는 일반적으로 **유효하지만 상태가 지정되지 않은(valid but unspecified)** 상태입니다. 타입이 문서화한 불변식은 유지되지만, 이동 전 값이나 구체적인 내용에 의존해서는 안 됩니다. Rust는 비-`Copy` 값을 이동한 뒤 원래 바인딩의 재사용을 타입 검사로 막습니다. 소멸자는 호출되고, 실수로 접근하면 조용히 이상한 값이 나옵니다. Rust는 이동한 쪽을 아예 못 쓰게 만듭니다.

택배 상자로 생각해 보세요. 상자는 한 사람만 들 수 있습니다. 옆 사람에게 넘기면 내 손에는 없습니다. 남의 상자를 보기만 하려면 잠깐 열어 보면 되지만, 내가 상자에 물건을 넣고 있는 동안에는 다른 사람이 동시에 열어 보면 안 됩니다. Rust는 이 규칙을 컴파일러가 지킵니다.

### 5.1. 소유권의 세 가지 규칙 (C++: 소멸자 · 복사/이동 의미론)

1. 모든 값에는 **소유자**가 있습니다.
2. 한 순간에 소유자는 **하나뿐**입니다.
3. 소유자가 스코프를 벗어나면 값은 **자동으로 해제**됩니다(`Drop`).

```rust
fn main() {
    let s1 = String::from("hello");   // s1이 소유자
    let s2 = s1;                      // 소유권이 s2로 이동(move)

    // println!("{}", s1);            // 컴파일 에러! value borrowed after move
    println!("{}", s2);               // s2만 유효

    let n1 = 42;
    let n2 = n1;                      // i32는 Copy라서 복사됨. 둘 다 유효
    println!("{} {}", n1, n2);
}   // s2 해제 -> 힙 메모리 반환
```

> **C++ 감각으로는** `std::string s2 = std::move(s1);`을 쓴 뒤 `s1`을 다시 쓰는 코드를 컴파일러가 막아주는 셈입니다. 그리고 `int`처럼 값만 복사되는 타입은 원래대로 복사됩니다.

### 5.2. 대여: 불변 참조와 가변 참조 (C++: `const T&` · `T&`)

소유권을 매번 넘기면 불편합니다. 그래서 Rust는 참조자(`&T`, `&mut T`)로 빌려 쓰는 방법을 줍니다. 대여 검사기는 다음 규칙을 컴파일 타임에 강제합니다.

* **불변 참조 `&T`**: 동시에 여러 개 있어도 됩니다. (C++ `const T&`)
* **가변 참조 `&mut T`**: 같은 스코프에서 **딱 하나**만 가능합니다. (C++ `T&`)
* **배타성**: 가변 참조가 살아있는 동안 불변 참조는 있을 수 없습니다.

도서관 비유가 정확합니다. 같은 책을 여러 명이 동시에 읽는 건 문제없습니다. 그런데 누군가 연필로 밑줄을 긋고 있다면, 다른 사람은 그 책을 펼쳐 보면 안 됩니다.

```cpp
// C++: 컴파일은 되지만 런타임에 UB가 될 수 있는 코드
#include <iostream>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3};
    int& ref = vec[0];

    vec.push_back(4); // 재할당이 일어나면 ref는 댕글링 참조!

    // std::cout << ref << "\n"; // UB
}
```

```rust
// Rust: 같은 실수를 컴파일러가 막는다
fn main() {
    let mut vec = vec![1, 2, 3];
    let ref_elem = &vec[0];      // 불변 대여 시작

    // vec.push(4);              // 컴파일 에러!
    // cannot borrow `vec` as mutable because it is also borrowed as immutable

    println!("첫 원소: {}", ref_elem); // 불변 대여 사용 완료
    vec.push(4);                 // 이제 가변 대여 가능
    println!("{:?}", vec);
}
```

여기서 배울 점은 에러 메시지가 알려주는 해결책입니다. "불변 대여가 끝난 뒤에 push하라"는 뜻이므로, `println!`을 위로 올리면 그만입니다. 대여 검사기와 싸우는 대신 순서를 바꾸는 겁니다.

### 5.3. 왜 이 규칙이 데이터 경합을 막는가 (C++: `std::mutex` · 데이터 경합)

데이터 경합(data race)은 세 조건이 동시에 성립할 때 생깁니다. 두 스레드가 같은 메모리에, 최소 하나는 쓰기로, 동기화 없이 접근하는 경우입니다.

Rust에서는 "가변 참조는 하나뿐"이라는 규칙 때문에 이 세 조건이 동시에 성립할 수 없습니다. 쓰기 접근은 `&mut`이고, `&mut`는 유일해야 하니 다른 스레드가 같은 메모리를 동시에 볼 수 없습니다. 그래서 Rust는 "컴파일되면 데이터 경합은 없다"고 말할 수 있습니다. 이 보장이 스레드로 확장된 것이 9장의 `Send` / `Sync`입니다.

> **주의** 이 규칙은 `unsafe`를 쓰면 깨집니다. `unsafe`는 "나는 규칙을 지킬 자신이 있다"는 선언이고, 지키지 못하면 다시 UB입니다. 안전한 Rust 코드만 쓰는 한 이 보장은 유지됩니다.

> **팁** 컴파일 에러가 났을 때 스스로에게 물어볼 세 가지. (1) 이 값의 소유자는 누구인가? (2) 참조가 필요한가, 소유권을 넘겨야 하는가? (3) 이 참조는 언제까지 살아야 하는가?

**한 줄 정리**
- 값의 주인은 하나. 넘기면 원래 쪽은 못 쓴다.
- 불변 참조는 여럿, 가변 참조는 하나. 이 배타성이 데이터 경합을 컴파일 타임에 없앤다.

---

### 5.4. 수명: 참조가 유효한 기간 (C++: 댕글링 참조 · 임시 수명 연장)

**한 줄 요약: Rust의 수명은 "이 참조를 몇 시까지 쓸 수 있는가"를 타입에 적어 두는 장치입니다. 대부분은 컴파일러가 추론하므로 직접 쓰지 않습니다.**

도서관 책을 빌리면 반납 기한이 정해집니다. 빌린 책에 메모를 붙여 두는 건 자유지만, 책이 반납된 뒤에 그 메모를 읽으면 아무것도 가리키지 못합니다. Rust는 이 기한을 타입에 적습니다. `&'a T`의 `'a`가 기한입니다.

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;          // x의 수명은 이 블록까지
    }
    // println!("{}", r);   // 컴파일 에러: x가 이미 사라짐
}
```

C++은 같은 코드가 컴파일되고, 실행하면 쓰레기 값을 읽습니다. Rust는 컴파일 단계에서 멈춥니다.

| 상황 | C++ | Rust |
| :--- | :--- | :--- |
| 참조가 원본보다 오래 삼 | 컴파일됨(UB 가능) | 컴파일 에러 |
| `std::string_view`가 원본 소멸 후 사용 | 컴파일됨 | 컴파일 에러 |
| 수명 표기 | 없음(타입 시스템 밖) | `&'a T`, 대부분 생략 |
| 프로그램 전체 수명 | `static` 변수 | `&'static T` |

대부분의 함수는 수명을 쓰지 않습니다. 참조가 하나 들어가고 하나 나오면, 컴파일러가 "들어온 것과 같은 기간"으로 알아서 묶습니다. 이걸 수명 생략 규칙이라고 합니다.

```rust
// 수명을 안 써도 되는 경우: 참조가 하나뿐
fn first_word(s: &str) -> &str {
    s.split_whitespace().next().unwrap_or("")
}
```

명시가 필요한 경우는 정해져 있습니다. 참조가 **둘 이상** 들어오고 그중 하나를 반환할 때입니다.

```rust
// a와 b 중 누가 살아남는지 컴파일러가 알 수 없다
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() > b.len() { a } else { b }
}
```

읽는 법은 "`a`와 `b`는 같은 기간 `'a` 동안 유효하고, 반환값도 그 기간 동안만 유효하다"입니다.

> **주의** `'a`를 붙인다고 수명이 늘어나지 않습니다. 수명 표기는 **관계를 서술**할 뿐이고, 실제 기간은 값이 사는 스코프가 정합니다. "수명을 늘리는 문법"으로 오해하면 컴파일 에러를 이해할 수 없습니다.

> **팁** 문자열 리터럴 `"hello"`의 타입은 `&'static str`입니다. 프로그램이 끝날 때까지 사는 정적 메모리에 있기 때문입니다. 4.2절에서 `&'static str`이 나온 이유가 이것입니다.

> **C++ 감각으로는** C++에서 `std::string_view`나 참조를 반환할 때 생기던 댕글링을, 컴파일러가 호출 그래프 전체를 훑어 잡아주는 셈입니다. 런타임 비용은 0입니다. 수명은 타입 검사 단계에서만 존재하고 코드에는 남지 않습니다.

**한 줄 정리**
- 수명은 참조의 유효 기간이고, 대부분은 생략된다.
- 참조가 둘 이상 들어와 하나를 반환할 때만 `'a`를 쓴다.
- `'static`은 프로그램 전체 수명. 문자열 리터럴이 그렇다.
- 수명 표기는 관계를 적는 것이지 기간을 늘리는 것이 아니다.

### 5.5. 값으로 넘길까, 참조로 넘길까 (C++: 값 전달 · `const T&`)

**한 줄 요약: Rust는 인자 자리에서 참조를 자동으로 만들어 주지 않습니다. `&`를 직접 붙여야 하고, 이 한 글자가 소유권 이동 여부를 결정합니다.**

C++은 "값을 넘기라고 썼는데 참조로 받는" 함수를 자연스럽게 받아들입니다. Rust는 그 자동 변환을 하지 않습니다. 값을 넘기면 정말 넘어가고(이동), 빌려주려면 빌려준다고 명시해야 합니다. 택배로 보낼지, 잠깐 빌려줄지 고르는 것과 같습니다.

```cpp
// C++: const T&에 값을 넘기면 자동으로 참조가 만들어진다
void print(const std::string& s);
print(name);            // OK
```

```rust
// Rust: 타입이 &str이면 호출할 때도 &
fn print(s: &str) {
    println!("{}", s);
}

let name = String::from("Rust");
print(&name);           // & 를 직접 붙여야 한다
// print(name);         // 컴파일 에러: &str 자리에 String
```

세 가지 전달 방식을 표로 정리하면 이렇습니다.

| 시그니처 | 의미 | 호출하는 쪽 | 호출 후 원본 | C++ 대응 |
| :--- | :--- | :--- | :--- | :--- |
| `fn f(x: T)` | 소유권 이전 | `f(v)` | 사용 불가 | `f(std::move(v))` |
| `fn f(x: &T)` | 읽기 빌림 | `f(&v)` | 그대로 | `f(const T&)` |
| `fn f(x: &mut T)` | 쓰기 빌림 | `f(&mut v)` | 그대로, 수정됨 | `f(T&)` |

```rust
fn takes_ownership(s: String) {
    println!("{}", s);
}

fn modify(v: &mut Vec<i32>) {
    v.push(1);
}

fn main() {
    let s = String::from("hi");
    takes_ownership(s);
    // println!("{}", s);        // 에러: 이미 이동됨

    let mut v = vec![];
    modify(&mut v);              // &mut 를 명시
    println!("{:?}", v);         // [1]
}
```

예외가 하나 있습니다. **역참조 강제**(Deref coercion)는 인자 자리에서도 동작합니다. `&String`은 `&str` 자리에, `&Vec<T>`는 `&[T]` 자리에 그대로 들어갑니다.

```rust
fn len_of(s: &str) -> usize { s.len() }

let owned = String::from("hello");
println!("{}", len_of(&owned));   // &String -> &str 자동 변환
```

`&`를 붙이는 것과 `Box`를 벗기는 것도 다릅니다. `&boxed`는 `&Box<T>`이고, 이 역시 역참조 강제로 `&T` 자리에 들어갑니다. 반대로 `boxed`를 그냥 넘기면 소유권이 이동합니다.

> **주의** C++ 개발자가 가장 자주 만나는 컴파일 에러가 "expected `&T`, found `T`"입니다. 함수 본문이 아니라 호출 지점을 보세요. 인자 자동 참조는 없고, `&`를 빠뜨린 것입니다.

> **팁** 작은 `Copy` 타입(`i32`, `f64`, 좌표 구조체)은 값으로 넘기는 편이 오히려 단순하고 빠릅니다. `&`를 붙이는 기준은 "복사 비용이 큰가"와 "원본을 계속 써야 하는가" 두 가지입니다.

> **주의** 임시 값의 참조를 반환하면 컴파일 에러입니다. `fn make() -> &str { &String::from("x") }`는 5.4절의 수명 규칙에 걸립니다. 이때는 `String`을 반환하세요. 소유권을 넘기면 문제가 사라집니다.

**한 줄 정리**
- 값 전달은 이동, `&`는 읽기 빌림, `&mut`은 쓰기 빌림.
- 인자 자동 참조는 없다. `&`를 직접 붙인다.
- 예외는 역참조 강제(`&String` → `&str`, `&Vec<T>` → `&[T]`).
- 빌린 값을 반환하려면 수명이 맞아야 한다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 6. 포인터와 스마트 포인터 (C++: `T*` · `unique_ptr`)

**한 줄 요약: `Box`는 혼자 사는 집, `Rc`/`Arc`는 공동 소유 계약서입니다.**

### 6.1. 세 가지 스마트 포인터 (C++: `unique_ptr` · `shared_ptr` · `weak_ptr`)

C++에서 `std::unique_ptr`, `std::shared_ptr`를 쓰던 자리에 그대로 대응하는 타입이 있습니다. 이름만 다르고 역할은 같습니다.

| Rust | C++ 대응 | 한 줄 설명 |
| :--- | :--- | :--- |
| `Box<T>` | `std::unique_ptr<T>` | 단독 소유 힙 할당. 포인터 크기 수준의 표현이며, 할당 자체 비용은 존재 |
| `Rc<T>` | 단일 스레드용 `shared_ptr` | 비원자적 참조 카운트. 기본 공유 접근은 불변이며, `RefCell<T>` 등으로 내부 가변성을 구성할 수 있음 |
| `Arc<T>` | `std::shared_ptr<T>` | 원자적 참조 카운트. 내부 데이터의 스레드 안전성은 `T`와 동기화 방식에 달림 |
| `*const T` / `*mut T` | `const T*` / `T*` | 원시 포인터. 역참조는 `unsafe` |

#### `Box<T>`의 3대 핵심 용도와 C++ 차이점

`Box<T>`는 C++의 `std::unique_ptr<T>`와 대응되지만, Rust 특유의 타입 시스템과 결합하여 다음과 같은 3가지 핵심 용도로 사용됩니다:
1. **컴파일 타임 크기 확정 (재귀 자료구조)**: 컴파일 타임에 크기를 알 수 없는 재귀적 타입(Linked List, Tree, AST 등)을 힙으로 빼서 포인터 크기(8바이트)로 고정시킵니다.
2. **Move 비용 절감 및 스택 오버플로우 방지**: 대용량 데이터를 스택 프레임에 두지 않고 힙에 할당하여, 소유권 이동(Move) 시 거대한 데이터 복사 없이 8바이트 주소만 빠르게 복사합니다.
3. **트레이트 객체 다형성**: `Box<dyn Trait>` 형태로 동적 디스패치(런타임 vtable 다형성)를 구현할 때 필수적인 스마트 포인터입니다.

> **C++과의 결정적 차이 (Move 후 안전성)**: C++의 `std::unique_ptr`는 `std::move` 후에도 원본 변수가 `nullptr` 상태로 스코프에 남아있어, 부주의 시 런타임 댕글링/널 포인터 역참조 버그를 유발할 위험이 있습니다. 반면 Rust의 `Box<T>`는 Move가 일어나는 즉시 **컴파일러가 원본 변수에 대한 접근을 컴파일 타임에 원천 차단**합니다.

`Box`는 "이사 박스에 넣어 창고에 두기"입니다. 값은 힙에 있고, 스택에는 주소만 있습니다. 재귀 구조체나 큰 값을 옮길 때 씁니다. `Rc`와 `Arc`는 "공동 소유 계약서에 이름을 여러 개 올리기"입니다. 마지막 이름이 지워질 때 값이 해제됩니다.

```cpp
// C++: unique_ptr과 shared_ptr
#include <memory>

struct Node { int data; };

int main() {
    auto u_node = std::make_unique<Node>(Node{10}); // 혼자 소유
    auto s_node = std::make_shared<Node>(Node{20}); // 공유 소유
    auto s_node_clone = s_node;                     // 참조 카운트 2
}
```

```rust
// Rust: Box와 Arc
use std::sync::Arc;

struct Node { data: i32 }

fn main() {
    let b_node: Box<Node> = Box::new(Node { data: 10 });   // unique_ptr 대응
    let a_node: Arc<Node> = Arc::new(Node { data: 20 });   // shared_ptr 대응
    let a_node_clone = Arc::clone(&a_node);                // 참조 카운트 2

    println!("{} {}", b_node.data, a_node_clone.data);
}
```

> **C++ 감각으로는** `Rc<T>`는 `shared_ptr`에서 원자적 증가/감소를 뺀 경량 버전입니다. 그래서 다른 스레드로 넘길 수 없습니다(`Send`가 아님). 스레드 사이에서 공유하려면 반드시 `Arc`를 쓰세요.

> **주의** `Rc`로 순환 참조를 만들면 C++ `shared_ptr`과 똑같이 메모리가 안 풀립니다. 필요하면 `Weak`를 씁니다. 그리고 `Arc`는 읽기 전용 공유라서, 안의 값을 고치려면 `Mutex`가 필요합니다. C++의 `shared_ptr<mutex<T>>` 자리에 `Arc<Mutex<T>>`가 옵니다.

**한 줄 정리**
- 혼자 소유는 `Box`, 여러 명이 소유하면 `Rc`(단일 스레드) 또는 `Arc`(멀티 스레드).
- 원시 포인터는 존재하지만 안전한 Rust에서는 거의 쓰지 않는다.

---

### 6.2. 내부 가변성: `Cell`과 `RefCell` (C++: `mutable` · `const_cast`)

**한 줄 요약: 여러 명이 공유하는 값은 기본적으로 읽기 전용입니다. 안을 고쳐야 한다면 `Cell`(복사 가능한 값) 또는 `RefCell`(그 외 전부)을 함께 씁니다.**

공동 소유 계약서에 이름이 여러 개 올라 있으면 아무나 내용을 고칠 수는 없습니다. 고치려면 출입 장부를 두고 들어갈 때마다 서명해야 합니다. `RefCell`이 그 장부입니다. 서명 확인은 컴파일 타임이 아니라 **런타임**에 하고, 규칙을 어기면(읽는 중에 고치려 들면) 그 자리에서 멈춥니다.

| | `Cell<T>` | `RefCell<T>` |
| :--- | :--- | :--- |
| 대상 타입 | 모든 `T`를 감쌀 수 있음. `get()`은 `T: Copy`일 때만 제공 | 모든 타입 |
| 접근 방식 | `get()`(복사) / `set()` | `borrow()` / `borrow_mut()` |
| 검사 시점 | 없음(복사라 충돌이 없음) | 런타임 대여 검사 |
| 위반 시 | 해당 없음 | `panic!` |
| C++ 대응 | `mutable` 멤버, 값 복사 | `mutable` 멤버, 참조 반환 |

```rust
use std::cell::{Cell, RefCell};

fn main() {
    // Cell: 값을 복사해서 꺼내고, 통째로 갈아끼운다
    let counter = Cell::new(0);
    counter.set(counter.get() + 1);
    println!("{}", counter.get());      // 1

    // RefCell: 참조로 빌려준다. 대여 규칙은 런타임에 검사
    let log = RefCell::new(Vec::new());
    log.borrow_mut().push("start");
    println!("{:?}", log.borrow());     // ["start"]
}
```

C++에서 같은 일을 하던 코드는 `mutable`이었습니다.

```cpp
// C++: const 메서드에서도 카운터를 올리고 싶을 때
class Request {
    mutable int hits = 0;
public:
    int count() const { return ++hits; }
};
```

`Rc`와 조합하면 "여러 곳에서 소유하고, 아무 데서나 고치는" 값이 됩니다. 단일 스레드에서 `shared_ptr<T>` + `mutable`을 쓰던 자리입니다.

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn main() {
    let shared = Rc::new(RefCell::new(Vec::new()));
    let a = Rc::clone(&shared);
    let b = Rc::clone(&shared);

    a.borrow_mut().push(1);
    b.borrow_mut().push(2);

    println!("{:?}", shared.borrow());  // [1, 2]
}
```

> **주의** `borrow_mut()`을 두 번 겹쳐 부르면 런타임 panic입니다. 컴파일러가 잡아주지 않으니, 빌림이 겹치지 않게 스코프를 짧게 유지하세요. 실패해도 되는 자리라면 `try_borrow_mut()`이 `Result`를 돌려줍니다.

> **주의** `RefCell<T>`은 `T: Send`이면 소유권을 다른 스레드로 이동(`Send`)할 수 있지만, 여러 스레드에서 공유(`Sync`)할 수는 없습니다. 스레드 간 공유 가변성이 필요하면 보통 `Arc<Mutex<T>>` 또는 읽기 위주라면 `Arc<RwLock<T>>`를 검토합니다. 단일 스레드에서 `Arc<Mutex<T>>`를 쓰는 건 정확하지만 과합니다. 그 자리는 `Rc<RefCell<T>>`입니다.

> **C++ 감각으로는** C++의 `mutable`은 "이 멤버는 `const` 취급에서 빼달라"는 선언이고, `RefCell`은 같은 일을 하되 **대여 규칙까지 런타임에 검사**합니다. `const` 정확성 검사를 컴파일 타임에서 런타임으로 미룬 대가로, 공유 가변성을 안전하게 만들 수 있습니다.

**한 줄 정리**
- 공유 값은 기본이 불변이다. 가변이 필요하면 `Cell`/`RefCell`을 함께 쓴다.
- `Cell<T>`는 모든 타입을 감쌀 수 있고, `get()`은 `T: Copy`일 때 사용할 수 있다. `RefCell<T>`은 런타임 대여 검사를 제공한다.
- `RefCell`의 검사는 런타임이고, 위반하면 panic이다.
- 단일 스레드의 공유 가변은 `Rc<RefCell<T>>`.

### 6.3. 순환 참조와 `Weak` (C++: `weak_ptr` · `enable_shared_from_this`)

**한 줄 요약: `Rc`/`Arc`가 서로를 가리키면 참조 카운트가 0이 되지 않아 영원히 해제되지 않습니다. 한쪽을 `Weak`로 바꿔 소유 관계에서 빼야 합니다.**

두 사람이 서로의 계약서에 보증인으로 이름을 올렸습니다. 둘 다 "상대가 먼저 지우면 나도 지운다"고 기다리니 아무도 지우지 못합니다. `Weak`는 보증인 없는 관찰자입니다. 계약서를 들고 있지 않으니 카운트에 영향이 없고, 상대가 사라지면 조용히 `None`이 됩니다.

| | `Rc<T>` / `Arc<T>` | `Weak<T>` |
| :--- | :--- | :--- |
| 소유 | 한다 | 하지 않는다 |
| 참조 카운트 | 증가 | 증가하지 않음 |
| 값 접근 | 직접 | `upgrade()` → `Option<Rc<T>>` |
| 쓰는 자리 | 소유 관계(부모 → 자식) | 역방향 관찰(자식 → 부모) |

트리 구조에서 관용적인 배치는 "아래로는 강한 참조, 위로는 약한 참조"입니다.

```rust
use std::cell::RefCell;
use std::rc::{Rc, Weak};

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,          // 위로는 약하게
    children: RefCell<Vec<Rc<Node>>>,     // 아래로는 강하게
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    // 올라갈 때는 upgrade로 확인한다. 이미 해제됐으면 None
    if let Some(p) = leaf.parent.borrow().upgrade() {
        println!("parent = {}", p.value);   // 5
    }
}
```

그래프는 소유권 모델과 정면으로 부딪힙니다. 해법은 세 가지로 정리됩니다.

1. **방향을 정해 트리로 만든다.** 소유는 한 방향으로만 두고, 반대 방향은 `Weak`로 관찰한다.
2. **인덱스로 가리킨다.** 노드는 `Vec<Node>`에 두고, 참조 대신 `usize` 인덱스를 저장한다. 소유권 문제가 사라진다.
3. **아레나에 모아 둔다.** 모든 노드를 한 컨테이너에 두고 서로를 참조한다(`bumpalo`, `typed-arena` 같은 크레이트). 수명이 하나로 통일된다.

> **주의** `Weak`는 `Rc`/`Arc` 전용입니다. 원시 포인터와는 다른 물건이고, `Weak`를 쓰려면 값이 반드시 `Rc`나 `Arc`로 소유되고 있어야 합니다.

> **C++ 감각으로는** `shared_ptr` 순환을 `weak_ptr`로 끊던 그 습관이 그대로 통합니다. 다른 점은 Rust가 순환을 만들어도 컴파일러가 경고하지 않는다는 것입니다. 누수는 여전히 프로그래머 책임입니다.

**한 줄 정리**
- `Rc`/`Arc` 순환은 누수다. 한쪽을 `Weak`로 끊는다.
- 관용 배치는 아래로 강한 참조, 위로 약한 참조.
- 그래프는 `Weak`, 인덱스 참조, 아레나 셋 중 하나로 푼다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 7. 데이터와 행위의 분리 (C++: `class` · 멤버 함수)

**한 줄 요약: Rust는 데이터(`struct`)와 메서드(`impl`)를 따로 씁니다. 접근 제어는 클래스가 아니라 모듈 단위입니다.**

C++은 `class` 하나에 멤버 변수, 메서드, `public`/`private`를 다 넣습니다. Rust는 그 봉투를 두 개로 찢습니다. 데이터는 `struct`, 행위는 `impl`입니다. 처음엔 번거로워 보이지만, 나중에 "이 타입에 이 기능만 따로 붙이기"가 자연스러워집니다.

### 7.1. `struct`와 `impl` (C++: `class` · 멤버 함수 정의)

```cpp
// C++: 클래스 하나에 다 넣는다
#include <iostream>

class Rectangle {
private:
    double width;
    double height;

public:
    Rectangle(double w, double h) : width(w), height(h) {}

    double area() const { return width * height; }
};

int main() {
    Rectangle rect(10.0, 5.0);
    std::cout << "Area: " << rect.area() << "\n";
}
```

```rust
// Rust: struct + impl
pub struct Rectangle {
    width: f64,   // 기본은 모듈 내 비공개
    height: f64,
}

impl Rectangle {
    // 생성자 관례: new 라는 연관 함수 (static 메서드 자리)
    pub fn new(width: f64, height: f64) -> Self {
        Rectangle { width, height }
    }

    // &self는 C++의 const 멤버 함수 자리
    pub fn area(&self) -> f64 {
        self.width * self.height
    }

    // &mut self는 값을 고치는 메서드
    pub fn scale(&mut self, factor: f64) {
        self.width *= factor;
        self.height *= factor;
    }
}

fn main() {
    let mut rect = Rectangle::new(10.0, 5.0);
    println!("Area: {}", rect.area());
    rect.scale(2.0);
    println!("Area: {}", rect.area());
}
```

세 가지가 C++과 다릅니다.

* **생성자는 없다.** `Rectangle::new(...)` 같은 연관 함수를 관례적으로 `new`라고 부를 뿐입니다. `Default`를 구현하면 `Rectangle::default()`도 쓸 수 있습니다.
* **`self`가 명시적이며 언어 키워드다.**
  * **지위와 위치**: 반드시 첫 번째 매개변수여야 하는 정식 예약어(Keyword)입니다. (Python의 `def func(self):`와 역할이 같지만, Python은 관례일 뿐이고 Rust는 언어 키워드로 강제합니다.)
  * **타입 생략 (문법 설탕)**: `impl` 블록 안에서 첫 번째 인자가 `self`일 때만 타입을 생략할 수 있습니다.
    * `self` ➔ `self: Self` (값으로 소유권을 완전히 가져와 소비/파괴하는 메서드)
    * `&self` ➔ `self: &Self` (불변 참조 빌림, C++의 `const` 멤버 함수에 해당)
    * `&mut self` ➔ `self: &mut Self` (가변 참조 빌림, 인스턴스 값을 수정하는 메서드)
  * **소문자 `self` vs 대문자 `Self`**:
    * `self`: 현재 호출된 **인스턴스(값)** 자체.
    * `Self`: 현재 구조체/트레이트의 **타입(설계도)** (예: `fn new(...) -> Self`).
  * **자동 역참조 (Auto-dereferencing)**: C++에서는 포인터나 참조자일 때 `->`를 쓰거나 역참조 `(*self).field`를 해야 하지만, Rust는 점(`.`) 연산자가 자동으로 역참조를 수행하므로 `&mut self`에서도 `self.width`로 깔끔하게 접근합니다.
* **필드 접근 제어는 모듈 단위다.** `private:` 키워드가 없고, `pub`을 안 붙이면 그 모듈 밖에서 보이지 않습니다. 구조체 필드 하나하나에도 `pub`을 붙일 수 있습니다.

> **C++ 감각으로는** C++23에서 도입된 명시적 객체 매개변수(`void foo(this Self& self)` / `double area(this const Rectangle& self)`)가 바로 이 방식입니다. Rust는 처음부터 모든 메서드에 `&self`, `&mut self`처럼 수신자를 명시적으로 적습니다.

> **팁** `impl` 블록은 여러 개로 나눠도 됩니다. 그래서 `impl Display`, `impl Default`처럼 트레이트별로 블록을 따로 두는 게 관례입니다. 9장에서 계속 나옵니다.

**한 줄 정리**
- 데이터는 `struct`, 행위는 `impl`. 생성자는 `new` 관례.
- `&self`는 읽기 전용 메서드, `&mut self`는 수정 메서드.

---

### 7.2. `enum`: C의 enum이 아닙니다 (C++: `enum class` · `std::variant`)

**한 줄 요약: Rust의 `enum`은 숫자에 이름을 붙이는 도구가 아니라, 정해진 모양 중 하나를 담는 상자입니다. `Option<T>`가 이 상자로 null을 대체합니다.**

다용도 상자에 라벨이 붙어 있고, 라벨마다 요구하는 내용물이 다르다고 생각하세요. "숫자 둘" 라벨이면 정수 둘, "숫자 하나" 라벨이면 정수 하나, "없음" 라벨이면 아무것도 넣지 않습니다. 상자를 열 때는 라벨을 반드시 확인해야 하고, 확인하지 않으면 컴파일러가 막습니다.

C++에서 같은 일을 하려면 `std::variant`를 씁니다.

```cpp
// C++: variant로 흉내낸 태그된 공용체
#include <variant>

struct Add { int a, b; };
struct Lit { int v; };
using Expr = std::variant<Add, Lit>;

int eval(const Expr& e) {
    if (auto p = std::get_if<Add>(&e)) return p->a + p->b;
    return std::get<Lit>(e).v;
}
```

```rust
// Rust: enum이 언어 차원에 있다
enum Expr {
    Add(i32, i32),
    Lit(i32),
}

fn eval(e: &Expr) -> i32 {
    match e {
        Expr::Add(a, b) => a + b,
        Expr::Lit(v) => *v,
    }
}
```

결정적인 차이는 `match`의 완전성 검사입니다. `Lit` 갈래를 빼먹으면 컴파일이 안 됩니다. `std::variant`에서는 `std::get`이 런타임에 예외를 던지거나 `std::bad_variant_access`로 죽는 자리입니다.

| C++ | Rust |
| :--- | :--- |
| `enum class Color { Red, Green };` | `enum Color { Red, Green }` |
| `std::variant<A, B>` | `enum E { A(..), B(..) }` |
| `std::optional<T>` | `Option<T>` |
| `std::get_if` + if 체인 | `match` (완전성 검사) |
| variant마다 다른 데이터 | variant마다 다른 필드 구성 |

숫자만 쓰는 C 스타일 enum도 그대로 됩니다. 정수로 캐스팅할 때는 `#[repr(...)]`을 붙입니다.

```rust
#[repr(i32)]
enum Status {
    Ok = 200,
    NotFound = 404,
}

let code = Status::NotFound as i32;   // 404
```

> **주의** 데이터를 담는 variant와 숫자만 있는 variant를 한 enum에 섞으면 정수 캐스팅이 안 됩니다. "C 스타일로 숫자를 붙이는 enum"과 "데이터를 담는 enum"은 서로 다른 용도로 보고, 섞지 마세요. 4.4절의 캐스팅 예제는 전자입니다.

C++의 상속으로 만들던 "여러 모양 중 하나"는 상당 부분 enum으로 더 짧게 표현됩니다. 가상 함수와 `unique_ptr`로 이뤄지던 일이, 값 타입 하나와 `match` 한 블록으로 끝납니다.

### 7.3. `Option<T>`: null을 타입으로 대체 (C++: `std::optional` · `nullptr`)

**한 줄 요약: Rust에는 null이 없습니다. "값이 없을 수 있음"은 `Option<T>`라는 타입으로 표현하고, 꺼낼 때 `match`나 `if let`을 강제합니다.**

C++의 null은 "아무것도 가리키지 않는 포인터"라는 특별한 값입니다. Rust는 그 특별한 값을 없애고, 대신 "빈 칸이 인쇄된 신청서"를 씁니다. 빈 칸이 있으면 빈 칸이라고 읽어야 하고, 읽지 않고 넘어가면 컴파일이 안 됩니다.

`Option<T>`는 표준 라이브러리에 이렇게 정의되어 있습니다.

```rust
enum Option<T> {
    None,
    Some(T),
}
```

`T`가 무엇이든 담을 수 있고, "없음"은 `None`이라는 별도 variant입니다. C++의 `std::optional`과 역할이 같습니다.

```cpp
// C++: optional
#include <optional>

std::optional<size_t> find_index(const std::vector<int>& v, int target);

if (auto idx = find_index(v, 30)) {
    std::cout << *idx;
}
```

```rust
// Rust: Option
fn find_index(v: &[i32], target: i32) -> Option<usize> {
    v.iter().position(|x| *x == target)
}

let idx = find_index(&[10, 20, 30], 30);   // Some(2)
```

꺼내는 방법은 네 가지입니다.

```rust
// 1. match: 두 경우를 모두 처리
match idx {
    Some(i) => println!("인덱스 {}", i),
    None => println!("없음"),
}

// 2. if let: 있는 경우만 처리
if let Some(i) = idx {
    println!("인덱스 {}", i);
}

// 3. 기본값으로 대체
let i = idx.unwrap_or(0);

// 4. 없으면 즉시 반환 (? 연산자, 11장)
fn get(v: &[i32], n: usize) -> Option<i32> {
    let value = v.get(n)?;   // None이면 여기서 None 반환
    Some(*value)
}
```

> **주의** `unwrap()`은 값이 없을 때 panic합니다. "절대 없을 거야"라고 확신하는 자리(테스트, 프로토타입)에서는 괜찮지만, 사용자 입력이 지나가는 경로에서는 쓰지 마세요. `unwrap_or`, `unwrap_or_else`, `if let`, `?` 중 하나로 바꾸는 게 보통 정답입니다.

> **팁** `Option<Box<T>>`나 `Option<&T>`는 포인터 크기 그대로입니다. `None`이 null 포인터로 표현되도록 컴파일러가 최적화하기 때문입니다. "Option을 쓰면 메모리가 늘어난다"는 걱정은 대부분 사실이 아닙니다.

> **C++ 감각으로는** `std::optional`과 문법은 비슷하지만 결정적으로 다릅니다. Rust의 일반 참조(`&T`, `&mut T`)는 null이 될 수 없습니다. 값의 부재는 `Option<&T>`처럼 타입으로 표현합니다. C++의 포인터나 `optional<reference_wrapper<T>>`와 비교하는 편이 더 정확합니다. 검사를 잊는 것이 불가능한 `optional`이라고 보면 됩니다. 또한 C++23에서는 `std::optional`에도 `.and_then()`, `.transform()`, `.or_else()` 같은 모나딕 연산이 추가되어, Rust `Option`의 `.and_then()`, `.map()`, `.or_else()`와 같은 방식으로 값을 꺼내지 않고 연쇄 변환할 수 있습니다.

**한 줄 정리**
- `enum`은 variant마다 다른 데이터를 담는 상자다. `std::variant`의 상위 호환.
- `Option<T>`는 `None`/`Some(T)` 두 갈래이고, null을 대체한다.
- 꺼낼 때 `match`, `if let`, `unwrap_or`, `?` 중 하나를 반드시 쓴다.
- 숫자 캐스팅용 enum과 데이터용 enum은 섞지 않는다.

### 7.4. 패턴 매칭과 구조 분해 (C++: 구조적 바인딩 · `std::visit`)

**한 줄 요약: Rust는 `let`, 함수 인자, `match` 어디서든 값을 모양으로 분해할 수 있습니다. C++의 structured bindings가 언어 전체로 확장된 셈입니다.**

택배 상자를 열어 내용물을 미리 정해진 칸에 나눠 담는 일입니다. `let (a, b) = pair;`는 "이 상자에는 두 칸이 있으니 각각 꺼내라"는 뜻입니다.

```cpp
// C++17: structured bindings
std::pair<int, int> pair = {1, 2};
auto [a, b] = pair;
```

```rust
// Rust: 튜플
let pair = (1, 2);
let (a, b) = pair;
println!("{} {}", a, b);
println!("{}", pair.0);       // 인덱스로도 접근
```

튜플은 이름 없는 구조체입니다. 필드에 이름이 필요한 순간이 오면 튜플 구조체를 씁니다.

```rust
struct Point(i32, i32);        // 튜플 구조체

let p = Point(3, 4);
let Point(x, y) = p;
println!("{} {}", x, y);
println!("{}", p.0);           // 여전히 인덱스 접근 가능
```

필드가 아예 없는 구조체도 있습니다. 값이 하나뿐임을 타입으로 표시할 때 씁니다.

```rust
struct Meters;
struct Feet;

fn walk(distance: Meters) { /* ... */ }
// walk(Feet);   // 컴파일 에러: 타입이 다르다
```

함수 인자 자리에서도 분해할 수 있습니다.

```rust
fn distance((x1, y1): (i32, i32), (x2, y2): (i32, i32)) -> i32 {
    let dx = x2 - x1;
    let dy = y2 - y1;
    dx * dx + dy * dy
}

println!("{}", distance((0, 0), (3, 4)));
```

`match`는 패턴 매칭의 본체입니다. 리터럴, 범위, OR, 조건 가드, 바인딩을 한 블록에서 씁니다.

```rust
fn classify(n: i32) -> &'static str {
    match n {
        0 => "zero",
        1..=9 => "digit",
        x if x < 0 => "negative",   // 조건 가드
        _ => "big",
    }
}
```

`@`는 "값을 바인딩하면서 동시에 패턴으로 검사"합니다. `..`는 나머지 필드를 무시합니다.

```rust
match age {
    n @ 0..=12 => println!("어린이 {}", n),
    n @ 13..=19 => println!("청소년 {}", n),
    n => println!("성인 {}", n),
}

struct User {
    name: String,
    age: u32,
    email: String,
}

let user = User {
    name: "kim".to_string(),
    age: 30,
    email: "kim@example.com".to_string(),
};

let User { name, .. } = user;   // 나머지 필드는 무시
println!("{}", name);
```

`if let`과 `while let`은 "한 갈래만 관심 있을 때" 쓰는 축약형입니다.

```rust
if let Some(v) = maybe_value {
    println!("{}", v);
}

// 스택이 빌 때까지 꺼내기
while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

> **주의** `let`에는 반증 불가능한 패턴만 올 수 있습니다. `let Some(x) = opt;`는 "None이면 어떻게 하나"에 답이 없어서 컴파일 에러입니다. 이때는 `if let`을 쓰거나, 최신 문법으로 `let Some(x) = opt else { return; };`(let-else)를 씁니다.

> **팁** 참조를 매칭할 때 `&`나 `ref`를 일일이 붙이지 않아도 됩니다. 최신 Rust는 참조를 자동으로 벗겨서 매칭합니다(match ergonomics). 예전 코드에서 보이는 `ref`는 "이동하지 않고 빌리겠다"는 명시이고, 지금은 대부분 필요 없습니다.

> **C++ 감각으로는** `auto [a, b] = pair;`가 `let` 패턴에 해당하고, `std::visit`이 `match`에 해당합니다. 다른 점은 Rust의 `match`가 모든 경우를 요구한다는 것, 그리고 함수 인자와 반복문 조건에도 패턴을 쓸 수 있다는 것입니다.

**한 줄 정리**
- `let`, 함수 인자, `match` 어디서든 구조 분해가 된다.
- 튜플 / 튜플 구조체 / 유닛 구조체 세 가지 모양이 있다.
- `match`는 리터럴·범위·OR·가드·`@`·`..`를 한 블록에서 쓴다.
- `let`은 반증 불가 패턴만. 실패 가능하면 `if let`이나 let-else.

### 7.5. 깊은 불변성 (C++: `const` 멤버 · `constexpr`)

**한 줄 요약: Rust의 `mut`은 변수 하나에만 붙고, 그 변수를 통한 모든 접근이 가변입니다. 구조체 필드에 `mut`을 붙이는 문법은 없습니다.**

C++의 `const`는 "이 창구에서는 못 고침"입니다. 다른 창구로는 얼마든지 고칠 수 있습니다. Rust의 불변성은 "상자 전체가 봉인됨"입니다. 봉인을 풀려면 상자를 새로 만들어야 합니다. 그게 `let mut`입니다.

```cpp
// C++: const는 이 객체를 통한 접근에만 적용된다
struct S { int f; };

const S cs{1};
// cs.f = 2;          // 에러: const 객체를 통한 수정

S s{1};
s.f = 2;              // OK: const가 아닌 객체

void byRef(S& s) { s.f = 3; }   // 참조를 통하면 const 객체도 아닌 것처럼 다룰 수 있다
```

```rust
// Rust: 바인딩이 불변이면 필드 전체가 불변
struct S { f: i32 }

let s = S { f: 1 };
// s.f = 2;           // 컴파일 에러: 바인딩이 불변

let mut s = S { f: 1 };
s.f = 2;              // OK

// "이 필드만 가변"이라는 문법은 없다. 필요하면 RefCell을 쓴다
```

| C++ | Rust | 차이 |
| :--- | :--- | :--- |
| `T x;` | `let mut x;` | 기본값이 반대 |
| `const T x;` | `let x;` | |
| `const` 멤버 함수 | `fn f(&self)` | 수신자로 판단 |
| `T&` (비 const) | `&mut T` | |
| `mutable` 필드 | `Cell`/`RefCell` | 6.2절 |

참조 필드는 예외처럼 보이지만 규칙은 같습니다. 불변성은 **포인터 값 자체**에 적용되고, 포인터가 가리키는 곳은 별개입니다.

```rust
struct Holder<'a> {
    value: &'a mut i32,
}

let mut n = 0;
let h = Holder { value: &mut n };   // h 자체는 불변 바인딩
*h.value += 1;                       // 그래도 가리키는 대상은 고칠 수 있다
```

7.1절의 `fn scale(&mut self)`도 같은 규칙입니다. 메서드가 `self`를 가변으로 빌리겠다는 선언이고, 호출하려면 변수가 `let mut`이어야 합니다. 3장의 "기본 불변"과 모순이 아니라, 그 규칙이 메서드 호출까지 일관되게 적용된 것입니다.

> **주의** 필드 하나만 가변으로 두는 문법은 없습니다. 정말 필요하면 그 필드를 `RefCell<T>`로 감싸세요. 그러면 바깥 바인딩이 불변이어도 그 필드만 런타임 검사를 거쳐 수정할 수 있습니다.

> **C++ 감각으로는** C++의 `const` 정확성이 "객체 전체"로 확장된 것입니다. `const` 멤버 함수를 붙였다 뗐다 하던 판단이 `&self`/`&mut self`로 타입에 새겨지고, 컴파일러가 호출 지점마다 검사합니다.

**한 줄 정리**
- `mut`은 바인딩에만 붙고, 필드 전체에 적용된다.
- 필드별 가변성은 문법에 없다. 필요하면 `RefCell`.
- `&mut` 필드를 통한 수정은 바깥 불변성과 별개다.
- `&mut self` 메서드는 `let mut` 변수에서만 호출된다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 8. 트레이트와 다형성 (C++: `template` · `virtual` · 컨셉트)

**한 줄 요약: 상속 대신 "이런 메서드가 있으면 이 타입이다"라는 자격증(트레이트)으로 다형성을 만듭니다.**

C++은 부모 클래스를 상속하고 순수 가상 함수를 오버라이드해서 다형성을 만듭니다. Rust에는 상속이 없습니다. 대신 트레이트로 "무엇을 할 수 있는가"만 정의하고, 타입은 그 트레이트를 구현합니다. 리모컨 비유가 맞습니다. "전원 버튼과 볼륨 버튼이 있으면 리모컨이라고 부른다"는 규격이 트레이트이고, TV 리모컨이든 에어컨 리모컨이든 규격만 맞으면 같은 자리에 꽂을 수 있습니다.

### 8.1. 정적 디스패치와 동적 디스패치 (C++: `template` vs `virtual`)

* **정적 디스패치(제네릭)**: C++ 템플릿처럼 컴파일 타임에 타입별로 코드가 구체화됩니다(monomorphization). 런타임 오버헤드 0.
* **동적 디스패치(`dyn Trait`)**: C++의 vtable + vptr과 같은 방식입니다. 런타임에 실제 타입을 찾아 호출합니다.

#### 제네릭(`<T>`) 타입 추론 메커니즘과 결정 시점

Rust 컴파일러는 매우 강력한 **양방향 타입 추론(Bidirectional Type Inference)**을 지원합니다:
* **결정 시점**: 변수 선언 시점에 타입을 명시하지 않아도, 변수가 생성될 때(예: `St::new(20)`) 또는 생성 직후 **최초로 호출하는 메서드의 인자 타입**(예: `vec.push("apple")`)에 의해 `T`가 단 하나로 지연 확정됩니다.
* **단일 정적 타입 보장**: 한 번 결정된 `T`는 해당 변수의 수명이 다할 때까지 **절대 다른 타입으로 바뀔 수 없습니다.**

| 언어 | 타입 결정 시점 | 타입 변경 가능 여부 | 비고 |
| :--- | :--- | :--- | :--- |
| **C++ (`auto`)** | 변수 선언/초기화 시점 | 불가 (정적 타입) | 초기화 표현식의 타입으로 즉시 고정 |
| **Rust (`<T>`)** | 선언뿐 아니라 후속 메서드 호출 문맥까지 보고 확정 | 불가 (정적 타입) | 단형성화(Monomorphization)로 런타임 오버헤드 0 |
| **Python** | 런타임 실행 시점 | 가능 (동적 타입) | 변수에 언제든 다른 타입의 객체 대입 가능 |

```cpp
// C++: 순수 가상 함수 + 템플릿
#include <iostream>

class Drawable {
public:
    virtual ~Drawable() = default;
    virtual void draw() const = 0;
};

class Circle : public Drawable {
public:
    double radius;
    explicit Circle(double r) : radius(r) {}
    void draw() const override {
        std::cout << "Circle r=" << radius << "\n";
    }
};

// 1. 정적 디스패치 (템플릿)
template <typename T>
void render_static(const T& item) { item.draw(); }

// 2. 동적 디스패치 (vtable)
void render_dynamic(const Drawable& item) { item.draw(); }
```

```rust
// Rust: 트레이트
pub trait Drawable {
    fn draw(&self);
}

pub struct Circle {
    pub radius: f64,
}

impl Drawable for Circle {
    fn draw(&self) {
        println!("Circle r={}", self.radius);
    }
}

// 1. 정적 디스패치 (monomorphization)
fn render_static<T: Drawable>(item: &T) {
    item.draw();
}

// 2. 동적 디스패치 (vtable)
fn render_dynamic(item: &dyn Drawable) {
    item.draw();
}

fn main() {
    let c = Circle { radius: 1.5 };
    render_static(&c);   // Circle 전용 코드가 생성됨
    render_dynamic(&c);  // vtable을 통해 호출
}
```

> **C++ 감각으로는** `virtual void draw() const = 0;`이 `fn draw(&self);`가 되고, `: public Drawable`이 `impl Drawable for Circle`이 됩니다. 소멸자 걱정(`virtual ~Drawable()`)은 없습니다. Rust는 `Drop`을 자동으로 처리합니다.

> **주의** `dyn Trait`는 크기가 정해지지 않은 타입이라 항상 참조(`&dyn Trait`)나 `Box<dyn Trait>` 형태로 씁니다. C++에서 `Drawable*`를 쓰던 자리라고 보면 됩니다.

### 8.2. 제네릭 문법 읽는 법 (C++: `template<typename T>`)

`fn render_static<T: Drawable>(item: &T)`는 두 조각으로 나뉩니다.

* **`<T>`**: 호출할 때 정해지는 임의의 타입을 받겠다는 선언. 타입이 들어갈 빈칸입니다.
* **`: Drawable`**: 그 빈칸에 들어올 자격 조건(트레이트 바운드). `Drawable`을 구현한 타입만 올 수 있습니다.

C++ 템플릿은 바운드 없이도 본문에서 `item.draw()`를 써두고, 실제 인스턴스화될 때 오류를 냅니다(덕 타이핑). Rust는 바운드가 없으면 `T`에 `.draw()`가 있는지 알 수 없으므로 **제네릭 정의 단계에서 바로 에러**를 냅니다. C++20 `concepts`의 `template <typename T> requires DrawableConcept<T>`(또는 축약형 `template <DrawableConcept T>`)와 같은 역할입니다.

```cpp
// C++20: concepts와 타입 별칭
#include <concepts>

template <typename T>
concept DrawableConcept = requires(const T& t) { t.draw(); };

template <typename T> requires DrawableConcept<T>
void render_static(const T& item) { item.draw(); }

using Meter = double;   // 타입 별칭
```

```rust
// 매개변수 자리에 바로 쓰는 단축 문법 (8.1의 <T: Drawable> 버전과 의미 동일)
fn render_static_short(item: &impl Drawable) {
    item.draw();
}

type Meter = f64;   // 진짜 타입 별칭
```

`impl Trait`은 매번 `<T: Drawable>`을 쓰기 귀찮을 때 쓰는 단축입니다. C++20의 축약 함수 템플릿(Abbreviated Function Template)인 `void render_static_short(const DrawableConcept auto& item)`과 정확히 같은 역할입니다. `type Meter = f64;`는 C++ `using Meter = double;`과 같은 타입 별칭이고, 제네릭 파라미터와는 다른 개념입니다.

**한 줄 정리**
- 상속 대신 트레이트. "무엇을 할 수 있는가"만 정의한다.
- 제네릭은 컴파일 타임 구체화(빠름), `dyn Trait`은 vtable(유연함).
- C++ 템플릿은 나중에 검사하고, Rust는 바운드로 미리 검사한다.

---

### 8.3. 클로저: C++ 람다와 무엇이 다른가 (C++: 람다 `[&]` · `[=]` · `std::function`)

**한 줄 요약: 클로저는 `|x| x + 1` 문법의 익명 함수이고, 주변 변수를 자동으로 챙깁니다. `Fn`/`FnMut`/`FnOnce` 세 트레이트가 "얼마나 세게 챙겼는가"를 나타냅니다.**

클로저는 주머니에 넣어 다니는 도구입니다. 주변 물건을 빌려서 챙기면 원래 주인에게 그대로 있고, 내 것으로 만들면(`move`) 원래 자리에서 사라집니다. C++ 람다는 이 선택을 `[=]`, `[&]`, `[x]`로 직접 적었고, Rust는 쓰임을 보고 컴파일러가 정합니다.

```cpp
// C++: 캡처를 직접 적는다
#include <functional>
#include <iostream>

int main() {
    int base = 10;
    auto by_value = [base](int x) { return x + base; };       // 값 캡처
    auto by_ref = [&base](int x) { return x + base; };        // 참조 캡처
    std::function<int(int)> erased = by_value;                // 타입 지우기
    std::cout << by_value(5) << erased(5);
}
```

```rust
// Rust: 캡처는 자동, 필요하면 move로 명시
fn main() {
    let base = 10;
    let add = |x| x + base;          // base를 불변으로 빌림
    println!("{}", add(5));          // 15

    let add_owned = move |x| x + base;   // base를 소유(이동)
    println!("{}", add_owned(5));        // 15
}
```

| C++ | Rust | 비고 |
| :--- | :--- | :--- |
| `[=]` / `[&]` | (자동) | 쓰임을 보고 결정 |
| `[x]` | `move` | 소유를 넘김 |
| `std::function<int(int)>` | `Box<dyn Fn(i32) -> i32>` | 동적 디스패치 |
| 템플릿 파라미터 | `F: Fn(i32) -> i32` | 정적 디스패치 |

`Fn` 계열 트레이트는 세 개이고, 포함 관계가 있습니다.

| 트레이트 | 호출 방식 | 캡처 | 구현되는 클로저 |
| :--- | :--- | :--- | :--- |
| `FnOnce` | `self` (한 번) | 소유를 소비 | 모든 클로저 |
| `FnMut` | `&mut self` (여러 번) | 가변 빌림 | 상태를 바꾸지 않는 클로저 + 바꾸는 클로저 |
| `Fn` | `&self` (여러 번) | 불변 빌림 | 주변을 읽기만 하는 클로저 |

관계는 `Fn` ⊂ `FnMut` ⊂ `FnOnce`입니다. `Fn`을 만족하는 클로저는 `FnMut`와 `FnOnce`도 만족합니다. 그래서 함수 바운드는 **가장 느슨한 것**을 요구할수록 많은 클로저를 받습니다.

```rust
fn apply<F: Fn(i32) -> i32>(f: F, v: i32) -> i32 {
    f(v)
}

fn apply_twice<F: FnMut() -> i32>(mut f: F) -> i32 {
    f();          // FnMut이므로 여러 번 부를 수 있다
    f()
}

fn consume<F: FnOnce() -> String>(f: F) -> String {
    f()           // 한 번만 부른다
}
```

상태를 바꾸는 클로저는 `FnMut`가 됩니다.

```rust
let mut count = 0;
let mut inc = || { count += 1; };   // count를 가변으로 빌림 → FnMut

inc();
inc();
println!("{}", count);              // 2
```

> **주의** 클로저가 변수를 빌리는 동안에는 원본을 직접 만질 수 없습니다. 5.2절의 대여 규칙이 클로저에도 그대로 적용됩니다. `inc`를 만든 뒤 `count += 1;`을 쓰면 컴파일 에러입니다.

> **팁** 캡처가 없는 클로저는 `fn` 포인터로 변환됩니다. `let f: fn(i32) -> i32 = |x| x + 1;`이 됩니다. C 함수 포인터를 받는 FFI 자리에 그대로 넣을 수 있습니다.

> **C++ 감각으로는** `std::function`은 힙 할당과 간접 호출이 붙고, Rust의 `F: Fn()`은 제네릭이라 인라인됩니다. 8.1절의 정적/동적 디스패치 구도가 클로저에도 똑같이 적용됩니다. 성능이 중요한 자리에서는 `Box<dyn Fn>` 대신 제네릭 바운드를 쓰세요.

**한 줄 정리**
- 클로저는 `|인자| 본문`. 캡처는 자동, 소유가 필요하면 `move`.
- `Fn` ⊂ `FnMut` ⊂ `FnOnce`. 바운드는 가장 느슨한 것을 고른다.
- 상태를 바꾸면 `FnMut`, 소비하면 `FnOnce`.
- 캡처가 없으면 `fn` 포인터가 된다.

### 8.4. 이터레이터: `for`가 아니라 체인 (C++: `std::ranges` · 반복자 쌍)

**한 줄 요약: `for`는 인덱스 루프가 아니라 이터레이터 순회이고, `map`/`filter`/`collect`로 체인을 만듭니다. C++20 ranges와 같은 발상이지만 Rust에는 처음부터 있었습니다.**

이터레이터는 컨베이어 벨트입니다. 값이 벨트 위로 흐르고, 중간에 작업대(`map`)와 선별기(`filter`)를 놓습니다. 벨트는 게을러서, 끝에서 당길 때(`collect`, `for`) 비로소 돌기 시작합니다. 그래서 중간 단계가 몇 개든 실제 루프는 한 번입니다.

```cpp
// C++20 ranges
#include <ranges>
#include <vector>

std::vector<int> v{1, 2, 3, 4, 5};
auto out = v
    | std::views::filter([](int x) { return x % 2 == 0; })
    | std::views::transform([](int x) { return x * x; });

for (int x : out) std::cout << x << " ";   // 4 16
```

```rust
// Rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let out: Vec<i32> = v.iter()
        .filter(|x| *x % 2 == 0)
        .map(|x| x * x)
        .collect();

    println!("{:?}", out);   // [4, 16]
}
```

자주 쓰는 메서드만 모으면 이렇습니다.

| 메서드 | 역할 | C++ 대응 |
| :--- | :--- | :--- |
| `iter()` | `&T`로 순회 | `cbegin()` / `cend()` |
| `iter_mut()` | `&mut T`로 순회 | `begin()` / `end()` |
| `into_iter()` | 값 소유로 순회 | (이동 후 순회) |
| `map` | 각 값 변환 | `transform` |
| `filter` | 조건으로 선별 | `filter` |
| `collect` | 모아서 컬렉션 생성 | 수동 `push_back` |
| `fold` | 누적 계산 | `accumulate` |
| `sum` / `count` / `max` | 집계 | `accumulate` 응용 |
| `enumerate` | 인덱스 붙이기 | `views::enumerate` (C++23) |
| `zip` | 두 시퀀스 묶기 | `views::zip` (C++23) |
| `take` / `skip` | 앞에서 자르기 / 건너뛰기 | `views::take` / `views::drop` |
| `chain` | 이어 붙이기 | `views::concat` (C++26 예정) |
| `rev` | 역순 | `views::reverse` |
| `position` / `find` | 찾기 | `std::find` |

범위 문법은 `for`와 잘 맞습니다.

```rust
for i in 0..5 {          // 0, 1, 2, 3, 4
    print!("{} ", i);
}

for i in 0..=5 {         // 0부터 5까지 (끝 포함)
    print!("{} ", i);
}

for i in (0..10).step_by(2) {   // 0, 2, 4, 6, 8
    print!("{} ", i);
}
```

인덱스가 필요하면 C++처럼 변수를 따로 두지 않고 `enumerate`를 씁니다.

```rust
let names = vec!["a", "b", "c"];

for (i, name) in names.iter().enumerate() {
    println!("{}: {}", i, name);
}
```

소유권에 따라 세 가지 순회가 갈립니다. 4.5절의 `iter_mut()`이 여기서 설명됩니다.

```rust
let mut v = vec![1, 2, 3];

for x in &v { }          // iter(): &i32, 원본 유지
for x in &mut v {        // iter_mut(): &mut i32, 원본 수정
    *x *= 2;
}
for x in v { }           // into_iter(): i32, 원본 소비(이동)
```

> **주의** `collect()`는 무엇을 만들지 알 수 없으므로 타입을 알려줘야 합니다. `let out: Vec<i32> = ...collect();`처럼 변수 타입을 쓰거나, `collect::<Vec<_>>()`처럼 붙입니다. `HashMap`, `String`, `BTreeSet`도 같은 방식으로 모읍니다.

> **팁** `collect`는 실패를 전파하는 데도 씁니다. `Result`의 이터레이터에서 `collect::<Result<Vec<_>, _>>()`를 하면, 하나라도 `Err`일 때 전체가 `Err`가 됩니다. 11장의 `?` 연산자와 같은 발상입니다.

> **C++ 감각으로는** C++20의 ranges 파이프라인(`std::views::filter | std::views::transform`)과 같지만, Rust의 `.iter().filter().map()` 체인은 1.0부터 표준이었고 컴파일러가 체인 전체를 한 루프로 합칩니다. 또한 4.5절에서 본 `.chunks()`와 `.windows()`는 C++23의 `std::views::chunk`, `std::views::slide`에 대응합니다. `for (int i = 0; i < n; ++i)` 습관은 `for (i, x) in v.iter().enumerate()`로 옮기면 대부분 그대로 읽힙니다.

**한 줄 정리**
- `for`는 이터레이터 순회다. 인덱스 루프가 아니다.
- `map`/`filter`/`collect` 체인은 지연 평가되고 한 번의 루프로 합쳐진다.
- `iter()` / `iter_mut()` / `into_iter()`가 소유권을 결정한다.
- 인덱스가 필요하면 `enumerate`, 집계는 `fold`/`sum`.

---

<div style="break-before: page; page-break-before: always;"></div>

## 9. 표준 트레이트 (C++: 복사/이동 생성자 · `operator==`)

**한 줄 요약: C++의 복사 생성자, 소멸자, 대입 연산자, 비교 연산자가 전부 이름 붙은 트레이트로 정리되어 있습니다.**

C++은 컴파일러가 특수 멤버 함수를 자동 생성합니다. 기본 생성자, 복사 생성자, 이동 생성자, 대입 연산자, 소멸자. `= default`로 명시하기도 하고, 안 쓰면 암묵적으로 생깁니다. Rust는 같은 일을 전부 트레이트로 이름 붙여 노출합니다. 그래서 "이 타입은 복사가 되는가", "이 타입은 출력이 되는가"를 타입 선언만 보고 알 수 있습니다.

### 9.1. `#[derive(...)]` (C++: 컴파일러 생성 복사 · `operator==` · `operator<=>`)

C++20에서 `bool operator==(const Point&) const = default;`나 삼중 비교 연산자(`auto operator<=>(const Point&) const = default;`) 한 줄로 비교 연산자를 자동 생성하는 것처럼, Rust는 `#[derive(...)]`로 표준 트레이트 구현(`PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Clone`, `Debug` 등)을 자동 생성합니다.

또한 속성(Attribute) 중 `#[must_use]`는 C++의 `[[nodiscard]]`와 정확히 같습니다. 반환값을 받지 않고 무시하면 컴파일 경고가 발생하며, 의도적으로 반환값을 버릴 때는 `let _ = func();` 관례를 사용합니다.

```rust
// Eq, PartialEq, Debug, Clone, Copy, Default 구현이 자동 생성됨
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct Point {
    pub x: i32,
    pub y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);              // Debug
    let q = p;                        // Copy라서 p도 그대로 유효
    assert_eq!(p, q);                 // PartialEq
    println!("{:?} {:?}", p, q);
    let origin = Point::default();    // Default
    println!("{:?}", origin);
}
```

### 9.2. 자주 쓰는 트레이트 요약 (C++: `operator<<` · `std::hash`)

C++의 연산자 오버로딩, 소멸자, 인터페이스 상속, 특수 멤버 함수는 Rust에서 모두 **`impl [트레이트] for [타입]`** 문법 하나로 일관되게 구현됩니다 (여기서 `for`는 "~를 위하여"라는 의미입니다).

| 트레이트 | 핵심 구현 메서드 시그니처 | 역할 및 C++ 대응 개념 |
| :--- | :--- | :--- |
| `Drop` | `fn drop(&mut self)` | 스코프 종료 시 자동 해제 (소멸자 `~T()`, 직접 호출 금지, 조기 해제는 `std::mem::drop(x)`) |
| `Default` | `fn default() -> Self` | 인자 없는 기본값 생성 (기본 생성자 `T() = default`, `{}` 초기화) |
| `Display` | `fn fmt(&self, f: &mut Formatter) -> Result` | `println!("{}", x)` 사용자용 일반 출력 포맷 (`operator<<`) |
| `Debug` | `fn fmt(&self, f: &mut Formatter) -> Result` | `println!("{:?}", x)` 개발자 디버그용 출력 포맷 |
| `From<T>` | `fn from(val: T) -> Self` | 타입 변환 생성자 (구현 시 `.into()` 자동 생성) |
| `PartialEq` | `fn eq(&self, other: &Self) -> bool` | `==`, `!=` 동등성 연산자 오버로딩 |
| `Add<Rhs>` | `fn add(self, rhs: Rhs) -> Self::Output` | `+` 덧셈 연산자 오버로딩 (`rhs`는 우변 관례) |
| `Iterator` | `fn next(&mut self) -> Option<Self::Item>` | `for x in obj` 루프 순회 가능하게 함 (`std::ranges` / 반복자) |
| `Deref` | `fn deref(&self) -> &Self::Target` | `*` 연산자 역참조, 자동 타입 강제 (`operator*`, `operator->`) |
| `Fn` / `FnMut` / `FnOnce` | `fn call(...)` 등 | 클로저 호출 방식 분류 (`std::function` / 람다 캡처) |
| `Send` / `Sync` | 마커 트레이트 (메서드 없음) | 스레드 간 소유권 이동 / 참조 공유 안전성 마커 |

`Clone`과 `Copy`의 차이를 명확히 구분하세요:
* **`Copy`**: 스택 메모리의 단순 비트 복사(`memcpy`, C++의 `trivially copyable`에 해당). `#[derive(Copy, Clone)]` 형태로 사용하며, `String`, `Box` 등 힙 자원을 가진 타입이나 `Drop`을 구현한 타입은 구현할 수 없습니다.
* **`Clone`**: `.clone()`을 명시적으로 호출해 수행하는 복제이며, 힙 메모리 재할당을 동반하는 깊은 복사를 수행할 수 있습니다.

### 9.3. `Default`: 기본값 만들기 (C++: 기본 생성자 · `{}` 초기화)

`Default`는 인자 없이 기본 상태를 만드는 표준 방법입니다. 구조체 일부 필드만 바꾸고 나머지는 기본값으로 채우는 `..Default::default()` 문법과 자주 같이 씁니다.

```cpp
// C++: 기본값 + C++20 지정 초기화
#include <iostream>
#include <string>

struct Config {
    int port = 8080;
    std::string host = "localhost";
};

int main() {
    Config defaultConfig;
    Config customConfig{ .port = 9090 };   // port만 바꾸고 나머지는 기본값
    std::cout << customConfig.host << ":" << customConfig.port << "\n";
}
```

```rust
#[derive(Debug)]
pub struct Config {
    pub port: u16,
    pub host: String,
}

impl Default for Config {
    fn default() -> Self {
        Config {
            port: 8080,
            host: String::from("localhost"),
        }
    }
}

fn main() {
    let default_config = Config::default();

    let custom_config = Config {
        port: 9090,
        ..Config::default()   // 나머지는 기본값으로
    };
    println!("{:?}", custom_config);
}
```

> **팁** 설정 구조체에 `Default`를 구현해 두면 테스트가 아주 편해집니다. 테스트마다 필요한 필드만 바꿔서 만들 수 있으니까요.

### 9.4. `Clone`과 `Copy` (C++: 복사 생성자 · `memcpy` 가능 타입)

C++은 대입하면 기본적으로 복사입니다. Rust는 기본이 **이동(move)** 입니다. 이 차이가 처음에 가장 크게 느껴집니다.

```cpp
// C++: 대입은 복사, 이동은 std::move로 명시
#include <iostream>
#include <string>

int main() {
    int x = 42;
    int y = x;              // 복사
    std::cout << x << " " << y << "\n";

    std::string s1 = "hello";
    std::string s2 = s1;    // 암시적 깊은 복사 (move를 쓰면 이동)
    std::cout << s1 << " " << s2 << "\n";
}
```

```rust
// Copy가 구현된 타입 (i32, f64, bool 등)
fn copy_demo() {
    let x: i32 = 42;
    let y = x;                 // 비트 복사
    println!("{} {}", x, y);   // 둘 다 유효
}

// Clone만 있는 타입 (String)
fn clone_demo() {
    let s1 = String::from("hello");
    // let s2 = s1;            // s1이 s2로 move되어 s1 사용 불가
    let s2 = s1.clone();       // 명시적 깊은 복사
    println!("{} {}", s1, s2);
}
```

* **`Copy`**: `let b = a;` 후에도 `a`가 그대로 유효한 타입입니다. 조건은 두 가지입니다. **모든 필드가 `Copy`여야 하고, `Drop`을 구현하면 안 됩니다.** `String`이나 `Vec`을 필드로 가진 타입은 첫 조건에서 자동으로 탈락합니다. `#[derive(Copy)]`가 실패하면 십중팔구 필드 하나가 `Copy`가 아닙니다.
* **`Clone`**: 비용이 있는 복사입니다. `.clone()`을 직접 불러야만 복사됩니다.

> **주의** 빌린 값은 그 빌림이 끝나기 전에 이동할 수 없습니다. `let r = &s; consume(s);`는 `r`이 아직 `s`를 빌리고 있어서 컴파일 에러입니다. 구조체도 같습니다. 한 필드를 꺼내 쓴 뒤 다른 필드를 읽는 것(부분 이동)은 되지만, 부분 이동이 일어난 구조체를 통째로 넘기거나 `Drop`을 구현한 타입에서 일부만 꺼내는 것은 막힙니다.

> **주의** 컴파일러가 "value moved here"라고 하면 십중팔구 `String`, `Vec`, 구조체처럼 힙을 가진 값입니다. `.clone()`으로 넘길 수도 있지만, 정말 복사가 필요한지 먼저 생각해 보세요. 대부분은 참조(`&`)로 넘기면 해결됩니다.

### 9.5. `Drop`: 소멸자 (C++: 소멸자 `~T()`)

C++의 `~ClassName()`에 대응합니다. 스코프를 벗어날 때 컴파일러가 자동 호출합니다. 직접 `x.drop()`을 부를 수는 없고, 조기 해제가 필요하면 `std::mem::drop`(보통 그냥 `drop`)을 씁니다.

```cpp
// C++: RAII 소멸자
#include <iostream>
#include <memory>
#include <string>

struct CustomResource {
    std::string name;
    ~CustomResource() {
        std::cout << "Resource " << name << " freed!\n";
    }
};

int main() {
    auto r1 = std::make_unique<CustomResource>(CustomResource{"A"});
    {
        CustomResource r2{"B"};
    }                    // B 소멸
    r1.reset();          // A 조기 해제
}
```

```rust
struct CustomResource {
    name: String,
}

impl Drop for CustomResource {
    fn drop(&mut self) {
        println!("Resource {} freed!", self.name);
    }
}

fn main() {
    let r1 = CustomResource { name: String::from("A") };
    {
        let _r2 = CustomResource { name: String::from("B") };
    }                    // B의 drop 자동 호출

    // r1.drop();        // 에러! explicit destructor calls not allowed
    drop(r1);            // A의 drop 호출
}
```

> **C++ 감각으로는** `unique_ptr::reset()`이 `drop(x)`이고, 블록을 벗어날 때의 자동 소멸은 그대로입니다. 다른 점은 이동한 객체의 소멸자가 두 번 불릴 걱정이 없다는 것입니다. 이동한 쪽은 아예 사용할 수 없으니까요.

### 9.6. `Deref`: `*` 연산자와 자동 타입 강제 (C++: `operator*` · `operator->`)

`Deref`를 구현하면 `*x`로 안쪽 값을 꺼낼 수 있습니다. 그리고 컴파일러가 `Deref Coercion`을 해줍니다. `&String`을 `&str`을 받는 함수에 넘기면 알아서 변환해 줍니다. C++에서 `std::string`이 `std::string_view`로 암묵 변환되던 것과 비슷하지만, 규칙이 타입 시스템에 명시되어 있습니다.

```cpp
// C++: operator* 오버로딩 + string_view 변환
#include <iostream>
#include <string>
#include <string_view>

template <typename T>
struct MyBox {
    T value;
    explicit MyBox(T x) : value(std::move(x)) {}
    const T& operator*() const { return value; }
};

void print_slice(std::string_view s) {
    std::cout << "Slice: " << s << "\n";
}

int main() {
    MyBox<std::string> b("Hello Deref");
    std::cout << "Value: " << *b << "\n";
    print_slice(*b);   // string -> string_view
}
```

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> { MyBox(x) }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target { &self.0 }
}

fn print_slice(s: &str) {
    println!("Slice: {}", s);
}

fn main() {
    let b = MyBox::new(String::from("Hello Deref"));
    println!("Value: {}", *b);

    // &MyBox<String> -> &String -> &str 로 자동 변환
    print_slice(&b);
}
```

> **팁** Deref Coercion은 편리하지만 남용하면 추적이 어려워집니다. 표준 라이브러리의 `Box`, `Rc`, `String`이 하는 정도를 참고하세요.

### 9.7. `From`과 `Into`: 값 변환 (C++: 변환 생성자 · `operator T()`)

C++의 변환 생성자에 대응합니다. `From<T> for U`를 구현하면 표준 라이브러리의 blanket implementation을 통해 `Into<U> for T`가 자동으로 제공됩니다. `From`은 실패하지 않는 변환을 표현하지만, 의미상 무손실 여부는 도메인 설계자가 보장해야 합니다. 그래서 보통 `From`만 구현하면 됩니다.

```cpp
// C++: 변환 생성자
#include <iostream>

struct Distance {
    double meters;
    Distance(double km) : meters(km * 1000.0) {}
};

int main() {
    Distance d1(2.5);     // 직접 생성
    Distance d2 = 3.5;    // 변환 생성자를 통한 암묵 변환
    std::cout << d1.meters << " " << d2.meters << "\n";
}
```

```rust
pub struct Distance {
    pub meters: f64,
}

impl From<f64> for Distance {
    fn from(km: f64) -> Self {
        Distance { meters: km * 1000.0 }
    }
}

fn main() {
    let d1 = Distance::from(2.5);   // From 직접 호출
    let d2: Distance = 3.5.into();  // Into는 자동 생성됨
    println!("{}m {}m", d1.meters, d2.meters);
}
```

> **주의** Rust는 암묵 변환을 하지 않습니다. C++처럼 `Distance d2 = 3.5;`가 조용히 되는 일은 없고, `.into()`를 직접 써야 합니다. 대신 어디서 변환이 일어났는지 코드에서 바로 보입니다.

### 9.8. `Send`와 `Sync`: 마커 트레이트 (C++: 스레드 안전성 계약)

메서드가 하나도 없는 마커 트레이트입니다. "이 타입은 스레드 사이에서 안전하다"는 증명 딱지만 붙이는 역할입니다.

* **`Send`**: 소유권을 다른 스레드로 넘겨도 안전합니다. `Rc`는 단일 스레드 전용이라 `Send`가 아닙니다.
* **`Sync`**: 여러 스레드가 동시에 `&T`로 접근해도 안전합니다. `T`가 `Sync`면 `&T`는 `Send`입니다.

`Arc<T>` 자체가 내부 값을 자동으로 스레드 안전하게 만드는 것은 아닙니다. `Arc<T>`를 스레드 간 공유 가능한 형태로 사용하려면 일반적으로 `T: Send + Sync`가 필요합니다. `Arc<Mutex<T>>`처럼 동기화 타입으로 감싸면 그 타입의 안전성 계약에 따라 공유 가변성을 구현할 수 있습니다. 5.3절에서 본 데이터 경합 차단이 스레드 경계까지 확장된 것입니다.

**한 줄 정리**
- `#[derive(...)]`로 대부분의 표준 트레이트를 공짜로 얻는다.
- 기본은 이동. 암묵 복사가 적절한 타입은 `Copy`, 명시적 복제가 필요한 타입은 `Clone`을 사용합니다. `Clone`이 반드시 깊은 복사인 것은 아닙니다.
- `From`을 구현하면 `Into`가 따라온다. `Drop`은 소멸자, `Send`/`Sync`는 스레드 안전 증명.

---

<div style="break-before: page; page-break-before: always;"></div>

## 10. 모듈 시스템 (C++: 헤더 · `namespace` · `#include`)

**한 줄 요약: C++ `namespace`는 이름 충돌 방지용 가상 공간이고, Rust `mod`는 실제 파일 구조와 1:1로 연결된 서랍장입니다.**

C++에서는 헤더와 소스를 `#include`로 자유롭게 엮습니다. Rust는 그렇지 않습니다. 루트 파일에서 `mod`로 한 번 선언하면, 그 이름의 파일을 컴파일러가 직접 찾아 모듈 트리에 넣습니다. 서랍장을 한 번 조립해 두면, 서랍 이름으로 물건을 꺼내는 방식입니다.

### 10.1. 모듈 정의와 파일 분리 (C++: 헤더/소스 분리 · C++20 모듈)

`mod 이름 { ... }`으로 파일 안에 인라인 모듈을 쓸 수도 있고, `mod 이름;`처럼 세미콜론만 적으면 파일 시스템에서 찾습니다.

```text
src/
├── main.rs            # 크레이트 루트: mod network; 선언
├── network.rs         # network 모듈 본문: pub mod http; 선언
└── network/
    └── http.rs        # network::http 모듈 본문
```

모듈 파일은 `src/network.rs` + `src/network/http.rs` 형태 또는 `src/network/mod.rs` 형태를 사용할 수 있습니다. `mod.rs` 방식도 현재 유효하며, 어느 쪽을 택할지는 프로젝트의 일관성과 모듈 구조에 맞추면 됩니다. 파일명이 겹치지 않아서 찾기 쉽기 때문입니다.

중요한 차이가 하나 있습니다. C++은 각 소스 파일이 필요한 헤더를 `#include`합니다. Rust는 **루트(`main.rs` 또는 `lib.rs`)에서 `mod`로 한 번만 선언**하면, 프로젝트 어디서든 경로로 참조할 수 있습니다.

> **C++ 감각으로는** C++20에서 `#include` 텍스트 복붙의 한계를 넘기 위해 모듈(`export module` / `import`)을 도입하고 C++23에서 표준 라이브러리 모듈(`import std;`)을 추가한 것과 달리, Rust는 애초에 헤더 파일 없이 언어 차원의 모듈 시스템을 기본으로 설계했습니다.

### 10.2. 경로와 `use` (C++: `namespace` · `using`)

* **`crate::`**: 크레이트 루트에서 시작하는 절대 경로 (C++ `::` 기준 참조)
* **`super::`**: 부모 모듈 기준 상대 경로 (파일 시스템 `../`)
* **`use`**: 경로를 현재 스코프로 가져오는 단축 (C++ `using`)
* **`pub use`**: 재수출. 내부 계층을 숨기고 최상위 경로로 API를 노출

```cpp
// C++: namespace, 내부 은닉, using
#include <iostream>
#include <string_view>

namespace network {
    inline void log_prefix() { std::cout << "[NET] "; }

    namespace http {
        namespace {
            void internal_helper() { /* 파일 내부 전용 */ }
        }

        void get(std::string_view url) {
            network::log_prefix();
            std::cout << "GET " << url << "\n";
        }
    }

    using http::get;   // 재노출
}

int main() {
    ::network::http::get("https://example.com");

    using network::http::get;
    get("https://rust-lang.org");

    network::get("https://cppreference.com");
}
```

```rust
mod network {
    // pub이 없으므로 network 밖에서는 보이지 않는다
    fn log_prefix() {
        print!("[NET] ");
    }

    pub mod http {
        fn internal_helper() {
            // http 모듈 내부 전용
        }

        pub fn get(url: &str) {
            super::log_prefix();   // 부모 모듈의 비공개 함수에 접근
            println!("GET {}", url);
        }
    }

    pub use http::get;   // 재수출: 외부에서 network::get 으로 접근
}

fn main() {
    // 1) 절대 경로
    crate::network::http::get("https://example.com");

    // 2) use로 가져와 짧게
    use crate::network::http;
    http::get("https://rust-lang.org");

    // 3) 재수출된 짧은 경로
    network::get("https://cppreference.com");
}
```

> **주의** Rust의 접근 제어는 클래스가 아니라 **모듈** 단위입니다. 구조체 필드에도 `pub`을 붙여야 밖에서 보입니다. `pub struct Config { pub port: u16 }`처럼 필드마다 명시해야 하는 게 C++ `private:`와 다른 점입니다.

`pub`의 범위는 더 좁힐 수 있습니다.

* **`pub(crate)`**: 이 크레이트 안에서만 공개
* **`pub(super)`**: 부모 모듈까지만 공개

### 10.3. 외부 크레이트 사용하기 (C++: `find_package` · `vcpkg` · `FetchContent`)

지금까지는 같은 크레이트 안의 모듈이었습니다. 별도 프로젝트를 가져오려면 `Cargo.toml`에 의존성을 적고, 크레이트 이름으로 경로를 시작합니다.

| 연결 방식 | `Cargo.toml` | C++ 대응 |
| :--- | :--- | :--- |
| 로컬 경로 | `math_utils = { path = "../math_utils" }` | `add_subdirectory()` |
| Git 저장소 | `math_utils = { git = "https://github.com/org/math_utils", tag = "v0.1.0" }` | `FetchContent` / Submodule |
| crates.io | `serde = { version = "1.0", features = ["derive"] }` | `vcpkg` / `Conan` + `find_package` |
| Workspace | 루트 `[workspace]` + `math_utils = { workspace = true }` | 멀티 타겟 루트 `CMakeLists.txt` |

로컬 경로로 연결하는 예를 봅시다. 라이브러리 크레이트와, 그걸 쓰는 바이너리 크레이트입니다.

```text
workspace/
├── math_utils/                 # 라이브러리 크레이트
│   ├── Cargo.toml              # name = "math_utils"
│   └── src/
│       ├── lib.rs              # pub mod geometry;
│       └── geometry.rs         # pub fn distance(...)
└── app/                        # 바이너리 크레이트
    ├── Cargo.toml              # [dependencies] math_utils = { path = "../math_utils" }
    └── src/
        └── main.rs             # use math_utils::geometry;
```

```cmake
# C++ (app/CMakeLists.txt): 외부 프로젝트 연결과 링크
add_subdirectory(../math_utils math_utils_build)
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE math_utils)
```

```cpp
// C++ (app/src/main.cpp): 헤더 포함 후 네임스페이스 참조
#include <iostream>
#include "math_utils/geometry.hpp"

int main() {
    double d = math_utils::geometry::distance(0.0, 0.0, 3.0, 4.0);
    std::cout << "Distance: " << d << "\n";
}
```

```toml
# Rust (app/Cargo.toml): 의존성 등록
[package]
name = "app"
version = "0.1.0"
edition = "2021"

[dependencies]
math_utils = { path = "../math_utils" }
# 원격 Git 저장소라면:
# math_utils = { git = "https://github.com/example/math_utils.git", branch = "main" }
```

```rust
// Rust (math_utils/src/lib.rs): 외부에 공개할 모듈 선언
pub mod geometry;
```

```rust
// Rust (app/src/main.rs): 외부 크레이트의 모듈 참조
use math_utils::geometry;

fn main() {
    // use로 가져온 모듈을 통해 호출
    let d1 = geometry::distance(0.0, 0.0, 3.0, 4.0);

    // 크레이트 이름부터 전체 경로로 직접 호출
    let d2 = math_utils::geometry::distance(1.0, 1.0, 4.0, 5.0);

    println!("Distance: {}, {}", d1, d2);
}
```

주의할 점 두 가지입니다.

* **`src/lib.rs`가 있어야 합니다.** 외부 프로젝트가 `src/main.rs`만 있는 순수 바이너리라면 다른 프로젝트에서 참조할 수 없습니다. 공개할 모듈은 반드시 `src/lib.rs`에 `pub mod ...;`로 노출해야 합니다.
* **하이픈은 언더스코어로 바뀝니다.** `Cargo.toml`에 `math-utils = { path = "../math-utils" }`처럼 하이픈을 썼다면, 코드에서는 `use math_utils::geometry;`처럼 언더스코어를 씁니다. 식별자 규칙 때문입니다.

> **C++ 감각으로는** `target_link_libraries`와 `#include` 경로 설정을 Cargo가 대신해 준다고 보면 됩니다. 헤더 파일이 없다는 점만 다릅니다.

**한 줄 정리**
- `mod`는 실제 파일 구조와 연결된다. 루트에서 한 번 선언하면 끝.
- 접근 제어는 모듈 단위. 필드도 `pub`을 붙여야 보인다.
- 외부 크레이트는 `Cargo.toml`에 적고 `use 크레이트명::...`으로 쓴다.

---

<div style="break-before: page; page-break-before: always;"></div>

## 11. 에러 처리 (C++: 예외 · `std::error_code` · `std::expected`)

**한 줄 요약: C++은 예외를 던져서 위로 보내고, Rust는 `Result`라는 반환값으로 돌려줍니다.**

C++에서 `throw`는 함수의 정상 흐름을 끊고 스택을 되감습니다. 편리하지만 함수 시그니처만 봐서는 이 함수가 예외를 던지는지 알 수 없습니다. Rust에는 `try`/`catch`가 없고, 복구 가능한 에러는 전부 `Result<T, E>` 값으로 돌아옵니다(C++23에서 도입된 `std::expected<T, E>`가 이에 해당합니다). 택배 송장이라고 생각하세요. 성공이면 물건(`Ok`), 실패면 반품 사유(`Err`)가 같이 옵니다. 송장을 안 열어볼 수는 없습니다.

| 항목 | C++ 예외 | Rust `Result<T, E>` |
| :--- | :--- | :--- |
| 전달 방식 | `throw`로 제어 흐름 점프 | `Ok(T)` / `Err(E)` 반환값 |
| 시그니처 | 함수 선언으로 예외 여부를 알기 어려움 | 반환 타입에 에러 타입이 명시됨 |
| 미처리 방지 | 호출 스택 최상위까지 예외가 처리되지 않으면 `std::terminate` | `#[must_use]` 경고. `match`, `?` 등으로 처리 |
| 런타임 비용 | 예외 경로에서 스택 되감기 등 | 일반적으로 값 기반 분기. 최적화와 에러 타입에 따라 비용은 달라짐 |
| 복구 불가 | `std::terminate()` / `std::abort()` | `panic!` (기본 unwind는 해당 스레드를 unwind하며, main 스레드의 미처리 panic은 프로세스를 종료시킴. `panic = "abort"`는 즉시 abort) |

### 11.1. `try`/`catch` 대신 `Result` + `match` (C++: 예외 · `std::error_code`)

`match`는 C++ `switch`와 닮았지만 결정적인 차이가 있습니다. **모든 경우를 처리했는지 컴파일러가 검사합니다.** `Ok`와 `Err` 중 하나를 빼먹으면 컴파일이 안 됩니다. 다만 `Result`를 반드시 `match`로 처리해야 하는 것은 아닙니다. `?`, `map`, `unwrap_or` 등을 사용할 수 있고, 값을 의도적으로 버릴 수도 있습니다(이 경우 `must_use` 경고를 확인해야 합니다).

```cpp
// C++: throw + try/catch
#include <iostream>
#include <stdexcept>

double divide(double a, double b) {
    if (b == 0.0) {
        throw std::invalid_argument("0으로 나눌 수 없습니다.");
    }
    return a / b;
}

int main() {
    try {
        double result = divide(10.0, 0.0);
        std::cout << "결과: " << result << "\n";
    } catch (const std::invalid_argument& e) {
        std::cout << "에러 발생: " << e.what() << "\n";
    }
}
```

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        return Err(String::from("0으로 나눌 수 없습니다."));
    }
    Ok(a / b)
}

fn main() {
    match divide(10.0, 0.0) {
        Ok(result) => println!("결과: {}", result),
        Err(e) => println!("에러 발생: {}", e),
    }
}
```

`divide`의 시그니처만 봐도 "이 함수는 실패할 수 있고, 실패하면 `String`을 준다"는 걸 알 수 있습니다. C++에서 주석으로 "이 함수는 예외를 던집니다"라고 쓰던 일이 타입에 들어온 셈입니다.

### 11.2. `match` 패턴과 `?` 연산자 (C++: `try` 블록 · 조기 반환)

`match`는 `Ok`/`Err` 분기만 하는 게 아니라 값 바인딩, 범위, 조건 가드, OR 패턴까지 지원하는 표현식입니다.

```cpp
// C++: switch
#include <iostream>

const char* describe_code(int code) {
    switch (code) {
        case 200: return "성공";
        case 403:
        case 404: return "접근 불가 또는 없음";
        default:  return "기타 상태";
    }
}
```

```rust
// Rust: match (OR 패턴, 범위, 가드, 와일드카드)
fn describe_code(code: i32) -> &'static str {
    match code {
        200 => "성공",
        403 | 404 => "접근 불가 또는 없음",
        500..=599 => "서버 내부 오류",
        n if n < 0 => "유효하지 않은 음수 코드",
        _ => "기타 상태",
    }
}
```

에러를 위로 넘기기만 할 때는 매번 `match`를 쓰지 않고 `?` 연산자를 씁니다. `Err`면 즉시 `Err`를 반환하고, `Ok`면 값을 꺼냅니다. C++에서 예외가 자동으로 전파되던 자리인데, Rust는 전파 지점이 `?`로 눈에 보입니다.

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        return Err(String::from("0으로 나눌 수 없습니다."));
    }
    Ok(a / b)
}

fn half_divide(a: f64, b: f64) -> Result<f64, String> {
    let val = divide(a, b)?;   // Err면 여기서 즉시 반환
    Ok(val / 2.0)
}

fn main() {
    // 성공 케이스만 처리하고 싶으면 if let
    if let Ok(v) = divide(10.0, 2.0) {
        println!("절반: {}", v / 2.0);
    }

    println!("{:?}", half_divide(10.0, 0.0));
}
```

> **팁** `?`를 쓰려면 함수의 반환 타입이 `Result`(또는 `Option`)여야 합니다. `main`에서 바로 쓰고 싶으면 `fn main() -> Result<(), Box<dyn std::error::Error>>` 형태를 쓰면 됩니다.

`?` 외에도 `Result<T, E>`(와 `Option<T>`)는 값을 꺼내지 않고 연쇄 변환하는 **모나딕 체이닝** 메서드 `.and_then()`, `.map()`을 제공합니다. C++23의 `std::expected<T, E>`(및 `std::optional<T>`)에서도 `.and_then()`과 `.transform()`으로 같은 패턴을 쓸 수 있습니다.

```cpp
// C++23: std::expected + 모나딕 체이닝
#include <expected>
#include <string>

std::expected<double, std::string> divide(double a, double b);

std::expected<double, std::string> scaled_half(double a, double b) {
    return divide(a, b)
        .and_then([](double v) -> std::expected<double, std::string> {
            return v / 2.0;
        })
        .transform([](double v) { return v * 10.0; });
}
```

```rust
// Rust: .and_then()과 .map()으로 연쇄 처리
fn scaled_half(a: f64, b: f64) -> Result<f64, String> {
    divide(a, b)
        .and_then(|v| Ok(v / 2.0))
        .map(|v| v * 10.0)
}
```

> **주의** 정말 복구할 수 없는 상황(배열 범위 초과, `unwrap()` 실패 등)은 `panic!`입니다. C++의 `std::terminate`에 가깝습니다. 라이브러리 코드에서는 `panic!` 대신 `Result`를 반환하는 게 예의입니다.

**한 줄 정리**
- 복구 가능한 에러는 `Result<T, E>`. 시그니처에 드러난다.
- `match`는 빠짐없이 처리해야 컴파일된다. 전파는 `?`.
- `panic!`은 최후의 수단, `Result`가 기본.

---

<div style="break-before: page; page-break-before: always;"></div>

## 12. 마무리: C++ 개발자가 Rust로 넘어갈 때 (C++: 이동 의미론 · `const` 정확성)

Rust에서 가장 중요한 태도는 **컴파일러와 싸우지 않기**입니다. C++에서는 컴파일러가 허용하는 위험을 개발자가 머릿속 규칙으로 막았습니다. Rust에서는 그 검증을 컴파일러에게 넘겼습니다. 에러 메시지는 방해가 아니라 조언입니다. 실제로 Rust 컴파일러는 "왜 안 되는지"와 "이렇게 고치면 된다"를 꽤 구체적으로 알려줍니다.

1. **대여 검사기 에러를 만나면 구조를 의심하세요.** 에러를 우회하려 하지 말고, 소유자가 누구인지, 참조가 얼마나 오래 살아야 하는지 다시 보세요. 대부분은 참조 범위를 좁히거나 `clone` 대신 빌림으로 바꾸면 풀립니다.
2. **표준 트레이트를 먼저 익히세요.** `Default`, `Clone`, `Copy`, `Drop`, `Display`, `Debug`, `Deref`, `From`/`Into`. 이걸 구현해 두면 표준 라이브러리와 생태계 크레이트가 공짜로 호환됩니다. `#[derive(...)]` 한 줄로 해결되는 일이 생각보다 많습니다.
3. **자료 구조를 다시 설계하세요.** 서로를 가리키는 포인터, 복잡한 그래프는 소유권 모델과 잘 안 맞습니다. 인덱스 기반 참조(`Vec`의 인덱스로 가리키기)나 `Arc<Mutex<T>>`로 바꾸는 게 보통 정답입니다.
4. **컴파일러 진단을 읽으세요.** `rustc --explain E0382` 같은 명령은 해당 에러의 원리와 예시를 문서로 보여줍니다. 에러 코드를 검색하는 습관이 초반 학습 속도를 크게 올립니다.

처음 일주일은 느리게 느껴질 겁니다. C++에서 10분이면 쓰던 코드가 1시간 걸리기도 합니다. 그 시간은 사라지는 게 아니라, 나중에 디버거를 붙잡고 있던 시간으로 바뀌어 돌아옵니다. 소유권 규칙이 손에 붙는 순간부터는 오히려 "이 코드는 왜 이렇게 짰지"를 컴파일러가 먼저 물어봐 주는 느낌이 듭니다. 그때부터 Rust가 편해집니다.

### 모던 C++(C++17/20/23)과 Rust 주요 대응 한눈에 보기

| 개념 / 기능 | Rust 표현식 | 모던 C++ 대응 기능 | 도입 표준 |
| :--- | :--- | :--- | :--- |
| 문자열 뷰 (대여) | `&str` | `std::string_view` | C++17 |
| 합타입 (태그된 공용체) | `enum` + `match` | `std::variant` + `std::visit` | C++17 |
| 널 대체 (값의 유무) | `Option<T>` | `std::optional<T>` | C++17 |
| 구조 분해 | `let (a, b) = pair;` | `auto [a, b] = pair;` | C++17 |
| 인터페이스 제약 | `T: Trait` / `impl Trait` | `requires Concept<T>` / `Concept auto` | C++20 |
| 비교 연산자 자동 생성 | `#[derive(PartialEq, PartialOrd)]` | `operator<=>() = default` | C++20 |
| 모듈 시스템 | `mod` / `use` | `export module` / `import` (`import std;`) | C++20 / 23 |
| 컬렉션 파이프라인 | `.iter().filter().map()` | `std::views::filter \| transform` | C++20 |
| 순회 유틸리티 | `.enumerate()`, `.zip()`, `.chunks()`, `.windows()` | `views::enumerate`, `zip`, `chunk`, `slide` | C++23 |
| 에러 반환 타입 | `Result<T, E>` | `std::expected<T, E>` | C++23 |
| 모나딕 체이닝 | `.and_then()`, `.map()` (`Option`/`Result`) | `.and_then()`, `.transform()` (`optional`/`expected`) | C++23 |
| 명시적 수신자 | `fn foo(&mut self)` | `void foo(this Self& self)` | C++23 |
| 포맷 출력 | `println!("{}", x)` | `std::println("{}", x)` | C++23 |

---

## 13. 테스트와 품질 도구 (C++: CTest · GoogleTest · 정적 분석)

**핵심: Rust는 테스트를 언어와 Cargo 워크플로에 기본 통합합니다.**

C++에서 GoogleTest와 CTest를 조합하듯 Rust에서는 `#[test]`, `cargo test`, `clippy`, `rustfmt`를 기본 도구로 사용합니다.

### 13.1. 단위 테스트와 통합 테스트

- **단위 테스트**: 보통 테스트 대상 모듈 안의 `#[cfg(test)] mod tests`에 둡니다. private 항목도 테스트할 수 있습니다.
- **통합 테스트**: `tests/` 디렉터리에 별도 파일로 둡니다. 외부에서 사용하는 것처럼 공개 API를 검증합니다.
- **문서 테스트**: 문서 주석의 코드 블록을 실제 테스트로 실행합니다.

```rust
// src/lib.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    #[should_panic(expected = "invalid")]
    fn demonstrates_expected_panic() {
        panic!("invalid input");
    }
}
```

```bash
cargo test
cargo test adds_two_numbers
cargo test --doc
```

### 13.2. 통합 테스트

```text
my_project/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    └── api_test.rs
```

```rust
// tests/api_test.rs
use my_project::add;

#[test]
fn public_api_adds() {
    assert_eq!(add(10, 20), 30);
}
```

통합 테스트는 라이브러리의 공개 API만 사용할 수 있습니다. 따라서 외부 사용자의 관점에서 API가 올바른지 확인하기 좋습니다.

### 13.3. 개발 기본 도구

| 명령 | 역할 | C++에서 가까운 도구 |
|---|---|---|
| `cargo fmt` | 코드 포맷팅 | clang-format |
| `cargo clippy` | 린트와 관용구 진단 | clang-tidy 일부 |
| `cargo test` | 테스트 실행 | CTest / GoogleTest |
| `cargo doc --open` | 문서 생성 및 열기 | Doxygen 일부 |
| `cargo check` | 빠른 컴파일 검사 | IDE 진단 / 구문·타입 검사 |
| `cargo audit` | 의존성 취약점 점검(별도 설치) | 의존성 보안 스캐너 |

CI에서는 다음과 같은 순서를 기본으로 둘 수 있습니다.

```bash
cargo fmt --all -- --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --all
cargo build --release
```

프로젝트에 따라 모든 feature 조합을 동시에 빌드하는 것이 불가능할 수 있으므로 CI 매트릭스를 별도로 설계해야 합니다.

---

## 14. 동시성 프로그래밍 (C++: `std::thread` · `mutex` · atomic)

**핵심: Rust는 스레드 간 이동과 공유 가능성을 `Send`와 `Sync`로 표현하고, 동기화 방식은 타입으로 드러냅니다.**

### 14.1. 스레드 생성과 소유권 이동

```cpp
// C++: thread에 값을 전달
std::thread t([value = 42] {
    std::cout << value << "\n";
});
t.join();
```

```rust
use std::thread;

fn main() {
    let value = 42;

    let handle = thread::spawn(move || {
        println!("{value}");
    });

    handle.join().expect("thread panicked");
}
```

`move` 클로저는 캡처한 값을 클로저 안으로 이동시킵니다. 스레드는 호출한 함수보다 오래 실행될 수 있으므로, `thread::spawn`은 스레드가 소유할 수 있는 `'static` 수명 조건을 요구합니다. 여기서 `'static`은 값이 반드시 전역 변수라는 뜻이 아니라, 빌린 데이터가 스레드보다 먼저 사라지지 않아야 한다는 타입 제약입니다.

### 14.2. 공유 가변성: `Arc<Mutex<T>>`

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..4 {
        let counter = Arc::clone(&counter);

        handles.push(thread::spawn(move || {
            let mut value = counter.lock().expect("mutex poisoned");
            *value += 1;
        }));
    }

    for handle in handles {
        handle.join().expect("thread panicked");
    }

    println!("{}", *counter.lock().unwrap());
}
```

- `Arc<T>`: 여러 소유자가 원자적 참조 카운트로 하나의 값을 공유합니다.
- `Mutex<T>`: 한 번에 하나의 스레드만 내부 값에 접근하도록 잠급니다.
- `lock()`은 guard를 반환합니다. guard가 살아 있는 동안 잠금이 유지되고, guard가 drop되면 잠금이 해제됩니다.

`Arc`만 사용한다고 내부 데이터의 동시 수정이 안전해지는 것은 아닙니다. 내부 가변성이 필요하면 `Mutex`, `RwLock`, atomic 타입 등 적절한 동기화 도구를 함께 사용해야 합니다.

### 14.3. 채널을 이용한 메시지 전달

공유 메모리를 직접 수정하는 대신 메시지를 보내는 방식도 있습니다.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(String::from("hello")).unwrap();
    });

    let message = rx.recv().unwrap();
    println!("{message}");
}
```

`mpsc`는 multiple producer, single consumer 채널입니다. 메시지 전달은 데이터 소유권을 함께 이동시키는 구조를 만들기 쉬워 공유 가변성을 줄이는 데 유용합니다.

### 14.4. C++과의 비교에서 기억할 점

| C++ | Rust | 차이 |
|---|---|---|
| `std::thread` | `std::thread::spawn` | 스레드 핸들 개념은 유사 |
| `std::mutex` | `Mutex<T>` | 잠금 guard의 RAII 해제 |
| `std::shared_ptr<T>` | `Arc<T>` | 원자적 참조 카운트 |
| `std::atomic<T>` | `Atomic*` 타입 | 원자적 메모리 연산 |
| 큐/메시지 라이브러리 | `mpsc` 등 | 소유권 이동과 결합 |

---

## 15. 비동기 프로그래밍 (C++: `std::future` · coroutine)

**핵심: Rust의 `async` 함수는 Future를 반환하며, 실제 실행과 스케줄링은 executor가 담당합니다.**

Rust 표준 라이브러리는 Future 트레이트와 async/await 문법을 제공하지만, 범용 비동기 런타임을 기본 제공하지는 않습니다. 네트워크 서버나 타이머 등을 만들려면 Tokio, async-std 등의 런타임을 선택하는 경우가 많습니다.

### 15.1. `async`와 `await`

```rust
async fn fetch_value() -> u32 {
    42
}
```

이 함수는 호출 즉시 `u32`를 반환하는 것이 아니라, 나중에 완료될 작업을 나타내는 `Future<Output = u32>`를 반환합니다.

```rust
// Tokio 예시: Cargo.toml에 tokio 의존성이 필요
#[tokio::main]
async fn main() {
    let value = fetch_value().await;
    println!("{value}");
}
```

### 15.2. C++ coroutine과의 차이

| 항목 | C++20 coroutine | Rust async |
|---|---|---|
| 지연 실행 | coroutine handle 및 호출 방식에 따라 다름 | Future는 poll되기 전까지 실행되지 않음 |
| 스케줄링 | 표준 언어 기능만으로 executor 제공 안 함 | 표준 Future만으로 executor 제공 안 함 |
| 대기 문법 | `co_await` | `.await` |
| 작업 표현 | promise/awaiter 프로토콜 | `Future` 트레이트 |
| 런타임 | 별도 구현 필요 | Tokio 등 선택 |

### 15.3. 실무에서 주의할 점

- `.await`는 스레드를 무조건 새로 만드는 기능이 아닙니다.
- CPU 집약 작업을 async 함수 안에서 오래 실행하면 executor의 다른 작업을 막을 수 있습니다.
- `std::thread::spawn`과 async task는 실행 모델이 다릅니다.
- 비동기 함수가 참조를 보유하면 Future의 수명과 `Send` 요구 조건에 영향을 줄 수 있습니다.
- 런타임별 API와 기능이 다르므로 프로젝트 초기에 런타임을 선택하는 것이 좋습니다.

---

## 16. C/C++ 연동과 `unsafe` (C ABI · 포인터 · FFI)

**핵심: Rust는 C ABI를 통해 기존 C/C++ 시스템과 연결할 수 있지만, FFI 경계는 안전성 계약을 직접 관리해야 합니다.**

### 16.1. C ABI로 함수 내보내기

```rust
// Rust 라이브러리
#[unsafe(no_mangle)]
pub extern "C" fn rust_add(a: i32, b: i32) -> i32 {
    a + b
}
```

`extern "C"`는 C ABI 호출 규약을 사용한다는 뜻입니다. `no_mangle`은 심볼 이름을 지정한 이름으로 내보내기 위한 속성입니다. Rust 2024 Edition에서는 일부 unsafe 속성을 `#[unsafe(...)]` 형태로 명시합니다.

C/C++에서는 헤더 선언을 통해 호출할 수 있습니다.

```cpp
extern "C" int rust_add(int a, int b);

int main() {
    return rust_add(10, 20);
}
```

실제 프로젝트에서는 `Cargo.toml`의 `crate-type`을 `staticlib`, `cdylib` 등으로 설정하고 플랫폼별 링크 설정을 해야 합니다.

### 16.2. `unsafe`가 의미하는 것

`unsafe`는 "이 코드는 안전하다"는 인증이 아닙니다. 컴파일러가 자동으로 검증할 수 없는 일부 작업을 개발자가 책임지고 수행하겠다는 경계입니다.

대표적인 작업은 다음과 같습니다.

- raw pointer 역참조
- `unsafe` 함수 호출
- `unsafe` trait 구현
- union 필드 접근
- 외부 메모리 및 ABI 계약 처리

```rust
fn main() {
    let value = 10;
    let ptr: *const i32 = &value;

    // 포인터가 유효하고 정렬되어 있으며 수명이 유지된다는 전제가 필요
    let copied = unsafe { *ptr };
    println!("{copied}");
}
```

`unsafe` 블록 안에 있다고 해서 모든 검증이 사라지는 것은 아닙니다. 컴파일러는 여전히 일반적인 타입 검사 등을 수행하지만, 포인터 유효성이나 aliasing 같은 일부 불변식은 작성자가 보장해야 합니다.

### 16.3. FFI 경계 설계 원칙

1. C ABI에서 표현 가능한 타입을 우선 사용합니다.
2. Rust의 `String`, `Vec<T>`, trait object를 C 구조체처럼 직접 노출하지 않습니다.
3. 메모리를 할당한 쪽과 해제하는 쪽을 명확히 정합니다.
4. 포인터와 길이를 함께 전달하고, null 및 길이 검증을 수행합니다.
5. Rust panic이 C ABI 경계를 넘어 전파되지 않도록 설계합니다.
6. C++ 객체를 직접 ABI로 노출하기보다 C 인터페이스 래퍼를 두는 방식을 우선 검토합니다.

C++은 C ABI와 동일한 ABI를 보장하지 않습니다. C++과 Rust의 직접 연동은 별도 ABI 솔루션(cxx 등)을 검토하거나 C 호환 래퍼를 경계로 두는 편이 일반적으로 안전합니다.

---

## 17. 실무 학습 로드맵 (C++ 개발자의 전환 순서)

기존 C++ 경험을 활용하되, 문법보다 소유권과 API 설계를 먼저 익히는 편이 효율적입니다.

### 17.1. 학습 순서

| 단계 | 학습 주제 | 확인할 내용 |
|---|---|---|
| 1 | Cargo와 기본 문법 | 프로젝트 생성, 모듈, 타입 추론 |
| 2 | 소유권과 대여 | move, borrow, mutable borrow |
| 3 | 수명과 참조 설계 | 함수 인자·반환값·구조체의 lifetime |
| 4 | enum과 패턴 매칭 | `Option`, `Result`, `match` |
| 5 | 제네릭과 트레이트 | trait bounds, `impl Trait`, `dyn Trait` |
| 6 | 컬렉션과 이터레이터 | `Vec`, `HashMap`, iterator adapter |
| 7 | 에러와 테스트 | `Result`, `?`, 단위·통합 테스트 |
| 8 | 동시성과 비동기 | `Arc`, `Mutex`, channel, Future |
| 9 | FFI와 unsafe | C ABI, raw pointer, 안전성 계약 |

### 17.2. C++ 개발자가 특히 바꿔야 할 습관

- `T*`를 먼저 떠올리기보다 값 소유자와 참조 관계를 먼저 설계합니다.
- `shared_ptr`를 기본 선택으로 두지 말고, 소유권을 단순화할 수 있는지 먼저 검토합니다.
- 컴파일 에러를 `clone()`으로 무조건 해결하지 않습니다. 복사 비용과 소유권 모델을 먼저 확인합니다.
- 클래스 상속 계층을 그대로 옮기기보다 `struct`와 trait 조합으로 모델링합니다.
- 예외를 던지는 설계 대신 `Result`로 실패 가능성을 API에 표현합니다.
- `unsafe`는 편의를 위한 우회로가 아니라 명확한 불변식을 구현하는 좁은 경계로 제한합니다.

### 17.3. 첫 번째 연습 프로젝트

작은 CLI 프로그램을 권장합니다.

- 입력: JSON 또는 CSV 파일
- 처리: 구조체, enum, iterator, 에러 전파
- 출력: 집계 결과와 파일 저장
- 품질: 단위 테스트, 통합 테스트, Clippy
- 확장: 병렬 처리 또는 비동기 네트워크 호출

이 정도 프로젝트를 소유권을 무리하게 복제하지 않고 구현할 수 있다면, Rust의 기본 설계 감각을 갖추기 시작한 것입니다.

---

## 부록 A. 기존 내용에서 바로잡은 핵심 개념

| 항목 | 정확한 이해 |
|---|---|
| `From` | 실패하지 않는 변환 계약. 모든 경우에 수학적 무손실을 뜻하지 않음 |
| `Cell<T>` | 모든 타입을 감쌀 수 있음. `get()`은 `T: Copy`일 때 가능 |
| `RefCell<T>` | `T: Send`이면 이동 가능할 수 있지만 `Sync`는 아님 |
| `Arc<T>` | 참조 카운트의 스레드 안전성과 내부 데이터의 스레드 안전성은 별개 |
| `Result<T, E>` | `#[must_use]` 경고 대상. `match` 외에도 `?`, `map`, `unwrap_or` 등으로 처리 |
| `Copy` | C++의 POD나 trivially copyable과 동일하지 않음 |
| `Clone` | 명시적 복제지만 깊은 복사를 보장하지 않음 |
| 정수 오버플로 | 프로파일 설정에 따라 검사 또는 래핑. 프로젝트 설정 확인 필요 |
| `&[T]` | 일반적으로 주소와 길이를 담지만 구체적 ABI 표현에 의존하지 말 것 |
| `cargo check` | 최종 링크는 하지 않지만 build script/proc macro/의존성 빌드가 수행될 수 있음 |

## 부록 B. 참고할 공식 문서

- The Rust Programming Language: https://doc.rust-lang.org/book/
- Rust Reference: https://doc.rust-lang.org/reference/
- The Rust Edition Guide: https://doc.rust-lang.org/edition-guide/
- The Rust Standard Library: https://doc.rust-lang.org/std/
- The Cargo Book: https://doc.rust-lang.org/cargo/
- The Rustonomicon (`unsafe`와 메모리 모델): https://doc.rust-lang.org/nomicon/
- The Rust API Guidelines: https://rust-lang.github.io/api-guidelines/



---

<div style="break-before: page; page-break-before: always;"></div>

## 부록 C. 🦀 Rust 핵심 문법 & 메모리 치트시트 (C++ / Python 3자 비교)

C++ 및 Python 개발자가 Rust 코드를 작성하거나 리뷰할 때 가장 자주 참조하게 되는 5대 핵심 문법과 메모리 모델 규칙을 일목요연하게 정리한 치트시트입니다.

---

### 1. 메서드와 `self`, `Self`

* **`self`의 위치와 지위**: 반드시 첫 번째 매개변수여야 하는 **정식 예약어(Keyword)**입니다. (Python의 `self`와 역할이 같지만, Python은 단순 관례일 뿐이고 Rust는 언어 키워드로 강제합니다.)
* **타입 생략 (문법 설탕 - Syntactic Sugar)**: `impl` 블록 안에서 첫 번째 인자가 `self`일 때만 타입을 생략할 수 있습니다.
  * `self` ➔ `self: Self` (값으로 소유권을 완전히 가져와 소비/파괴하는 메서드)
  * `&self` ➔ `self: &Self` (불변 참조 빌림, C++의 `const` 멤버 함수에 해당)
  * `&mut self` ➔ `self: &mut Self` (가변 참조 빌림, 인스턴스 내부 값을 수정하는 메서드)
* **자동 역참조 (Auto-dereferencing)**: `&mut self` 같은 참조자라도 C++처럼 `(*self).name`이나 `self->name`을 쓸 필요 없이, 점(`.`) 연산자가 자동으로 역참조를 수행하므로 `self.name`으로 바로 접근합니다.
* **소문자 `self` vs 대문자 `Self`**:
  * `self`: 현재 호출된 **인스턴스(값)**
  * `Self`: 현재 구조체/트레이트의 **타입(설계도)** (예: `fn new(n: T) -> Self`)

---

### 2. 가변성 (`mut`)과 소유권 / 힙 할당 (`Box<T>`)

* **가변성 원칙 (상속된 가변성 - Inherited Mutability)**: 구조체 정의 시 개별 멤버에 `mut`를 적지 않습니다 (`struct Foo { mut x: i32 }`는 컴파일 에러). 변수 바인딩 선언 시 `let mut s = ...`를 붙이면 모든 멤버가 자동으로 가변 상태가 됩니다. (특정 필드만 수정하려면 C++의 `mutable`과 유사하게 `Cell<T>` / `RefCell<T>` 내부 가변성 패턴을 사용합니다.)
* **힙 할당 (`Box<T>`)**: C++의 `std::unique_ptr<T>`와 같습니다.
  * **3대 핵심 용도**:
    1. **컴파일 타임 크기 확정**: 재귀적 데이터 구조체(Linked List, Tree, AST)의 크기 확정.
    2. **Move 비용 절감**: 대용량 데이터를 힙에 두고 스택에서는 8바이트 주소만 복사하여 Move 비용 절감 및 스택 오버플로우 방지.
    3. **동적 다형성**: `Box<dyn Trait>`를 통한 vtable 기반 다형성 구현.
  * **C++과의 차이점**: Move 이후 원본 변수 접근이 **컴파일 타임에 원천 차단**됩니다 (C++의 `nullptr` 댕글링 위험 없음).

---

### 3. 속성 (`#[...]`)과 자동 구현 (`#[derive]`)

* **`#[must_use]`**: C++의 `[[nodiscard]]`와 같습니다. 함수의 반환값을 변수에 바인딩하거나 처리하지 않으면 컴파일 경고가 발생합니다. 의도적으로 결과를 버릴 때는 `let _ = func();` 관례를 사용합니다.
* **`#[derive(...)]`**: C++의 `= default`와 같습니다. 모든 멤버 필드를 1:1로 대응시켜 기본 트레이트 구현 코드를 자동 생성합니다.
  * 주요 항목: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `Default`
* **`Copy` vs `Clone`**:
  * **`Copy`**: 스택 메모리의 단순 비트 복사 (`memcpy`, C++의 `trivially copyable`). `#[derive(Copy, Clone)]` 형태로 사용하며, `String`, `Box` 등 힙 자원을 가진 타입이나 `Drop`을 구현한 타입은 구현할 수 없습니다.
  * **`Clone`**: `.clone()`을 명시적으로 호출해 수행하는 깊은 복사 (독립된 힙 할당 수반 가능).

---

### 4. 트레이트 구현 (`impl Trait for Type`)

C++의 연산자 오버로딩, 소멸자, 인터페이스 상속은 Rust에서 모두 **`impl [트레이트] for [타입]`** 문법 하나로 일관되게 구현합니다 (여기서 `for`는 "~를 위하여"라는 뜻입니다).

| 트레이트 | 핵심 구현 메서드 시그니처 | 역할 및 C++ 대응 개념 |
| :--- | :--- | :--- |
| `Drop` | `fn drop(&mut self)` | 스코프 종료 시 자동 실행 (소멸자 `~T()`, 직접 호출 금지, 조기 해제는 `std::mem::drop(x)`) |
| `Default` | `fn default() -> Self` | 인자 없는 기본값 생성 (기본 생성자 `T() = default`, `{}` 초기화) |
| `Display` | `fn fmt(&self, f: &mut Formatter) -> Result` | `println!("{}", x)` 사용자용 일반 출력 포맷 (`operator<<`) |
| `From<T>` | `fn from(val: T) -> Self` | 타입 변환 생성자 (구현 시 `.into()` 자동 생성) |
| `PartialEq` | `fn eq(&self, other: &Self) -> bool` | `==`, `!=` 연산자 오버로딩 |
| `Add` | `fn add(self, rhs: Rhs) -> Self::Output` | `+` 연산자 오버로딩 (`rhs`는 우변 관례) |
| `Iterator` | `fn next(&mut self) -> Option<Self::Item>` | `for x in obj` 루프 순회 가능하게 함 (`std::ranges` / 반복자) |

---

### 5. 제네릭 (`<T>`) 타입 추론과 불변성

* **결정 시점 (지연 양방향 추론)**: 변수가 생성될 때(예: `St::new(20)`) 또는 생성 직후 최초로 호출하는 메서드의 인자 타입(예: `vec.push("apple")`)에 의해 `T`가 단 하나로 지연 확정됩니다.
* **단일 정적 타입 확정**: 한 번 결정된 `T`는 해당 변수의 수명이 다할 때까지 **절대 다른 타입으로 바꿀 수 없습니다.** (Python 등 동적 타입 언어와의 결정적 차이이며, C++ 템플릿의 컴파일 타임 인스턴스화와 동일한 원리입니다.)
