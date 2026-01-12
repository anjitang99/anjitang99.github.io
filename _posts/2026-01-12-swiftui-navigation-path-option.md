---
title: SwiftUI NavigationStack - enum으로 네비게이션 타입 안전하게 관리하기
date: 2026-01-12 15:30:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - NavigationStack
  - NavigationPath
  - Enum
  - TypeSafety
---

## 들어가며

SwiftUI로 앱을 만들다 보면 여러 화면에서 같은 목적지로 이동하는 경우가 많아. 예를 들어:

- 탐색(Explore) 화면에서 → 채팅 화면으로
- 프로필 화면에서 → 채팅 화면으로
- 카테고리 리스트에서 → 채팅 화면으로

이런 식으로 여러 곳에서 `ChatView`로 이동하는데, 각 화면마다 `navigationDestination`을 중복으로 작성하면 코드가 지저분해지고 유지보수도 힘들어져. 😵

오늘은 **Enum 기반 NavigationPathOption 패턴**을 사용해서 네비게이션 로직을 중앙 집중식으로 관리하는 방법을 알아보자!

## 기존 방식의 문제점

일반적으로 NavigationStack을 사용할 때 이런 식으로 작성하잖아:

```swift
struct ExploreView: View {
    @State private var avatars: [AvatarModel] = AvatarModel.mocks

    var body: some View {
        NavigationStack {
            List {
                ForEach(avatars) { avatar in
                    // NavigationLink를 사용한 기본 방식
                    NavigationLink(value: avatar.avatarId) {
                        Text("Chat with \(avatar.name)")
                    }
                }
            }
            .navigationDestination(for: String.self) { avatarId in
                ChatView(avatarId: avatarId)
            }
        }
    }
}
```

그런데 문제가 뭐냐면:

1. **중복 코드**: 다른 화면에서도 ChatView로 이동하려면 똑같은 `navigationDestination` 코드를 또 써야 해
2. **타입 안전성 부족**: String이나 단순 타입으로 관리하면 오타나 잘못된 값 전달 가능성이 있어
3. **네비게이션 로직 분산**: 각 화면마다 네비게이션 로직이 흩어져 있어서 전체 흐름을 파악하기 어려워
4. **확장성 문제**: 새로운 화면을 추가할 때마다 모든 진입점에 코드를 추가해야 해

이걸 어떻게 개선할 수 있을까? 🤔

## NavigationPathOption - Enum으로 타입 정의하기

핵심 아이디어는 **가능한 모든 네비게이션 경로를 Enum으로 정의**하는 거야:

```swift
enum NavigationPathOption: Hashable {
    case chat(avatarId: String)
    case category(category: CharacterOption, imageName: String)
}
```

### 왜 Enum을 사용할까?

Enum을 사용하면 이런 장점들이 있어:

1. **타입 안전성**: 컴파일 타임에 잘못된 네비게이션을 잡아낼 수 있어
2. **Associated Values**: 각 케이스마다 필요한 데이터를 타입과 함께 전달할 수 있어
3. **Exhaustive Switch**: 모든 케이스를 처리하지 않으면 컴파일 에러가 나서 빠뜨림이 없어

### Hashable은 왜 채택하지?

NavigationStack의 `path` 파라미터는 `Hashable`을 준수하는 타입만 받아. 이유는:

- SwiftUI가 내부적으로 네비게이션 스택을 관리할 때 각 경로를 식별하기 위해 해시값을 사용하기 때문이야
- Enum은 자동으로 Hashable을 구현해주니까 딱히 추가 코드가 필요 없어 👍

```swift
// NavigationStack의 초기화 시그니처 (간략화)
init<Data>(path: Binding<[Data]>) where Data: Hashable
// Data 타입이 Hashable을 준수해야만 사용할 수 있어!
```

## 중앙 집중식 네비게이션 관리

이제 모든 네비게이션 로직을 **한 곳에 모아서 관리**해보자. View extension을 만들어서 재사용 가능하게 만들 거야:

