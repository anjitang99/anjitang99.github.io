---
title: SwiftUI의 @State가 가능한 이유 - nonmutating
date: 2025-12-11 13:50:00 +0900
categories:
  - iOS
  - Swift
tags:
  - Swift
  - SwiftUI
  - PropertyWrapper
  - nonmutating
pin: false
---

## 들어가며

사실 `PropertyWrapper`에 관한 글을 쓰고 싶었는데 어쩌다 보니 `nonmutating`을 정리하면서 원큐에 같이 정리하는 포스트가 될것같다. (근데 이제 뜬금없이 C언어 포인터를 곁들인...) 단순한 문법 키워드 ( `nonmutating`) 에서 파생된 궁금증이라 Top-Down 으로 파고 드는 느낌은 아니지만... 정리만 잘 되면 되지 않을까?? 일단 시작해보자

간단한 `PropertyWrapper` 예제코드를 보는 과정에서 문득 이런 의문이 들었다.

```swift
struct ContentView: View {  // struct인데...
    @State private var count = 0

    var body: some View {  // 이건 계산 프로퍼티인데...
        Button("Increment") {
            count += 1  // 어떻게 값을 변경할 수 있지?
        }
    }
}
```

**"View는 struct인데 어떻게 상태를 변경할 수 있을까?"**

Swift에서 struct는 값 타입이고, 프로퍼티를 변경하려면 `mutating` 키워드가 필요하다. 그런데 SwiftUI의 `body`는 `mutating`이 아니다. 대체 무슨 일이 벌어지고 있는 걸까?

이 글에서는 `@State`가 가능한 이유, 그리고 그 비밀의 열쇠인 **`nonmutating` 키워드**에 대해 깊이 알아보겠다.

---

## 1. 문제 상황: struct의 한계

먼저 일반적인 Swift struct의 동작을 살펴보자.

### 일반적인 struct

```swift
struct Counter {
    var count: Int = 0

    func increment() {  // ❌ 컴파일 에러!
        count += 1
        // Error: Cannot assign to property: 'self' is immutable
    }
}
```

struct에서 프로퍼티를 변경하려면 반드시 `mutating` 키워드가 필요하다.

```swift
struct Counter {
    var count: Int = 0

    mutating func increment() {  // ✅ OK!
        count += 1
    }
}
```

### SwiftUI에서는?

그런데 SwiftUI의 View는 어떻게 이게 가능할까?

```swift
struct ContentView: View {
    @State private var count = 0

    var body: some View {  // mutating 아님!
        Button("Increment") {
            count += 1  // 근데 가능함!
        }
    }
}
```

이것이 가능한 이유는 바로 **`@State` 프로퍼티 래퍼**가 `nonmutating set`을 사용하기 때문이다.

---

## 2. C 포인터로 이해하기

`nonmutating`을 이해하는 가장 쉬운 방법은 **C 언어의 포인터** 개념을 빌려오는 것이다.

### mutating vs nonmutating

#### mutating (일반적인 경우)

```swift
struct Point {
    var x: Int
    var y: Int

    mutating func moveRight() {
        x += 1  // struct 자체의 메모리 변경
    }
}
```

**C 언어로 표현하면:**

```c
struct Point {
    int x;
    int y;
};

void moveRight(struct Point *self) {
    self->x += 1;  // struct 내부 값 변경
}
```

**메모리:**
```
Stack:
Point: [x: 5, y: 10]  →  [x: 6, y: 10]
       ↑ struct 자체가 변경됨
```

#### nonmutating (포인터 활용)

```swift
struct StateWrapper {
    private var storage: Storage  // 참조 타입!

    var value: Int {
        get { storage.value }
        nonmutating set {
            storage.value = newValue  // 포인터가 가리키는 값만 변경
        }
    }
}
```

**C 언어로 표현하면:**

```c
struct StateWrapper {
    Storage *storage;  // 포인터!
};

void setValue(const struct StateWrapper *self, int newValue) {
    // self는 const (포인터 자체는 불변)
    // *self->storage는 변경 가능 (가리키는 값 변경)
    self->storage->value = newValue;
}
```

**메모리:**
```
Stack:
StateWrapper: [storage: 0x1234]  // 포인터 자체는 불변!
              ↓
Heap:
0x1234: [value: 5]  →  [value: 10]  // 가리키는 값만 변경
```

### 핵심 개념

| Swift | C 언어 | 설명 |
|-------|--------|------|
| `mutating set` | `self->value = x` | struct 내부 값 변경 |
| `nonmutating set` | `*self->pointer = x` | 포인터가 가리키는 값 변경 |
| `@State var x` | `int *x` | 힙을 가리키는 포인터 |
| `var x: Int` | `int x` | 스택의 값 |

