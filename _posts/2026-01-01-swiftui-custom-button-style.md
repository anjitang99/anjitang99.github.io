---
title: 6.SwiftUI ButtonStyle로 재사용 가능한 버튼 만들기
date: 2026-01-01 14:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - ButtonStyle
  - ViewModifier
  - CustomComponent
---

## 들어가며

2026년이다. 작년 12월 21일에 쓰고 꽤 오랜만에 돌아왔다 ㅎㅎ (1주일에 두개씩 쓰기로 한거같은데...) 사실 그동안 주제를 꽤 많이 고민해왔다. 중복되는게 많고 이번 주제도 굳이... 해야되나 싶었는데 UI 디자인 시스템을 정의하는데 필요한 중요 내용들이 많아 한번 정리하고 가려고 한다!

`Button`은 꽤 많이 쓰는 컴포넌트이다. 이렇게 많이 사용하는 컴포넌트를 정확히 정의하지 않고 개별뷰에서 만들어 처리하기 시작한다면 코드 중복이 되는건 불보듯 뻔하다. `Button` 자체는 간단하지만, 프로젝트가 커질수록 일관된 버튼 스타일을 유지하는 것이 점점 어려워진다.

이번 포스트에서는 **ButtonStyle 프로토콜**을 활용해 재사용 가능한 커스텀 버튼 스타일을 만들고, **View Extension**으로 편리하게 사용하는 방법을 알아보자.

실제 프로젝트에서 사용할 수 있는 두 가지 스타일을 구현해본다:
- **HighlightButtonStyle**: 눌렀을 때 하이라이트 효과
- **PressableButtonStyle**: 눌렀을 때 스케일 애니메이션

## ButtonStyle 프로토콜이란?

`ButtonStyle`은 SwiftUI에서 버튼의 외형과 동작을 커스터마이징하기 위한 프로토콜이다.

### 왜 ButtonStyle을 사용할까?

일반적인 방법으로 버튼 스타일을 만든다면:

```swift
// 매번 반복되는 코드
Button {
    print("Tapped")
} label: {
    Text("Button 1")
}
.scaleEffect(isPressed1 ? 0.95 : 1)
.animation(.smooth, value: isPressed1)

Button {
    print("Tapped")
} label: {
    Text("Button 2")
}
.scaleEffect(isPressed2 ? 0.95 : 1)
.animation(.smooth, value: isPressed2)
```

버튼마다 `@State` 변수를 만들고, 눌림 상태를 추적해야 한다. 코드 중복도 많고 관리하기도 어렵다.

하지만 `ButtonStyle`을 사용하면:

```swift
// 깔끔하고 재사용 가능
Button("Button 1") {
    print("Tapped")
}
.buttonStyle(PressableButtonStyle())

Button("Button 2") {
    print("Tapped")
}
.buttonStyle(PressableButtonStyle())
```

**ButtonStyle의 장점:**
- ✅ 눌림 상태(`isPressed`)를 자동으로 제공
- ✅ 재사용 가능한 스타일 컴포넌트
- ✅ 일관된 UI/UX
- ✅ @State 관리 불필요

## 구현 1: HighlightButtonStyle

버튼을 눌렀을 때 색상 오버레이로 하이라이트 효과를 주는 스타일이다.

```swift
struct HighlightButtonStyle: ButtonStyle {

    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .overlay {
                // isPressed 상태에 따라 오버레이 색상 변경
                configuration.isPressed ? Color.accent.opacity(0.4) : Color.clear.opacity(0)
            }
    }
}
```

### 핵심 개념

1. **Configuration 파라미터**
   - `configuration.label`: 버튼의 원본 레이블 (Text, Image 등)
   - `configuration.isPressed`: 버튼이 눌렸는지 여부 (Bool)

2. **삼항 연산자로 상태 처리**
   - 눌렸을 때: `Color.accent.opacity(0.4)` - 반투명 악센트 컬러
   - 기본 상태: `Color.clear.opacity(0)` - 투명

3. **overlay 모디파이어**
   - 원본 뷰 위에 새로운 뷰를 겹쳐 표시

### 사용 예시

```swift
Text("Tap Me")
    .padding()
    .background(Color.gray.opacity(0.2))
    .cornerRadius(8)
    .buttonStyle(HighlightButtonStyle())
```

## 구현 2: PressableButtonStyle

버튼을 눌렀을 때 살짝 축소되는 애니메이션 효과를 주는 스타일이다.

