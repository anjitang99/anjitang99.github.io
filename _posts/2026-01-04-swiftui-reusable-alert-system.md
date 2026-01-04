---
title: 7.SwiftUI Alert를 Optional로 관리하기 - Binding 변환의 마법
date: 2026-01-04 22:15:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - Binding
  - Alert
  - Optional
---

## 들어가며

SwiftUI에서 alert를 표시하려면 `Binding<Bool>`이 필요하다. 하지만 이 방식은 생각보다 불편하다. 오늘은 이 문제를 `Binding<Optional>`로 해결하는 방법을 알아보자.

## 기존 방식의 문제점

일반적으로 SwiftUI에서 alert를 구현하면 이렇게 된다:

```swift
struct ChatView: View {
    @State private var showErrorAlert = false
    @State private var errorMessage = ""
    @State private var showSettingsDialog = false

    var body: some View {
        VStack {
            // ...
        }
        .alert("Error", isPresented: $showErrorAlert) {
            Button("OK", role: .cancel) { }
        } message: {
            Text(errorMessage)
        }
        .confirmationDialog("Settings", isPresented: $showSettingsDialog) {
            Button("Report User", role: .destructive) { }
            Button("Delete Chat", role: .destructive) { }
        }
    }

    private func onSendMessagePressed() {
        do {
            try TextValidationHelper.checkIfTextIsValid(text: textFieldText)
            // 메시지 전송 로직
        } catch let error {
            // 에러 발생 시 2개의 상태를 업데이트해야 함
            errorMessage = error.localizedDescription
            showErrorAlert = true
        }
    }

    private func onSettingsPressed() {
        showSettingsDialog = true
    }
}
```

### 무엇이 불편한가?

1. **상태가 분리되어 있다**: `showErrorAlert`와 `errorMessage`를 따로 관리
2. **매번 2줄의 코드**: 에러가 날 때마다 메시지 설정 + Bool 설정
3. **타이틀, 버튼도 커스텀하려면?**: 상태가 더 늘어남
4. **alert가 여러 개면?**: `showErrorAlert`, `showSuccessAlert`, `showWarningAlert`... 끝이 없다

실제로 복잡한 화면에서는 이렇게 된다:

```swift
@State private var showErrorAlert = false
@State private var errorTitle = ""
@State private var errorMessage = ""

@State private var showSuccessAlert = false
@State private var successMessage = ""

@State private var showConfirmDialog = false
@State private var confirmTitle = ""
@State private var confirmMessage = ""
@State private var onConfirm: (() -> Void)?
```

상태 관리가 복잡해지고, 실수하기 쉬워진다.

## 더 나은 방법 - Optional로 관리하기

우리가 원하는 것은 이런 방식이다:

```swift
struct ChatView: View {
    @State private var alert: AnyAppAlert?
    @State private var settingsDialog: AnyAppAlert?

    var body: some View {
        VStack {
            // ...
        }
        .showCustomAlert(alert: $alert)
        .showCustomAlert(type: .confirmationDialog, alert: $settingsDialog)
    }

    private func onSendMessagePressed() {
        do {
            try TextValidationHelper.checkIfTextIsValid(text: textFieldText)
            // 메시지 전송 로직
        } catch let error {
            // 한 줄로 끝!
            alert = AnyAppAlert(error: error)
        }
    }
}
```

훨씬 깔끔하다. 하지만 문제가 있다. **SwiftUI의 `alert`는 `Binding<Bool>`을 요구한다.**

```swift
// SwiftUI의 alert API
func alert<S, A>(_ title: S, isPresented: Binding<Bool>, @ViewBuilder actions: () -> A) -> some View
```

우리는 `Binding<AnyAppAlert?>`를 가지고 있는데, `Binding<Bool>`이 필요하다. 어떻게 해야 할까?

## Binding 변환의 핵심

이 문제의 해결책은 **Binding을 변환**하는 것이다.

### Binding의 구조 이해하기

먼저 `Binding`이 뭔지 알아야 한다. Binding은 getter와 setter로 이루어진 양방향 참조다:

```swift
@frozen @propertyWrapper @dynamicMemberLookup
public struct Binding<Value> {
    public var wrappedValue: Value { get nonmutating set }

    public init(get: @escaping () -> Value, set: @escaping (Value) -> Void)
}
```

핵심은 **Binding은 직접 값을 저장하지 않는다**는 것이다. getter/setter를 통해 다른 곳의 값을 읽고 쓴다.

### 변환 로직 구현

이제 `Binding<AnyAppAlert?>`를 `Binding<Bool>`로 변환하는 Extension을 만들어보자:

```swift
extension Binding where Value == Bool {

    init<T: Sendable>(ifNotNil value: Binding<T?>) {
        self.init {
            // Getter: Optional이 nil이 아니면 true
            value.wrappedValue != nil
        } set: { newValue in
            // Setter: false가 되면 nil로 설정
            if !newValue {
                value.wrappedValue = nil
            }
        }
    }
}
```

#### Getter의 동작

```swift
value.wrappedValue != nil
```

- `alert`가 `nil`이면 → `false` 반환 → alert 안 보임
- `alert`가 `AnyAppAlert`이면 → `true` 반환 → alert 보임

#### Setter의 동작

```swift
if !newValue {
    value.wrappedValue = nil
}
```

이 부분이 **핵심**이다. SwiftUI가 alert를 닫을 때 `isPresented`를 `false`로 설정한다. 이때:

1. `newValue`가 `false`로 들어옴
2. 우리는 원본 Optional을 `nil`로 설정
3. 다음번에 Getter가 호출되면 `nil != nil` → `false`

**왜 `true`일 때는 처리하지 않을까?**

