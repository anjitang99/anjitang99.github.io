---
title: View에서 의존성 완전히 걷어내기 - CoreBuilder 패턴
date: 2026-02-18 12:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - DependencyInjection
  - Architecture
  - CoreBuilder
  - SwiftfulThinking
pin: false
---

## 들어가며

SwiftUI로 앱을 만들다 보면 이런 코드를 자주 보게 된다.
```swift
struct ExploreView: View {
    @State var viewModel: ExploreViewModel
    @Environment(DependencyContainer.self) private var container
    // ...
}
```

View가 `DependencyContainer`를 직접 알고 있는 구조. 뭐가 문제냐고? 지금 당장은 괜찮아 보여도 앱이 커지면 문제가 터지기 시작한다.

오늘은 `CoreBuilder` 패턴으로 View에서 의존성을 완전히 걷어내는 과정을 단계별로 알아보자.

## 기존 구조의 문제점

핵심은 **View가 너무 많은 걸 알고 있다**

```swift
struct ExploreView: View {
    @State var viewModel: ExploreViewModel
    @Environment(DependencyContainer.self) private var container  // ❌ DependencyContainer를 알고 있음

    var body: some View {
        NavigationStack(path: $viewModel.path) {
            // ...
            .navigationDestination(for: TabbarPathOption.self) { destination in
                // ❌ 어떤 View로 이동할지도 직접 결정
                switch destination {
                case .chat(let delegate):
                    ChatView(
                        viewModel: ChatViewModel(interactor: CoreInteractor(container: container)),
                        delegate: delegate
                    )
                case .categoryList(let delegate):
                    CategoryListView(
                        viewModel: CategoryListViewModel(interactor: CoreInteractor(container: container)),
                        delegate: delegate
                    )
                }
            }
        }
    }
}
```

문제를 정리하면:

1. **View가 ViewModel 생성 방법을 알아야 함** - `CoreInteractor`, `DependencyContainer`까지 알고 있어
2. **View가 라우팅 목적지를 직접 결정** - 어느 화면으로 이동할지 View 안에서 처리
3. **테스트/Preview가 복잡** - `DependencyContainer`를 환경에 주입해줘야만 Preview가 동작

이걸 세 단계에 걸쳐 개선해보자.

---

## Step 1. CoreBuilder 탄생

먼저 **View 대신 ViewModel을 생성해주는 팩토리 클래스**를 만든다.

```swift
@Observable
@MainActor
class CoreBuilder {
    let interactor: CoreInteractor

    init(interactor: CoreInteractor) {
        self.interactor = interactor
    }

    // 팩토리 메서드 - View 생성 책임을 여기서 담당
    func exploreView() -> some View {
        ExploreView(
            viewModel: ExploreViewModel(interactor: interactor)
        )
    }

    func createAccountView(delegate: CreateAccountDelegate = CreateAccountDelegate()) -> some View {
        CreateAccountView(
            viewModel: CreateAccountViewModel(interactor: interactor),
            delegate: delegate
        )
    }

    func devSettingsView() -> some View {
        DevSettingsView(
            viewModel: DevSettingsViewModel(interactor: interactor)
        )
    }
}
```

그리고 앱 진입점에서 `CoreBuilder`를 환경에 주입한다.

```swift
@main
struct AIChatCourseApp: App {
    var body: some Scene {
        WindowGroup {
            AppView()
                .environment(CoreBuilder(interactor: CoreInteractor(container: container)))
        }
    }
}
```

이제 View에서는 `DependencyContainer` 대신 `CoreBuilder`를 받으면 된다.

```swift
struct ExploreView: View {
    @State var viewModel: ExploreViewModel
    @Environment(CoreBuilder.self) private var builder  // DependencyContainer → CoreBuilder

    var body: some View {
        NavigationStack(path: $viewModel.path) {
            // ...
            .navigationDestination(for: TabbarPathOption.self) { destination in
                switch destination {
                case .chat(let delegate):
                    builder.chatView(delegate: delegate)  // 훨씬 깔끔!
                case .categoryList(let delegate):
                    builder.categoryListView(delegate: delegate)
                }
            }
        }
    }
}
```

View가 ViewModel을 직접 만들지 않아도 되게 됐다!

---

## Step 2. 앱 전체로 확산

Step 1에서 일부 뷰에만 적용했던 패턴을 **앱 전체로 확대**한다. `CoreBuilder`에 모든 뷰의 팩토리 메서드를 추가하는 것이 핵심.

```swift
class CoreBuilder {
    let interactor: CoreInteractor

    // Onboarding 관련
    func welcomeView() -> some View { ... }
    func onboardingColorView(delegate: OnboardingColorDelegate) -> some View { ... }
    func onboardingIntroView(delegate: OnboardingIntroDelegate) -> some View { ... }
    func onboardingCommunityView(delegate: OnboardingCommunityDelegate) -> some View { ... }
    func onboardingCompletedView(delegate: OnboardingCompletedDelegate) -> some View { ... }

    // 탭바 관련
    func tabbarView() -> some View { ... }
    func exploreView() -> some View { ... }
    func chatsView() -> some View { ... }
    func profileView() -> some View { ... }

    // 상세 화면
    func chatView(delegate: ChatViewDelegate) -> some View { ... }
    func categoryListView(delegate: CategoryListDelegate) -> some View { ... }
    func createAvatarView() -> some View { ... }
    func settingsView() -> some View { ... }
    func devSettingsView() -> some View { ... }

    // 셀
    func chatRowCell(delegate: ChatRowCellDelegate) -> some View { ... }
}
```

각 뷰의 파라미터들은 `Delegate` 구조체로 묶어서 전달한다.