```swift
struct PressableButtonStyle: ButtonStyle {

    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .scaleEffect(configuration.isPressed ? 0.95 : 1)
            .animation(.smooth, value: configuration.isPressed)
    }
}
```

### 핵심 개념

1. **scaleEffect**
   - 눌렸을 때: `0.95` (95% 크기로 축소)
   - 기본 상태: `1` (원래 크기)

2. **animation 모디파이어**
   - `.smooth`: iOS 17+에서 제공하는 부드러운 애니메이션 커브
   - `value: configuration.isPressed`: isPressed 값이 변경될 때만 애니메이션 실행

> iOS 17 미만에서는 `.smooth` 대신 `.easeInOut(duration: 0.2)` 등을 사용.
{: .prompt-tip }

## 편리한 사용을 위한 View Extension

매번 `.buttonStyle(HighlightButtonStyle())`를 작성하는 것도 번거롭다. View Extension으로 더 편리하게 만들어보자.

### ButtonStyleOption Enum

```swift
enum ButtonStyleOption {
    case press, highlight, plain
}
```

세 가지 버튼 스타일 옵션을 정의.

### anyButton 메서드

```swift
extension View {

    @ViewBuilder
    func anyButton(_ option: ButtonStyleOption = .plain, action: @escaping () -> Void) -> some View {
        switch option {
        case .press:
            self.pressableButton(action: action)
        case .highlight:
            self.highlightButton(action: action)
        case .plain:
            self.plainButton(action: action)
        }
    }

    private func plainButton(action: @escaping () -> Void) -> some View {
        Button {
            action()
        } label: {
            self
        }
        .buttonStyle(PlainButtonStyle())
    }

    private func highlightButton(action: @escaping () -> Void) -> some View {
        Button {
            action()
        } label: {
            self
        }
        .buttonStyle(HighlightButtonStyle())
    }

    private func pressableButton(action: @escaping () -> Void) -> some View {
        Button {
            action()
        } label: {
            self
        }
        .buttonStyle(PressableButtonStyle())
    }
}
```

### 핵심 개념

1. **@ViewBuilder**
   - `switch` 문에서 다른 타입의 View를 반환할 수 있게 해줌
   - ViewBuilder는 여러 View를 하나로 조합

2. **@escaping 클로저**
   - `action` 클로저가 함수 스코프를 벗어나서 실행될 수 있음을 표시
   - Button의 action은 나중에 사용자가 탭했을 때 실행되므로 `@escaping` 필요

3. **self를 label로 사용**
   - Extension의 `self`는 View 자체를 의미
   - 어떤 View든 버튼의 레이블로 만들 수 있음

4. **private 헬퍼 메서드**
   - 각 스타일별로 Button을 생성하는 메서드
   - 코드 중복 제거 및 가독성 향상

## 실전 사용 예시

```swift
// 1. Highlight 스타일
Text("Hello, world!")
    .padding()
    .frame(maxWidth: .infinity)
    .background(Color.blue)
    .cornerRadius(10)
    .anyButton(.highlight) {
        print("Highlight button tapped")
    }

// 2. Press 스타일
Text("Call to Action")
    .foregroundColor(.white)
    .padding()
    .background(Color.green)
    .cornerRadius(10)
    .anyButton(.press) {
        print("Pressable button tapped")
    }

// 3. Plain 스타일 (기본값)
Text("Plain Button")
    .anyButton {
        print("Plain button tapped")
    }
```

### 사용 팁

- **Highlight**: 배경색이 있는 버튼에 적합 (탭 피드백이 명확)
- **Press**: CTA 버튼, 중요한 액션 버튼에 적합 (시각적 피드백 강함)
- **Plain**: 텍스트 버튼, 미니멀한 디자인에 적합

## 관련 디자인 패턴: Strategy Pattern

ButtonStyle은 객체지향 디자인 패턴 중 **Strategy Pattern**을 활용한 구조다.

### Strategy Pattern이란?

"같은 목적을 다른 방식으로 수행"할 때 사용하는 패턴으로, **알고리즘(전략)을 런타임에 선택**할 수 있게 만든다.