```swift
extension View {

    func navigationDestinationForCoreModule(
        path: Binding<[NavigationPathOption]>
    ) -> some View {
        self
            .navigationDestination(for: NavigationPathOption.self) { option in
                switch option {
                case .chat(avatarId: let avatarId):
                    ChatView(avatarId: avatarId)

                case .category(category: let category, imageName: let imageName):
                    CategoryListView(
                        path: path,
                        category: category,
                        imageName: imageName
                    )
                }
            }
    }
}
```

### 코드 분석

- **View extension**: 모든 View에서 사용할 수 있도록 확장 메서드로 정의했어
- **Switch 문**: 모든 NavigationPathOption 케이스를 한 곳에서 처리해
- **Associated Values 추출**: `let` 바인딩으로 각 케이스의 데이터를 깔끔하게 가져와
- **path 전달**: CategoryListView처럼 중첩된 네비게이션이 필요한 경우 path를 계속 전달할 수 있어

이렇게 하면 네비게이션 목적지를 **딱 한 곳에서만 정의**하게 돼. 나중에 ChatView의 생성자가 바뀌거나 새로운 파라미터가 필요해져도 여기만 수정하면 끝! 🎯

## 실제 사용 예시

이제 각 화면에서 어떻게 사용하는지 볼까?

### 1. ExploreView - 탐색 화면

```swift
struct ExploreView: View {
    @State private var featuredAvatars: [AvatarModel] = AvatarModel.mocks
    @State private var categories: [CharacterOption] = CharacterOption.allCases
    @State private var popularAvatars: [AvatarModel] = AvatarModel.mocks

    // ✅ 네비게이션 path를 State로 관리
    @State private var path: [NavigationPathOption] = []

    var body: some View {
        NavigationStack(path: $path) {
            List {
                featuredSection
                categorySection
                popularSection
            }
            .navigationTitle("Explore")
            // ✅ 우리가 만든 extension 사용
            .navigationDestinationForCoreModule(path: $path)
        }
    }

    // Featured 섹션
    private var featuredSection: some View {
        Section {
            ScrollView(.horizontal) {
                HStack(spacing: 16) {
                    ForEach(featuredAvatars) { avatar in
                        HeroCellView(
                            title: avatar.name,
                            subtitle: avatar.characterDescription,
                            imageName: avatar.profileImageName
                        )
                        .anyButton {
                            // ✅ path에 추가만 하면 네비게이션 발생
                            onAvatarPressed(avatar: avatar)
                        }
                    }
                }
            }
        }
    }

    // Category 섹션
    private var categorySection: some View {
        Section {
            ScrollView(.horizontal) {
                HStack(spacing: 12) {
                    ForEach(categories, id: \.self) { category in
                        let imageName = popularAvatars.first(where: {
                            $0.characterOption == category
                        })?.profileImageName

                        if let imageName {
                            CategoryCellView(
                                title: category.plural.capitalized,
                                imageName: imageName
                            )
                            .anyButton {
                                // ✅ 카테고리 화면으로 이동
                                onCategoryPressed(
                                    category: category,
                                    imageName: imageName
                                )
                            }
                        }
                    }
                }
            }
        }
    }

    // ✅ 네비게이션 메서드들 - 아주 심플!
    private func onAvatarPressed(avatar: AvatarModel) {
        path.append(.chat(avatarId: avatar.avatarId))
    }

    private func onCategoryPressed(category: CharacterOption, imageName: String) {
        path.append(.category(category: category, imageName: imageName))
    }
}
```

### 2. ChatsView - 채팅 목록 화면

