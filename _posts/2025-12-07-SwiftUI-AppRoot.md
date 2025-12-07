---
title: Commit no.1 - App root
date: 2025-12-07 17:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - SwiftfulThinking
pin: false
---

## 들어가며
뜬금없이 상황극 좀 하겠다.. 당신은 처음으로 Production 수준의 앱을 처음 론칭하려는 사람이다. 어느화면 부터 시작하겠는가? 뱅킹앱이라면 이체부터, 보험앱이라면 청구부터, 다양한 의견이 있을것 같다.  

하지만 국룰은 역시 `TabbarView`, `OnboardingView` 먼저 구조를 잡는것이 시작하기는 가장 좋다고 생각한다. 이런 뷰들은 주제가 다른 앱이더라도 어느 앱이든 존재하는 기본뷰이니깐 말이다. 

## 초기 버전
당신은 처음 구조를 아래와 같이 구성하였다.

```swift
private struct AppView: View {
    @State private var showTabBar: Bool = false
    var body: some View {
        ZStack {
            if showTabBar {
                ZStack {
                    Color.red.ignoresSafeArea()
                    Text("Tabbar")
                }
                .transition(.move(edge: .traling))
            } else {
                ZStack {
                    Color.blue.ignoresSafeArea()
                    Text("Onboarding")
                }
                .transition(.move(edge: .leading))
            }
        }
        .animation(.smooth, value: showTabBar)
        .onTapGesture {
            showTabBar.toggle()
        }
    }
}
```

showTabBar의 상태에 따라 Tabbar뷰와 Onboarding뷰를 이동시키는 간단한 구조를 만들었다. 해당 구조를 좀더 재활용 가능한 형태로 바꿔보자. `@ViewBuilder`를 이용해 각각의 뷰들을 변수로 초기화 할 수 있는 구조를 생각해 보면 어떨까?

```swift
struct AppViewBuilder<TabbarView: View, OnboardingView: View>: View {
    var showTabBar: Bool = false

    @ViewBuilder var tabbarView: TabbarView
    @ViewBuilder var onBoardingView: OnboardingView

    var body: some View {
        ZStack {
            if showTabBar {
                tabbarView
                    .transition(.move(edge: .trailing))
            } else {
                onBoardingView
                    .transition(.move(edge: .leading))
            }
        }
        .animation(.smooth, value: showTabBar)
    }
}
```

AppViewBuilder는 TabbarView와 OnboardingView를 제네릭 타입으로 받음으로써 유연한 뷰 주입이 가능해졌다. 처음 만든 AppView 구조에서 AppViewBuilder를 tabbarView와 onBoardingView를 주입하여 초기화 하게 되면 궁극적으로 처음 만들었던 구조를 따르게 될것이다.

```swift
var body: some View {
    AppViewBuilder(
        showTabBar: showTabBar,
        tabbarView: {
            ZStack {
                Color.red.ignoresSafeArea()
                Text("Tabbar")
            }
        },
        onBoardingView: {
            ZStack {
                Color.blue.ignoresSafeArea()
                Text("Onboarding")
            }
        }
    )
    .onTapGesture {
        showTabBar.toggle()
    }
}
```

결과적으로 훨씬 깔끔하게 더욱 재활용 가능한 코드가 되었다. 다른 앱을 만들때도 비슷한 구조가 필요하다면 복붙해서 써도 될정도로 말이다. `generick`, `ViewBuilder` 와 같은 생소한 개념 때문에 힘들었지만 이렇게 활용할수도 있다는 사실이 놀라웠다. (역시 닉선생님의 구력이 장난아니다 진짜 ;;) 

다음은 TabBar를 좀 더 다듬고 NavigationStacks를 적용하는 포스트로 돌아오겠다.

## 참고 자료

- [Swiftful Thinking](https://www.swiftful-thinking.com/)