---

## 3. @State 내부 구조 분석

이제 실제 `@State`가 어떻게 구현되어 있는지 살펴보자.

### @State의 실제 구조 (단순화)

```swift
@propertyWrapper
struct State<Value> {

    // 힙에 할당되는 Storage 클래스
    private class Storage {
        var value: Value
        init(_ value: Value) {
            self.value = value
        }
    }

    // struct 내부에 Storage 참조 보관
    private var storage: Storage

    var wrappedValue: Value {
        get {
            storage.value  // 포인터 역참조
        }
        nonmutating set {
            storage.value = newValue  // 포인터가 가리키는 값만 변경
            // storage 자체는 불변!
        }
    }

    init(wrappedValue: Value) {
        self.storage = Storage(wrappedValue)
    }
}
```

### 왜 nonmutating이 필요한가?

만약 `nonmutating` 없이 일반 `set`을 사용한다면:

```swift
var wrappedValue: Value {
    get { storage.value }
    set {  // mutating으로 간주됨!
        storage.value = newValue
    }
}
```

**View에서 사용할 때 오류 발생:**

```swift
struct ContentView: View {
    @State private var count = 0

    var body: some View {  // body는 non-mutating
        Button("Increment") {
            count += 1  // ❌ Error: Cannot assign to property: 'self' is immutable
        }
    }
}
```

### nonmutating의 역할

`nonmutating`을 붙이면:
- Swift 컴파일러에게 **"이 setter는 struct 자체를 변경하지 않는다"**고 알림
- 실제로는 **외부 Storage(힙)만 변경**
- `const` 메서드에서도 호출 가능

---

## 4. 직접 만들어보기: 커스텀 프로퍼티 래퍼

이제 실제로 `nonmutating`을 사용하는 커스텀 프로퍼티 래퍼를 만들어보자.

### 예제 1: @UserDefault

UserDefaults에 자동으로 저장되는 프로퍼티 래퍼를 만들어보자.

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get {
            UserDefaults.standard.object(forKey: key) as? T ?? defaultValue
        }
        nonmutating set {
            UserDefaults.standard.set(newValue, forKey: key)
            // UserDefaults는 외부 저장소
            // struct 자체는 변경 안 함!
        }
    }

    init(wrappedValue: T, _ key: String) {
        self.key = key
        self.defaultValue = wrappedValue
    }
}
```

**사용:**

```swift
struct SettingsView: View {
    @UserDefault(wrappedValue: false, "isDarkMode") var isDarkMode: Bool
    @UserDefault(wrappedValue: "Guest", "username") var username: String

    var body: some View {
        VStack {
            Toggle("Dark Mode", isOn: $isDarkMode)
            TextField("Username", text: $username)
        }
    }
}
```

**왜 nonmutating이 필요한가?**
- `View`는 struct
- `body`는 non-mutating
- `nonmutating` 없으면 Toggle, TextField 사용 불가!

### 예제 2: @FileStorage

파일 시스템에 자동으로 저장되는 프로퍼티 래퍼.

```swift
@propertyWrapper
struct FileStorage: DynamicProperty {
    @State private var cachedValue: String
    let filename: String

    var wrappedValue: String {
        get {
            cachedValue
        }
        nonmutating set {
            save(newValue)
        }
    }

    var projectedValue: Binding<String> {
        Binding(
            get: { wrappedValue },
            set: { wrappedValue = $0 }
        )
    }

    init(wrappedValue: String, _ filename: String) {
        self.filename = filename

        // 파일에서 읽기
        let url = FileManager.default
            .urls(for: .documentDirectory, in: .userDomainMask)
            .first!
            .appendingPathComponent("\(filename).txt")

        if let data = try? String(contentsOf: url, encoding: .utf8) {
            _cachedValue = State(initialValue: data)
        } else {
            _cachedValue = State(initialValue: wrappedValue)
        }
    }

    func save(_ value: String) {
        cachedValue = value

        let url = FileManager.default
            .urls(for: .documentDirectory, in: .userDomainMask)
            .first!
            .appendingPathComponent("\(filename).txt")

        try? value.write(to: url, atomically: true, encoding: .utf8)
    }
}
```

**사용:**

```swift
struct NoteView: View {
    @FileStorage(wrappedValue: "", "note") var note: String

