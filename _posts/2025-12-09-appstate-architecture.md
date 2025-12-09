---
title: 3.AppState 아키텍처 - 앱 전역 상태 관리 설계
date: 2025-12-09 23:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - @Environment
pin: false
---

## 들어가며

앱의 루트 뷰 구조를 잡았다면, 다음 단계는 **"사용자가 로그인했는지 안했는지"** 같은 **앱 전역 상태**를 관리하는 것이다.

이번 커밋에서는 `AppState`라는 상태 관리 객체를 도입해서, 온보딩 화면과 메인 화면(TabBar) 사이의 전환을 제어하는 아키텍처를 구성했다.

## 아키텍처 개요

### 전체 구조

```
AIChatCourseApp (엔트리 포인트)
    ↓
AppView (루트 뷰 + AppState 주입)
    ├── .environment(appState) ← 상태 주입
    │
    ├── showTabBar = true  → TabBarView
    │   ├── ExploreView
    │   ├── ChatsView
    │   └── ProfileView
    │       └── SettingsView (로그아웃)
    │
    └── showTabBar = false → WelcomeView (온보딩)
        └── OnboardingCompletedView (온보딩 완료)
```

### 핵심 컴포넌트

1. **AppState**: 앱 전역 상태 관리 객체
2. **AppView**: 루트 뷰, AppState를 하위 모든 뷰에 주입
3. **WelcomeView → OnboardingCompletedView**: 온보딩 플로우
4. **ProfileView → SettingsView**: 로그아웃 기능

## AppState의 역할

AppState는 **"사용자가 온보딩을 완료했는지"**라는 상태를 저장한다.

```swift
@Observable
class AppState {
    private(set) var showTabBar: Bool

    func updateViewState(showTabBarView: Bool) {
        showTabBar = showTabBarView
    }
}
```

- `showTabBar`: 탭바를 보여줄지 여부 (= 로그인 상태)
- `updateViewState()`: 상태 업데이트는 이 메서드를 통해서만 가능
- `private(set)`: 외부에서는 읽기만 가능, 임의 수정 불가


앱을 껐다 켜도 상태가 유지되도록 UserDefaults와 연동:

```swift
private(set) var showTabBar: Bool {
    didSet {
        UserDefaults.showTabbarView = showTabBar
    }
}

init(showTabBar: Bool = UserDefaults.standard.bool(forKey: "showTabbarView")) {
    self.showTabBar = showTabBar
}
```

**데이터 플로우:**
1. 앱 시작 → UserDefaults에서 상태 읽기
2. 사용자 액션 → `updateViewState()` 호출
3. `didSet` 트리거 → UserDefaults에 자동 저장
4. 앱 재시작 → 저장된 상태로 복원

## 의존성 주입 패턴

### Environment를 통한 상태 전파

AppState는 **Environment**를 통해 앱 전체에 주입된다:

```swift
// AppView.swift
struct AppView: View {
    @State var appState: AppState = AppState()

    var body: some View {
        AppViewBuilder(...)
            .environment(appState)  // ← 주입 지점
    }
}
```

이렇게 주입하면, 뷰 계층 구조 어디에서든 AppState에 접근 가능:

```swift
// OnboardingCompletedView.swift (3단계 아래)
@Environment(AppState.self) private var root

func onFinishButtonPressed() {
    root.updateViewState(showTabBarView: true)  // ✅ 접근 가능
}

// SettingsView.swift (ProfileView의 Sheet 안)
@Environment(AppState.self) private var appState

func onSignOutPressed() {
    appState.updateViewState(showTabBarView: false)  // ✅ 접근 가능
}
```

### 장점

1. **Props Drilling 방지**: 중간 뷰들이 상태를 전달할 필요 없음
2. **느슨한 결합**: 각 뷰는 AppState만 알면 되고, 다른 뷰와 독립적
3. **테스트 용이성**: Mock AppState를 주입해서 테스트 가능

```swift
#Preview {
    OnboardingCompletedView()
        .environment(AppState(showTabBar: false))  // Mock 주입
}
```

이번 강의에서는 딱히 개념적으로 짚고 넘어가야될 부분은 보이지않아서 아키텍처 수준에서 앱구조를 어떻게 잡았는지에 대해서만 정리하였다. 대신, @Environment, @State등이 슬슬 등장하는김에 `property wrapper`에 대해 잠깐 정리하고 넘어가려고 한다.