```swift
// 파라미터를 구조체로 묶기
struct ChatViewDelegate {
    var avatarId: String = ""
}

struct CategoryListDelegate {
    var category: CharacterOption = .alien
    var imageName: String = ""
}
```

이렇게 하면 파라미터가 늘어나도 `Delegate` 구조체만 수정하면 돼. 호출부를 일일이 찾아다닐 필요가 없다.

---

## Step 3. View에서 CoreBuilder도 제거 ⭐

여기가 핵심이다. Step 2까지는 `@Environment(CoreBuilder.self)`를 통해 View가 여전히 `CoreBuilder`를 알고 있었다. 이것마저 없애버리는 게 마지막 목표.

방법은 간단? 하다. **View가 어떤 View로 이동할지를 외부에서 클로저로 주입**!

```swift
// ❌ Before: CoreBuilder를 Environment로 직접 참조
struct ExploreView: View {
    @State var viewModel: ExploreViewModel
    @Environment(CoreBuilder.self) private var builder

    var body: some View {
        NavigationStack(path: $viewModel.path) {
            // ...
            .navigationDestination(for: TabbarPathOption.self) { destination in
                switch destination {
                case .chat(let delegate):
                    builder.chatView(delegate: delegate)
                }
            }
        }
    }
}
```

```swift
// ✅ After: 클로저로 주입받기
struct ExploreView: View {
    @State var viewModel: ExploreViewModel
    @ViewBuilder var devSettingsView: () -> AnyView
    @ViewBuilder var createAccountView: () -> AnyView
    @ViewBuilder var chatView: (ChatViewDelegate) -> AnyView
    @ViewBuilder var categoryListView: (CategoryListDelegate) -> AnyView

    var body: some View {
        NavigationStack(path: $viewModel.path) {
            // ...
            .navigationDestination(for: TabbarPathOption.self) { destination in
                switch destination {
                case .chat(let delegate):
                    chatView(delegate)  // 그냥 클로저 호출!
                }
            }
        }
    }
}
```

`ExploreView`는 이제 `CoreBuilder`가 뭔지 전혀 모른다. 그냥 "채팅 화면 주세요" 하면 누군가가 만들어서 주는 구조다.

그렇다면 클로저는 누가 주입해줄까? `CoreBuilder`가 해준다.

```swift
class CoreBuilder {
    func exploreView() -> AnyView {
        ExploreView(
            viewModel: ExploreViewModel(interactor: interactor),
            devSettingsView: {
                self.devSettingsView()  // 클로저로 연결
            },
            createAccountView: {
                self.createAccountView()
            },
            chatView: { delegate in
                self.chatView(delegate: delegate)
            },
            categoryListView: { delegate in
                self.categoryListView(delegate: delegate)
            }
        )
        .any()
    }
}
```

`CoreBuilder`가 클로저를 통해 "이 화면이 필요하면 내가 만들어줄게"라고 약속을 주입해주는 구조.

### .any() 헬퍼

반환 타입을 `AnyView`로 통일하기 위해 View extension도 추가!

```swift
extension View {
    func any() -> AnyView {
        AnyView(self)
    }
}
```

`some View`는 컴파일 타임에 구체적인 타입이 정해져서 클로저 타입으로 쓰기 어렵다. `AnyView`로 타입을 지워서 `() -> AnyView` 형태로 통일하는 식으로 헤결했다.

---

## 변경 전후 비교

```
[Before]
View
 ├── @Environment(DependencyContainer) ← DependencyContainer를 직접 앎
 ├── ViewModel 직접 생성
 └── navigationDestination에서 목적지 View 직접 생성


[After]
CoreBuilder
 ├── ViewModel 생성 책임
 └── 목적지 View 생성 책임 (클로저로 주입)
      ↓
View
 ├── ViewModel (외부에서 주입)
 └── @ViewBuilder 클로저 (외부에서 주입)
     → View는 아무것도 직접 만들지 않음
```

View는 이제 정말로 **"어떻게 보여줄지"만 담당**하게 되었다

---

## 이렇게 하면 뭐가 좋을까?

**1. Preview가 쉬워진다**

```swift
// ❌ Before: DependencyContainer를 환경에 넣어줘야 Preview 동작
#Preview {
    ExploreView(viewModel: ExploreViewModel(interactor: ...))
        .environment(DependencyContainer(...))
}

// ✅ After: 클로저만 넣어주면 끝
#Preview {
    ExploreView(
        viewModel: ExploreViewModel(interactor: DevPreview.shared.interactor),
        devSettingsView: { AnyView(Text("DevSettings")) },
        createAccountView: { AnyView(Text("CreateAccount")) },
        chatView: { _ in AnyView(Text("Chat")) },
        categoryListView: { _ in AnyView(Text("CategoryList")) }
    )
}
```

**2. 의존성 교체가 쉬워진다**

`CoreBuilder` 하나만 갈아끼우면 앱 전체의 의존성이 교체돼. Mock 환경으로 전환할 때 특히 강력하다.

**3. View가 순수해진다**

View는 "무엇을 보여줄지"만 알고, "어떻게 만드는지"는 전혀 몰라. 역할이 명확하게 분리돼서 코드를 읽기 훨씬 쉬워진다.

---

## 마치며

세 단계를 거쳐서 View가 점점 순수해지는 과정을 봤다.

1. `DependencyContainer` 직접 참조 → `CoreBuilder` 참조
2. `CoreBuilder` 앱 전체 적용
3. `CoreBuilder` 참조마저 제거 → `@ViewBuilder` 클로저 주입

처음엔 클로저 파라미터가 많아 보여서 오히려 복잡한 것 같기도 하다. 하지만 View 자체가 얼마나 단순해지는지를 보면 충분히 납득이 간다.

**View는 그냥 View만 해야 한다.** 오늘의 핵심이다.