    var body: some View {
        TextEditor(text: $note)
            .padding()
    }
}
```

### 예제 3: @Clamped (값 범위 제한)

```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    private var value: Value
    private let range: ClosedRange<Value>

    var wrappedValue: Value {
        get { value }
        set {
            value = min(max(newValue, range.lowerBound), range.upperBound)
        }
    }

    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
}
```

**이 경우는 nonmutating이 필요 없다!**

왜냐하면:
- `value`는 struct 내부의 값 타입
- 실제로 struct가 변경됨
- SwiftUI View에서 직접 사용 불가 (mutating 필요)

**사용 예:**

```swift
struct Game {  // View가 아님!
    @Clamped(0...100) var health = 100

    mutating func takeDamage(_ amount: Int) {
        health -= amount  // 자동으로 0~100 범위 유지
    }
}
```

---

## 5. 실전 활용: SwiftUI의 다른 프로퍼티 래퍼들

### @Binding

```swift
@propertyWrapper
struct Binding<Value> {
    private var getValue: () -> Value
    private var setValue: (Value) -> Void

    var wrappedValue: Value {
        get { getValue() }
        nonmutating set { setValue(newValue) }
    }

    init(get: @escaping () -> Value, set: @escaping (Value) -> Void) {
        self.getValue = get
        self.setValue = set
    }
}
```

**포인트:**
- 클로저를 통한 간접 참조
- `nonmutating set`으로 View에서 사용 가능

### @Published (Combine)

```swift
@propertyWrapper
class Published<Value> {
    private var value: Value
    private let subject = PassthroughSubject<Value, Never>()

    var wrappedValue: Value {
        get { value }
        set {
            value = newValue
            subject.send(newValue)
        }
    }

    var projectedValue: Publisher<Value, Never> {
        subject.eraseToAnyPublisher()
    }

    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
}
```

**포인트:**
- `Published`는 **class**이므로 참조 타입
- 따라서 `nonmutating` 불필요
- ObservableObject와 함께 사용

### @StateObject vs @ObservedObject

```swift
// @StateObject: View가 객체의 lifetime 소유
@propertyWrapper
struct StateObject<ObjectType: ObservableObject> {
    private var storage: ObjectType?

    var wrappedValue: ObjectType {
        get { storage! }
        nonmutating set { }  // 실제로는 set 불가
    }
}

// @ObservedObject: 외부에서 전달받은 객체
@propertyWrapper
struct ObservedObject<ObjectType: ObservableObject> {
    var wrappedValue: ObjectType

    // 참조 타입이므로 nonmutating 불필요
}
```

---

## 6. 핵심 정리

### nonmutating이 필요한 경우

✅ **외부 저장소 사용 시:**
- @State (힙의 Storage 참조)
- @UserDefault (UserDefaults)
- @FileStorage (파일 시스템)
- @Binding (클로저를 통한 간접 참조)

✅ **struct인데 View에서 사용해야 할 때:**
- SwiftUI View는 struct
- `body`는 non-mutating
- `nonmutating set` 필수!

### nonmutating이 필요 없는 경우

❌ **참조 타입 (class) 사용 시:**
- @Published
- @ObservedObject
- class 자체가 참조이므로 불필요

❌ **struct 내부 값 직접 변경 시:**
- @Clamped (내부 값 타입)
- View 외부에서 사용
- mutating 메서드 필요

### 메모리 관점 요약

| 케이스 | struct 메모리 | 외부 메모리 | nonmutating 필요? |
|-------|-------------|-----------|-----------------|
| `var x: Int` | 변경됨 | - | ❌ (mutating) |
| `@State var x` | 불변 | 변경됨 (힙) | ✅ |
| `@UserDefault var x` | 불변 | 변경됨 (UserDefaults) | ✅ |
| `@Published var x` | - | 변경됨 (class 내부) | ❌ (class) |

---

## 마치며

**핵심 개념:**
1. **struct는 값 타입** → 변경하려면 mutating 필요
2. **nonmutating set** → "struct 자체는 안 바뀜" 선언
3. **포인터 개념** → 외부 저장소만 변경
4. **SwiftUI View** → body는 non-mutating이므로 필수

이 개념을 이해하면:
- @State, @Binding의 동작 원리 이해
- 커스텀 프로퍼티 래퍼 제작 가능
- SwiftUI의 아키텍처 깊이 이해 가능

다음에는 SwiftUI 4강을 정리한 포스트로 돌아오겠다.


---

## 참고 자료

- [Swift Property Wrappers - Swift.org](https://docs.swift.org/swift-book/LanguageGuide/Properties.html)
- [SwiftUI State and Data Flow - Apple](https://developer.apple.com/documentation/swiftui/state-and-data-flow)
- [Understanding @State in SwiftUI](https://www.hackingwithswift.com/quick-start/swiftui/what-is-the-state-property-wrapper)

---