```swift
struct ChatsView: View {
    @State private var chats: [ChatModel] = ChatModel.mocks
    @State private var path: [NavigationPathOption] = []

    var body: some View {
        NavigationStack(path: $path) {
            List {
                ForEach(chats) { chat in
                    ChatRowCellViewBuilder(
                        avatarId: chat.avatarId,
                        chat: chat,
                        getAvatar: {
                            try? await Task.sleep(for: .seconds(1))
                            return AvatarModel.mocks.randomElement()
                        },
                        getLastChatMessage: {
                            try? await Task.sleep(for: .seconds(1))
                            return ChatMessageModel.mocks.randomElement()!
                        }
                    )
                    .anyButton(.highlight) {
                        onChatPressed(chat: chat)
                    }
                    .removeListRowFormatting()
                }
            }
            .navigationTitle("Chats")
            .navigationDestinationForCoreModule(path: $path)
        }
    }

    private func onChatPressed(chat: ChatModel) {
        path.append(.chat(avatarId: chat.avatarId))
    }
}
```

### 3. ProfileView - 프로필 화면

```swift
struct ProfileView: View {
    @State private var myAvatars: [AvatarModel] = []
    @State private var path: [NavigationPathOption] = []

    var body: some View {
        NavigationStack(path: $path) {
            List {
                myInfoSection
                myAvatarsSection
            }
            .navigationTitle("Profile")
            .navigationDestinationForCoreModule(path: $path)
        }
    }

    private var myAvatarsSection: some View {
        Section {
            ForEach(myAvatars) { avatar in
                CustomListCellView(
                    imageName: avatar.profileImageName,
                    title: avatar.name,
                    subtitle: nil
                )
                .anyButton(.highlight) {
                    onAvatarPressed(avatar: avatar)
                }
                .removeListRowFormatting()
            }
        } header: {
            Text("My Avatars")
        }
    }

    private func onAvatarPressed(avatar: AvatarModel) {
        path.append(.chat(avatarId: avatar.avatarId))
    }
}
```

### 패턴 정리

모든 화면에서 공통적으로:

1. `@State private var path: [NavigationPathOption] = []` - path 상태 관리
2. `NavigationStack(path: $path)` - path와 바인딩
3. `.navigationDestinationForCoreModule(path: $path)` - 네비게이션 목적지 설정
4. `path.append(.chat(...))` - 간단한 네비게이션 트리거

딱 4줄의 보일러플레이트만 있으면 돼. 깔끔하지? ✨

## CategoryListView - 중첩 네비게이션

카테고리 화면에서도 다시 채팅으로 이동할 수 있어야 하니까, path를 계속 전달하는 게 중요해:

```swift
struct CategoryListView: View {
    // ✅ 부모로부터 path를 Binding으로 받아
    @Binding var path: [NavigationPathOption]
    var category: CharacterOption = .alien
    var imageName: String = Constants.randomImage
    @State private var avatars: [AvatarModel] = AvatarModel.mocks

    var body: some View {
        List {
            ForEach(avatars) { avatar in
                CustomListCellView(
                    imageName: avatar.profileImageName,
                    title: avatar.name,
                    subtitle: avatar.characterDescription
                )
                .anyButton(.highlight) {
                    onAvatarPressed(avatar: avatar)
                }
                .removeListRowFormatting()
            }
        }
        .navigationTitle(category.plural.capitalized)
    }

    private func onAvatarPressed(avatar: AvatarModel) {
        // ✅ 같은 path에 append하면 스택에 추가됨
        path.append(.chat(avatarId: avatar.avatarId))
    }
}
```

이렇게 하면 **네비게이션 스택이 제대로 유지**돼:

```
ExploreView → CategoryListView → ChatView
  (루트)           ↑                ↑
               path[0]          path[1]
```

루트 뷰(ExploreView)는 path에 포함되지 않고, push된 화면들만 path 배열에 들어가. 뒤로가기 버튼을 누르면 자동으로 이전 화면으로 돌아가. SwiftUI가 path 배열을 보고 스택을 관리해주거든! 👏

## NavigationPathOption의 장점 정리

### 1. 타입 안전성 보장

```swift
// ❌ 잘못된 사용 - 컴파일 에러!
path.append(.chat(avatarId: 123)) // String이 필요한데 Int를 넣음

// ✅ 올바른 사용
path.append(.chat(avatarId: "avatar-123"))
```