```java
// Java - Strategy Pattern 예시

// 전략 인터페이스: 버튼 렌더링 방식
interface ButtonRenderStrategy {
    void render(String label);
}

// 구체적인 전략 1: Plain 스타일
class PlainButtonStrategy implements ButtonRenderStrategy {
    @Override
    public void render(String label) {
        System.out.println("[" + label + "]");
    }
}

// 구체적인 전략 2: Highlight 스타일
class HighlightButtonStrategy implements ButtonRenderStrategy {
    @Override
    public void render(String label) {
        System.out.println("★ [" + label + "] ★");  // 하이라이트 효과
    }
}

// 구체적인 전략 3: Pressable 스타일
class PressableButtonStrategy implements ButtonRenderStrategy {
    @Override
    public void render(String label) {
        System.out.println(">> [" + label + "] <<");  // 눌림 효과
    }
}

// Context: 전략을 사용하는 버튼
class Button {
    private String label;
    private ButtonRenderStrategy strategy;

    public Button(String label) {
        this.label = label;
        this.strategy = new PlainButtonStrategy();  // 기본 전략
    }

    // 전략 변경 가능
    public void setRenderStrategy(ButtonRenderStrategy strategy) {
        this.strategy = strategy;
    }

    public void render() {
        strategy.render(label);  // 선택된 전략으로 렌더링
    }
}

// 사용
Button button = new Button("Click Me");

button.setRenderStrategy(new PlainButtonStrategy());
button.render();  // 출력: [Click Me]

button.setRenderStrategy(new HighlightButtonStrategy());
button.render();  // 출력: ★ [Click Me] ★

button.setRenderStrategy(new PressableButtonStrategy());
button.render();  // 출력: >> [Click Me] <<
```

### SwiftUI ButtonStyle과의 연결

SwiftUI의 ButtonStyle도 동일한 패턴:

```swift
// 전략 1: Plain
Button("Click Me") { }
    .buttonStyle(PlainButtonStyle())

// 전략 2: Highlight
Button("Click Me") { }
    .buttonStyle(HighlightButtonStyle())

// 전략 3: Pressable
Button("Click Me") { }
    .buttonStyle(PressableButtonStyle())
```

- **목적**: 버튼 렌더링
- **전략**: Plain, Highlight, Pressable 등 다양한 렌더링 방식
- **장점**: 새로운 스타일(전략) 추가 시 기존 코드 수정 불필요

### Strategy Pattern을 사용하는 이유

1. **if-else 지옥 탈출**
```java
// ❌ 나쁜 예: 조건문으로 처리
void renderButton(String label, String style) {
    if (style.equals("plain")) {
        System.out.println("[" + label + "]");
    } else if (style.equals("highlight")) {
        System.out.println("★ [" + label + "] ★");
    } else if (style.equals("pressable")) {
        System.out.println(">> [" + label + "] <<");
    }
    // 새로운 스타일 추가할 때마다 수정 필요...
}

// ✅ 좋은 예: 전략 패턴
button.setRenderStrategy(new HighlightButtonStrategy());
```

2. **Open-Closed Principle (개방-폐쇄 원칙)**
   - 확장에는 열려있고, 수정에는 닫혀있음
   - 새로운 ButtonStyle 추가 시 기존 코드를 건드리지 않음

3. **런타임 전략 교체**
   - 실행 중에 동작 방식을 변경할 수 있음

차이점은:
- **Java**: 명령형, 객체에 전략을 설정
- **SwiftUI**: 선언형, 뷰에 전략을 선언적으로 적용

## 마치며

ButtonStyle 프로토콜을 사용하면:
- ✅ **재사용 가능한** 버튼 컴포넌트 제작
- ✅ **일관된 UX** 제공
- ✅ **상태 관리 간소화** (isPressed 자동 제공)
- ✅ **유지보수성** 향상

특히 디자인 시스템을 구축할 때 ButtonStyle은 필수다. 프로젝트 초기에 기본 버튼 스타일들을 정의해두면, 이후 개발이 훨씬 수월해진다.

## 참고 자료

- [Apple Developer - ButtonStyle](https://developer.apple.com/documentation/swiftui/buttonstyle)
- [Apple Developer - PrimitiveButtonStyle](https://developer.apple.com/documentation/swiftui/primitivebuttonstyle)
- [WWDC - SwiftUI Essentials](https://developer.apple.com/videos/play/wwdc2019/216/)

---

[^1]: ButtonStyle vs PrimitiveButtonStyle: ButtonStyle은 기본 버튼 동작을 유지하면서 외형만 변경하지만, PrimitiveButtonStyle은 버튼의 동작까지 완전히 커스터마이징할 수 있습니다.