우리가 `alert = AnyAppAlert(...)`를 직접 설정하면:
- Optional이 값을 가지게 됨
- Getter가 자동으로 `true` 반환
- SwiftUI가 alert 표시

즉, `true`로 설정하는 것은 이미 우리가 Optional에 값을 넣는 것으로 처리되므로 Setter에서 신경 쓸 필요가 없다.

### 실제 동작 흐름

```swift
// 1. 초기 상태
@State private var alert: AnyAppAlert? = nil
// Binding<Bool>의 getter 호출 → nil != nil → false → alert 안 보임

// 2. 에러 발생
alert = AnyAppAlert(error: someError)
// Binding<Bool>의 getter 호출 → AnyAppAlert != nil → true → alert 보임

// 3. 사용자가 "OK" 버튼 클릭
// SwiftUI가 isPresented를 false로 설정
// Binding<Bool>의 setter 호출: newValue = false
// → alert = nil

// 4. alert 사라짐
// Binding<Bool>의 getter 호출 → nil != nil → false → alert 안 보임
```

## AnyAppAlert 구조체

이제 Binding 변환을 이해했으니, 실제 alert 데이터를 담을 구조체를 만들자:

```swift
struct AnyAppAlert: Sendable {
    var title: String
    var subTitle: String?
    var buttons: @Sendable () -> AnyView

    init(
        title: String,
        subTitle: String? = nil,
        buttons: (@Sendable () -> AnyView)? = nil
    ) {
        self.title = title
        self.subTitle = subTitle
        self.buttons = buttons ?? {
            AnyView(Button("OK", action: {}))
        }
    }

    init(error: Error) {
        self.init(
            title: "Error",
            subTitle: error.localizedDescription,
            buttons: nil
        )
    }
}
```

`AnyView`를 사용하는 이유는 다양한 타입의 버튼들을 하나의 타입으로 통합하기 위함이다.

## View Extension으로 완성하기

마지막으로 사용하기 편한 View Extension을 만들자:

```swift
enum AlertType {
    case alert, confirmationDialog
}

extension View {

    @ViewBuilder
    func showCustomAlert(
        type: AlertType = .alert,
        alert: Binding<AnyAppAlert?>
    ) -> some View {
        switch type {
        case .alert:
            self.alert(
                alert.wrappedValue?.title ?? "",
                isPresented: Binding(ifNotNil: alert)  // 여기서 변환!
            ) {
                alert.wrappedValue?.buttons()
            } message: {
                if let subtitle = alert.wrappedValue?.subTitle {
                    Text(subtitle)
                }
            }

        case .confirmationDialog:
            self.confirmationDialog(
                alert.wrappedValue?.title ?? "",
                isPresented: Binding(ifNotNil: alert)  // 여기서 변환!
            ) {
                alert.wrappedValue?.buttons()
            } message: {
                if let subtitle = alert.wrappedValue?.subTitle {
                    Text(subtitle)
                }
            }
        }
    }
}
```

핵심은 `Binding(ifNotNil: alert)` 부분이다. 우리가 만든 Extension을 사용해서 `Binding<AnyAppAlert?>`를 `Binding<Bool>`로 변환한다.

## 비교: Before vs After

### Before (기존 방식)

```swift
struct ChatView: View {
    @State private var showErrorAlert = false
    @State private var errorMessage = ""
    @State private var showSettingsDialog = false

    var body: some View {
        VStack {
            // ...
        }
        .alert("Error", isPresented: $showErrorAlert) {
            Button("OK", role: .cancel) { }
        } message: {
            Text(errorMessage)
        }
        .confirmationDialog("Settings", isPresented: $showSettingsDialog) {
            Button("Report", role: .destructive) { }
            Button("Delete", role: .destructive) { }
        }
    }

    private func onSendMessagePressed() {
        do {
            try validate(text)
        } catch let error {
            errorMessage = error.localizedDescription  // 1줄
            showErrorAlert = true                       // 2줄
        }
    }

    private func onSettingsPressed() {
        showSettingsDialog = true
    }
}
```

### After (Optional 방식)

```swift
struct ChatView: View {
    @State private var alert: AnyAppAlert?
    @State private var settingsDialog: AnyAppAlert?

    var body: some View {
        VStack {
            // ...
        }
        .showCustomAlert(alert: $alert)
        .showCustomAlert(type: .confirmationDialog, alert: $settingsDialog)
    }

    private func onSendMessagePressed() {
        do {
            try validate(text)
        } catch let error {
            alert = AnyAppAlert(error: error)  // 한 줄로 끝!
        }
    }

    private func onSettingsPressed() {
        settingsDialog = AnyAppAlert(
            title: "",
            subTitle: "What would you like to do",
            buttons: {
                AnyView(
                    Group {
                        Button("Report", role: .destructive) { }
                        Button("Delete", role: .destructive) { }
                    }
                )
            }
        )
    }
}
```

## 마치며

SwiftUI의 `alert`는 `Binding<Bool>`을 요구하지만, **Binding Extension을 활용하면 Optional로 더 편하게 관리**할 수 있다.

핵심 포인트:
1. **Binding은 getter/setter로 이루어진 참조**다
2. **Binding Extension으로 타입을 변환**할 수 있다
3. **Optional의 nil 여부를 Bool로 매핑**하면 상태 관리가 간단해진다

이 패턴은 alert뿐만 아니라 sheet, fullScreenCover 등 `isPresented`를 사용하는 모든 곳에 적용할 수 있다!

---

## 참고 자료

- [Apple Developer - Binding](https://developer.apple.com/documentation/swiftui/binding)
- [Swift.org - Property Wrappers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers)