컴파일러가 타입을 체크해주니까 런타임 에러가 줄어들어.

### 2. 중복 코드 제거

```swift
// ❌ 기존 방식: 모든 화면에서 반복
.navigationDestination(for: String.self) { avatarId in
    ChatView(avatarId: avatarId)
}

// ✅ 새로운 방식: 한 번만 정의
.navigationDestinationForCoreModule(path: $path)
```

### 3. 유지보수성 향상

ChatView 생성자가 바뀌어도 한 곳만 수정하면 돼:

```swift
extension View {
    func navigationDestinationForCoreModule(...) -> some View {
        self.navigationDestination(for: NavigationPathOption.self) { option in
            switch option {
            case .chat(avatarId: let avatarId):
                // ✅ 여기만 수정하면 모든 곳에 반영됨
                ChatView(
                    avatarId: avatarId,
                    initialMessage: "Hello!" // 새 파라미터 추가
                )
            // ...
            }
        }
    }
}
```

### 4. 쉬운 확장

새로운 화면을 추가하려면 Enum에 케이스만 추가하면 끝:

```swift
enum NavigationPathOption: Hashable {
    case chat(avatarId: String)
    case category(category: CharacterOption, imageName: String)
    case settings // ✅ 새 화면 추가
    case profile(userId: String) // ✅ 또 다른 화면
}

extension View {
    func navigationDestinationForCoreModule(
        path: Binding<[NavigationPathOption]> // ← 이 path를 받아서
    ) -> some View {
        self.navigationDestination(for: NavigationPathOption.self) { option in
            switch option {
            case .chat(avatarId: let avatarId):
                ChatView(avatarId: avatarId)
            case .category(category: let category, imageName: let imageName):
                // ✅ 중첩 네비게이션을 위해 path를 그대로 전달
                CategoryListView(path: path, category: category, imageName: imageName)
            case .settings:
                SettingsView()
            case .profile(userId: let userId):
                ProfileDetailView(userId: userId)
            }
        }
    }
}
```

Switch 문이 모든 케이스를 처리하지 않으면 컴파일 에러가 나니까 빠뜨릴 일이 없어! 🛡️

### 5. Programmatic Navigation 간편화

Path를 직접 조작할 수 있어서 복잡한 네비게이션도 가능해:

```swift
// 특정 화면으로 바로 이동
path = [.category(category: .alien, imageName: "alien.png")]

// 여러 단계 한번에 이동
path = [
    .category(category: .robot, imageName: "robot.png"),
    .chat(avatarId: "robot-123")
]

// 루트로 돌아가기
path = []

// 한 단계 뒤로
path.removeLast()
```

이런 식으로 네비게이션 스택을 프로그래밍 방식으로 제어할 수 있어. Deep linking 구현할 때 특히 유용해!

## 심화: Deep Linking과 URL 매핑

이 패턴의 진짜 파워는 **Deep Linking**을 구현할 때 드러나. URL을 NavigationPathOption으로 변환하면 외부 링크로 앱의 특정 화면을 열 수 있어:

```swift
extension NavigationPathOption {

    // URL → NavigationPathOption 변환
    static func from(url: URL) -> NavigationPathOption? {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let host = components.host else {
            return nil
        }

        switch host {
        case "chat":
            // myapp://chat?avatarId=123
            if let avatarId = components.queryItems?.first(where: { $0.name == "avatarId" })?.value {
                return .chat(avatarId: avatarId)
            }

        case "category":
            // myapp://category?name=alien&image=alien.png
            if let categoryName = components.queryItems?.first(where: { $0.name == "name" })?.value,
               let imageName = components.queryItems?.first(where: { $0.name == "image" })?.value,
               let category = CharacterOption.from(string: categoryName) {
                return .category(category: category, imageName: imageName)
            }

        default:
            return nil
        }

        return nil
    }
}
```

그러면 앱에서 이렇게 사용할 수 있어:

