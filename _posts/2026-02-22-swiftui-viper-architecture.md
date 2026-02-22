---
title: ViewModel을 해체하자 - SwiftUI VIPER 아키텍처 적용기
date: 2026-02-22 12:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - VIPER
  - Architecture
  - DependencyInjection
  - CoreBuilder
  - SwiftfulThinking
pin: false
---

## 들어가며

[지난 포스팅](https://anjitang99.github.io/posts/swiftui-corebuilder/)에서 `CoreBuilder` 패턴으로 **View에서 의존성을 완전히 걷어내는 과정**을 다뤘다. View는 이제 깔끔해졌다.

그런데 ViewModel은?

```swift
@Observable
class ExploreViewModel {
    private let interactor: ExploreInteractor

    // 상태 관리
    private(set) var featuredAvatars: [AvatarModel] = []
    private(set) var popularAvatars: [AvatarModel] = []

    // 라우팅 상태 😵
    var showPushNotificationModal: Bool = false
    var showCreateAccountView: Bool = false
    var showDevSettings: Bool = false
    var path: [TabbarPathOption] = []

    // 데이터 로딩
    func loadFeaturedAvatars() async { ... }

    // 라우팅 로직
    func onAvatarPressed(avatar: AvatarModel) {
        path.append(.chat(avatarId: avatar.avatarId, chat: nil))  // ❌ 네비게이션을 직접 관리
    }

    // 모달 관리
    func onDevSettingButtonPressed() {
        showDevSettings = true  // ❌ Bool로 화면 전환 관리
    }
}
```

ViewModel이 **데이터 로딩, 상태 관리, 네비게이션**까지 전부 다 하고 있다. CoreBuilder로 View는 날씬해졌는데, ViewModel이 뚱뚱한 채로 남아있는 거다.

오늘은 이 ViewModel을 **VIPER 패턴으로 해체**하는 과정을 알아보자.

---

## VIPER란?

VIPER는 각 화면을 5개 계층으로 나누는 아키텍처 패턴이다.

```
┌─────────────────────────┐
│     View (UI 렌더링)     │
└────────────┬────────────┘
             │ 사용자 이벤트
┌────────────▼────────────┐
│  Presenter (상태 관리)   │
└─────┬──────────────┬────┘
      │              │
┌─────▼─────┐  ┌─────▼─────┐
│ Interactor │  │  Router   │
│ (데이터)   │  │ (화면전환) │
└────────────┘  └───────────┘

         Entity (데이터 모델)
```

- **View**: UI만 담당. 사용자 이벤트는 Presenter한테 넘김
- **Interactor**: 데이터 접근 담당. API 호출, DB 조회 등
- **Presenter**: 상태 관리 + 비즈니스 로직. View와 Interactor/Router의 중간 다리
- **Entity**: 데이터 모델 (`AvatarModel`, `ChatModel` 등)
- **Router**: 화면 전환 담당. push, sheet, modal 등

핵심은 **ViewModel 하나가 하던 일을 Presenter + Interactor + Router로 쪼개는 것**이다.

---

## Step 1. Interactor 프로토콜 추출

먼저 ViewModel에서 **데이터 접근 부분**을 프로토콜로 분리한다.

이전 포스팅에서 이미 `CoreInteractor`라는 중앙 데이터 접근 레이어를 만들었었다. 여기서 한 단계 더 나아가서, **각 화면이 필요한 메서드만** 프로토콜로 정의한다.

```swift
// ExploreInteractor.swift

@MainActor
protocol ExploreInteractor {
    var categoryRowTest: CategoryRowTestOption { get }
    var createAccountTest: Bool { get }
    var auth: UserAuthInfo? { get }

    func trackEvent(event: LoggableEvent)
    func schedulePushNotificationsForTheNextWeek()
    func canRequestAuthorization() async -> Bool
    func requestAuthorization() async throws -> Bool
    func getFeturedAvatars() async throws -> [AvatarModel]
    func getPopularAvatars() async throws -> [AvatarModel]
}

extension CoreInteractor: ExploreInteractor { }
```

마지막 줄이 핵심이다. `CoreInteractor`가 이미 이 메서드들을 전부 갖고 있으니까, **빈 extension만 선언하면 끝**이다[^1].

[^1]: `CoreInteractor`는 내부적으로 `AvatarManager`, `AuthManager` 등 여러 Manager를 통해 이 메서드들을 구현하고 있다. 자세한 내용은 [CoreBuilder 포스팅](https://anjitang99.github.io/posts/swiftui-corebuilder/) 참고.

### 왜 프로토콜로 분리할까?

```swift
// ❌ Before: Presenter가 CoreInteractor의 모든 메서드에 접근 가능
class ExplorePresenter {
    private let interactor: CoreInteractor  // 채팅, 프로필, 설정 등 전부 접근 가능
}

// ✅ After: Explore에 필요한 메서드만 보인다
class ExplorePresenter {
    private let interactor: ExploreInteractor  // getFeaturedAvatars, getPopularAvatars만 보임
}
```

`ExplorePresenter`는 아바타 목록만 가져오면 되는데, `CoreInteractor`를 직접 들고 있으면 채팅 메시지 전송, 프로필 수정 같은 관련 없는 메서드까지 전부 접근할 수 있다. 프로토콜로 **필요한 인터페이스만 노출**하는 거다.

그리고 테스트할 때 `MockExploreInteractor`를 만들어서 끼워넣기도 쉬워진다.

---

## Step 2. Router 프로토콜 추출

다음은 **네비게이션 로직**을 분리한다. 이게 VIPER에서 가장 체감이 큰 변화다.

### Before: ViewModel이 네비게이션 상태를 직접 관리

```swift
@Observable
class ExploreViewModel {
    // 네비게이션 상태를 Bool/Array로 관리 😵
    var showPushNotificationModal: Bool = false
    var showCreateAccountView: Bool = false
    var showDevSettings: Bool = false
    var path: [TabbarPathOption] = []

    func onAvatarPressed(avatar: AvatarModel) {
        path.append(.chat(avatarId: avatar.avatarId, chat: nil))  // path에 직접 append
    }

    func onDevSettingButtonPressed() {
        showDevSettings = true  // Bool 토글
    }

    func onCancelPushNotificationsButtonPressed() {
        showPushNotificationModal = false  // Bool 토글
    }
}
```

ViewModel이 `path`, `showDevSettings`, `showPushNotificationModal` 같은 **네비게이션 상태**를 직접 들고 있다. 화면이 복잡해질수록 이런 Bool이 계속 늘어난다.

### After: Router 프로토콜로 분리

```swift
// ExploreRouter.swift

@MainActor
protocol ExploreRouter {
    // Segues (화면 이동)
    func showCategoryView(delegate: CategoryListDelegate)
    func showChatView(delegate: ChatViewDelegate)
    func showCreateAccountView(delegate: CreateAccountDelegate, onDisappear: (() -> Void)?)
    func showDevSettingsView()

    // Modals
    func showPushNotificationModal(onEnablePressed: @escaping () -> Void, onCancelPressed: @escaping () -> Void)
    func dismissModal()
}

extension CoreRouter: ExploreRouter { }
```

Presenter에서는 이렇게 쓴다.

```swift
// Before
func onAvatarPressed(avatar: AvatarModel) {
    path.append(.chat(avatarId: avatar.avatarId, chat: nil))  // ❌ path 직접 조작
}

// After
func onAvatarPressed(avatar: AvatarModel) {
    router.showChatView(delegate: ChatViewDelegate(chat: nil, avatarId: avatar.avatarId))  // ✅ Router에게 위임
}
```

`path.append`나 `showDevSettings = true` 같은 **네비게이션 구현 디테일**이 Presenter에서 완전히 사라졌다. Presenter는 "채팅 화면 보여줘"라고 말하기만 하면 되고, **어떻게 보여줄지**(push? sheet? fullScreenCover?)는 Router가 결정한다.

---

## Step 3. ViewModel → Presenter 리네이밍

Interactor와 Router를 분리했으니, ViewModel의 이름을 **Presenter**로 바꾼다.

```swift
// Before: ExploreViewModel.swift
@Observable
class ExploreViewModel {
    private let interactor: ExploreInteractor
    // path, showDevSettings 등 네비게이션 상태들...

    init(interactor: ExploreInteractor) {
        self.interactor = interactor
    }
}

// After: ExplorePresenter.swift
@Observable
class ExplorePresenter {
    private let interactor: ExploreInteractor
    private let router: ExploreRouter  // ✅ Router 추가

    init(interactor: ExploreInteractor, router: ExploreRouter) {
        self.interactor = interactor
        self.router = router
    }
}
```

단순히 이름만 바꾸는 게 아니다. **네비게이션 상태(`path`, `showDevSettings` 등)가 전부 사라진다**는 게 핵심이다. Router가 그 역할을 대신하니까.

View도 당연히 바뀐다.

```swift
// Before
struct ExploreView: View {
    @State var viewModel: ExploreViewModel
}

// After
struct ExploreView: View {
    @State var presenter: ExplorePresenter
}
```

---

## Step 4. CoreRouter - 네비게이션의 중앙 관리소

각 화면의 Router 프로토콜을 **누가 구현**하느냐? `CoreRouter`다.

```swift
@MainActor
struct CoreRouter {
    let router: Router       // CustomRouting 라이브러리의 Router
    let builder: CoreBuilder  // View를 생성하는 팩토리

    // MARK: Segues

    func showChatView(delegate: ChatViewDelegate) {
        router.showScreen(.push) { router in
            builder.chatView(router: router, delegate: delegate)
        }
    }

    func showDevSettingsView() {
        router.showScreen(.sheet) { router in
            builder.devSettingsView(router: router)
        }
    }

    func showCreateAvatarView(onDisappear: @escaping () -> Void) {
        router.showScreen(.fullScreenCover) { router in
            builder.createAvatarView(router: router)
                .onDisappear(perform: onDisappear)
        }
    }

    // MARK: Modals

    func showPushNotificationModal(onEnablePressed: @escaping () -> Void, onCancelPressed: @escaping () -> Void) {
        router.showModal(
            backgroundColor: Color.black.opacity(0.6),
            transition: .move(edge: .bottom)
        ) {
            CustomModalView(
                title: "Enable push notifications?",
                subtitle: "We'll send you reminders and updates!",
                primaryButtonTitle: "Enable",
                primaryButtonAction: { onEnablePressed() },
                secondaryButtonTitle: "Cancel",
                secondaryButtonAction: { onCancelPressed() }
            )
        }
    }

    // MARK: Alerts

    func showAlert(error: Error) {
        router.showAlert(.alert, title: "Error", subTitle: error.localizedDescription, buttons: nil)
    }

    func dismissModal() {
        router.dismissModal()
    }
}
```

`CoreRouter`가 모든 Router 프로토콜을 구현한다.

```swift
extension CoreRouter: ExploreRouter { }
extension CoreRouter: ChatRouter { }
extension CoreRouter: ProfileRouter { }
extension CoreRouter: SettingsRouter { }
// ... 모든 화면의 Router 프로토콜
```

Interactor 패턴이랑 똑같다. `CoreInteractor`가 모든 Interactor 프로토콜을 구현하듯, `CoreRouter`가 모든 Router 프로토콜을 구현한다.

> `CoreRouter`는 `CoreBuilder`를 들고 있어서 **어떤 View든 생성**할 수 있고, `CustomRouting`의 `Router`를 들고 있어서 **어떤 방식으로든 화면 전환**을 할 수 있다.
{: .prompt-info }

---

## Step 5. CoreBuilder 업데이트

`CoreBuilder`도 Router를 주입하는 형태로 바뀐다.

```swift
@MainActor
struct CoreBuilder {
    let interactor: CoreInteractor

    func exploreView(router: Router) -> some View {
        ExploreView(
            presenter: ExplorePresenter(
                interactor: interactor,
                router: CoreRouter(router: router, builder: self)  // ✅ Router 주입
            )
        )
    }

    func chatView(router: Router, delegate: ChatViewDelegate) -> some View {
        ChatView(
            presenter: ChatPresenter(
                interactor: interactor,
                router: CoreRouter(router: router, builder: self)
            ),
            delegate: delegate
        )
    }

    // 모든 화면에 같은 패턴 적용
}
```

모든 팩토리 메서드가 `router: Router`를 파라미터로 받는다. 이 `Router`는 `CustomRouting` 라이브러리가 제공하는 네비게이션 컨텍스트인데, 이걸 `CoreRouter`로 감싸서 Presenter에 주입하는 구조다.

탭바에서는 각 탭마다 `RouterView`로 독립적인 네비게이션 컨텍스트를 만들어준다.

```swift
func tabbarView() -> some View {
    TabBarView(
        tabs: [
            TabBarScreen(title: "Explore", systemImage: "eyes", screen: {
                RouterView { router in          // 각 탭마다 독립적인 Router
                    exploreView(router: router)
                }
                .any()
            }),
            TabBarScreen(title: "Chats", systemImage: "bubble.left.and.bubble.right.fill", screen: {
                RouterView { router in
                    chatsView(router: router)
                }
                .any()
            }),
            // ...
        ]
    )
}
```

---

## 전체 구조 정리

최종적으로 전체 의존성 흐름을 보면 이렇게 된다.

```
DependencyContainer
    │
    ▼
CoreInteractor (모든 Manager를 묶는 중앙 데이터 레이어)
    │
    ├── implements ExploreInteractor (프로토콜)
    ├── implements ChatInteractor (프로토콜)
    ├── implements ProfileInteractor (프로토콜)
    └── implements SettingsInteractor (프로토콜) ...
    │
CoreBuilder (View 팩토리)
    │
    │   각 View를 생성할 때:
    │   ┌──────────────────────────────────┐
    │   │ ExploreView                      │
    │   │  └─ ExplorePresenter             │
    │   │      ├─ interactor: CoreInteractor│  (ExploreInteractor 프로토콜로 접근)
    │   │      └─ router: CoreRouter        │  (ExploreRouter 프로토콜로 접근)
    │   └──────────────────────────────────┘
    │
CoreRouter (모든 화면 전환을 중앙에서 처리)
    │
    ├── implements ExploreRouter (프로토콜)
    ├── implements ChatRouter (프로토콜)
    ├── implements ProfileRouter (프로토콜)
    └── implements SettingsRouter (프로토콜) ...
```

각 Presenter는 **프로토콜을 통해서만** Interactor와 Router에 접근한다. 실제 구현체(`CoreInteractor`, `CoreRouter`)가 뭔지는 모른다.

---

## 테스트가 쉬워진다

프로토콜 기반이니까 테스트할 때 Mock을 끼워넣기가 아주 쉽다.

```swift
// Mock Interactor
class MockExploreInteractor: ExploreInteractor {
    var categoryRowTest: CategoryRowTestOption = .original
    var createAccountTest: Bool = false
    var auth: UserAuthInfo? = nil

    var mockAvatars: [AvatarModel] = []

    func getFeturedAvatars() async throws -> [AvatarModel] {
        return mockAvatars  // 원하는 데이터를 바로 반환
    }

    func getPopularAvatars() async throws -> [AvatarModel] {
        return mockAvatars
    }

    func trackEvent(event: LoggableEvent) { }
    // ...
}

// Mock Router
class MockExploreRouter: ExploreRouter {
    var showChatViewCalled = false
    var lastChatDelegate: ChatViewDelegate?

    func showChatView(delegate: ChatViewDelegate) {
        showChatViewCalled = true       // 호출 여부 추적
        lastChatDelegate = delegate     // 전달된 데이터 캡처
    }
    // ...
}

// 테스트
func testOnAvatarPressed() {
    let mockRouter = MockExploreRouter()
    let presenter = ExplorePresenter(
        interactor: MockExploreInteractor(),
        router: mockRouter
    )

    let avatar = AvatarModel.mock
    presenter.onAvatarPressed(avatar: avatar)

    XCTAssertTrue(mockRouter.showChatViewCalled)  // ✅ Router가 호출됐는지 검증
    XCTAssertEqual(mockRouter.lastChatDelegate?.avatarId, avatar.avatarId)  // ✅ 올바른 데이터가 전달됐는지 검증
}
```

> MVVM에서는 `path.append()`가 제대로 됐는지 확인하려면 path 배열을 뒤져야 했다. VIPER에서는 **"Router의 특정 메서드가 호출됐는가?"**만 확인하면 된다. 테스트 의도가 훨씬 명확하다.
{: .prompt-tip }

---

## 마치며

MVVM에서 VIPER까지의 진화 과정을 정리하면 이렇다.

```
[Step 1] MVVM
View ← ViewModel ← DependencyContainer
└─ View가 DependencyContainer를 직접 앎
└─ ViewModel이 데이터 + 라우팅 전부 담당

    ↓

[Step 2] CoreBuilder 도입
View ← ViewModel ← CoreInteractor
         ↑
     CoreBuilder (View 생성 책임 분리)
└─ View에서 의존성 제거

    ↓

[Step 3] VIPER
View ← Presenter ← Interactor (프로토콜)
                  ← Router (프로토콜)
         ↑
     CoreBuilder (Presenter + Router 주입)
└─ ViewModel의 책임을 3개로 분리
└─ 프로토콜 기반으로 테스트 용이
```

ViewModel 하나에 몰려있던 책임을 **Presenter(상태 관리) + Interactor(데이터) + Router(네비게이션)**로 쪼갠 것이 핵심이다.

물론 파일 수가 늘어나는 건 사실이다. 화면 하나당 최소 4개 파일(View, Presenter, Interactor, Router)이 생기니까. 하지만 각 파일의 역할이 명확해지고, 테스트가 쉬워지고, 한 파일이 비대해지는 문제가 사라진다. 이 트레이드오프를 어떻게 볼지는 프로젝트 규모에 따라 다르겠지만, 앱이 커질수록 VIPER의 진가가 드러난다.

**View는 View만, Presenter는 로직만, Router는 화면 전환만.** 각자 자기 일만 하면 된다. 😎