```swift
// 간단한 예시: 단일 NavigationStack 구조
struct SimpleAppView: View {
    @State private var path: [NavigationPathOption] = []

    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestinationForCoreModule(path: $path)
        }
        .onOpenURL { url in
            // ✅ URL을 받아서 path로 변환
            if let option = NavigationPathOption.from(url: url) {
                path.append(option)
            }
        }
    }
}
```

만약 **TabView 구조**라면 각 탭별로 path를 관리해야 해:

```swift
struct ContentView: View {
    @State private var selectedTab: Tab = .explore
    // 각 탭별 path 관리
    @State private var explorePath: [NavigationPathOption] = []
    @State private var chatsPath: [NavigationPathOption] = []
    @State private var profilePath: [NavigationPathOption] = []

    var body: some View {
        TabView(selection: $selectedTab) {
            // ⚠️ 주의: 각 탭 루트에서 NavigationStack을 만들면,
            // 각 View 내부의 NavigationStack은 제거해야 해 (중첩 방지)
            // ExploreView → ExploreContentView처럼 분리하거나,
            // 또는 각 View에서 NavigationStack을 제거하고 content만 남겨야 해

            NavigationStack(path: $explorePath) {
                ExploreContentView() // ← 내부에 NavigationStack 없는 버전
                    .navigationDestinationForCoreModule(path: $explorePath)
            }
            .tag(Tab.explore)

            NavigationStack(path: $chatsPath) {
                ChatsContentView()
                    .navigationDestinationForCoreModule(path: $chatsPath)
            }
            .tag(Tab.chats)

            NavigationStack(path: $profilePath) {
                ProfileContentView()
                    .navigationDestinationForCoreModule(path: $profilePath)
            }
            .tag(Tab.profile)
        }
        .onOpenURL { url in
            handleDeepLink(url)
        }
    }

    private func handleDeepLink(_ url: URL) {
        guard let option = NavigationPathOption.from(url: url) else { return }

        // URL에 따라 적절한 탭으로 이동하고 path 설정
        switch option {
        case .chat:
            selectedTab = .chats
            chatsPath.append(option)
        case .category:
            selectedTab = .explore
            explorePath.append(option)
        }
    }
}

enum Tab {
    case explore
    case chats
    case profile
}
```

이렇게 하면 **탭별로 독립적인 네비게이션 스택**을 유지하면서, Deep link로 특정 탭의 특정 화면으로 바로 이동할 수 있어! 🚀

## 마치며

Enum 기반 NavigationPathOption 패턴은 SwiftUI 네비게이션을 **타입 안전하고 확장 가능하게** 만들어줘.

### 핵심 포인트

1. **Enum으로 모든 네비게이션 경로 정의** - 타입 안전성 확보
2. **View extension으로 중앙 집중식 관리** - 중복 제거, 유지보수 용이
3. **Path 배열로 스택 제어** - Programmatic navigation, Deep linking 가능
4. **Hashable 프로토콜 준수** - NavigationStack 요구사항 충족

처음에는 보일러플레이트가 좀 생기는 것처럼 보일 수 있지만, 앱이 커질수록 이 패턴의 진가가 드러나. 특히:

- 새로운 화면 추가할 때
- 네비게이션 로직 수정할 때
- Deep linking 구현할 때
- 복잡한 네비게이션 플로우 관리할 때

이런 상황에서 엄청난 시간을 절약해줘.

SwiftUI는 선언형 프로그래밍이잖아. 네비게이션도 "어떻게"가 아니라 "무엇을"에 집중하면 훨씬 깔끔한 코드가 나와. NavigationPathOption 패턴은 바로 그런 사고방식의 결과물이야! 💪

다음에는 NavigationPath를 활용한 타입 지워진(Type-Erased) 네비게이션 관리 방법에 대해서도 알아보자. 여러 타입이 섞인 복잡한 네비게이션 스택을 어떻게 관리하는지 궁금하지? 😉
