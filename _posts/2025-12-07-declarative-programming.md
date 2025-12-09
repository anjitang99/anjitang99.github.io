---
title: 2.SwiftUI vs UIKit - declarative 관점에서
date: 2025-12-07 18:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - UIKit
  - 선언형프로그래밍
  - 명령형프로그래밍
pin: false
---

## 들어가며

2번째 강의는 NavigationStacks을 구성하는 강의였다. 사실 코드가 많다거나 내용이 많지는 않아 3강을 다 들은후에 내용을 합쳐서 블로그에 올려야 되나 고민하던 찰나 내 귀에 꽂혀버린 한문장이 있었으니...

> "Everything is dependent on the parents before it. That is the entirety of declarative programming."

이 한 문장이 SwiftUI와 UIKit의 근본적인 차이를 설명하고 있었다. 오늘은 재미없지만 알아야될 SwiftUI의 철학의 일부에 대해 알아본다.

## 선언형 프로그래밍이란?

**"모든 것은 그 위(부모)의 상태에 의존한다. 그게 선언형 프로그래밍의 전부다."**

쉽게 말하면:
- **무엇(What)**을 보여줄지만 선언하면
- **어떻게(How)** 보여줄지는 프레임워크가 알아서 처리

## 명령형 vs 선언형: 실제 비교

### UIKit - 명령형 프로그래밍

```swift
class LoginViewController: UIViewController {
    let loginButton = UIButton()
    let label = UILabel()
    let profileView = UIView()

    var isLoggedIn = false

    @objc func buttonTapped() {
        // 1단계: 상태 변경
        isLoggedIn = true

        // 2단계: UI를 "어떻게" 바꿀지 일일이 명령
        label.text = "로그인됨"
        label.textColor = .green
        loginButton.isHidden = true
        profileView.isHidden = false

        // 3단계: 애니메이션도 직접 제어
        UIView.animate(withDuration: 0.3) {
            self.view.layoutIfNeeded()
        }
    }
}
```

**명령형의 특징:**
- 상태 변경 + UI 업데이트 코드를 **모두 작성**
- "이거 숨기고, 저거 보이고, 색깔 바꾸고..." 일일이 명령

### SwiftUI - 선언형 프로그래밍

```swift
struct LoginView: View {
    @State private var isLoggedIn = false

    var body: some View {
        VStack {
            if isLoggedIn {
                Text("로그인됨")
                    .foregroundColor(.green)
                ProfileView()
            } else {
                Button("로그인") {
                    isLoggedIn = true  // 상태만 변경!
                }
            }
        }
    }
}
```

**선언형의 특징:**
- **"무엇"**을 보여줄지만 선언
- 상태가 바뀌면 UI는 **자동으로** 업데이트
- 애니메이션까지 자동 처리

## 부모-자식 의존성

선언형의 핵심은 **부모의 상태 변화가 자식에게 자동으로 전파**된다는 것이다.

### SwiftUI에서의 자동 전파

```swift
struct ParentView: View {
    @State private var userName = "지훈"

    var body: some View {
        VStack {
            ChildView(name: userName)

            Button("이름 변경") {
                userName = "안지훈"
                // ChildView가 자동으로 업데이트됨!
            }
        }
    }
}

struct ChildView: View {
    let name: String

    var body: some View {
        Text("안녕, \(name)")
        // 부모의 userName이 바뀌면 자동으로 다시 렌더링
    }
}
```

### UIKit에서의 수동 전파

```swift
class ParentViewController: UIViewController {
    var userName: String = "지훈" {
        didSet {
            // 수동으로 자식에게 알려야 함!
            childVC?.updateName(userName)
        }
    }

    var childVC: ChildViewController?
}

class ChildViewController: UIViewController {
    let label = UILabel()

    func updateName(_ name: String) {
        // 수동으로 UI 업데이트
        label.text = "안녕, \(name)"
    }
}
```

## 복잡도 비교

여러 상태를 관리할 때 차이가 극명하게 드러난다.

### SwiftUI - 간결함

```swift
@State var isLoggedIn = false
@State var hasNotification = false
@State var userName = ""

var body: some View {
    VStack {
        if isLoggedIn {
            Text("안녕, \(userName)")

            if hasNotification {
                NotificationBadge()
            }
        } else {
            LoginButton()
        }
    }
}
// 3개 상태 중 뭐가 바뀌든 자동 업데이트!
```

### UIKit - 복잡함

```swift
var isLoggedIn = false { didSet { updateUI() } }
var hasNotification = false { didSet { updateUI() } }
var userName = "" { didSet { updateUI() } }

func updateUI() {
    // 모든 상태 조합을 고려해야 함
    if isLoggedIn {
        nameLabel.text = userName
        nameLabel.isHidden = false

        if hasNotification {
            badgeView.isHidden = false
        } else {
            badgeView.isHidden = true
        }
    } else {
        nameLabel.isHidden = true
        badgeView.isHidden = true
        loginButton.isHidden = false
    }

    view.layoutIfNeeded()
}
```

## UIKit도 부모-자식 패턴을 만들 수 있지 않나?

처음에 이런 의문이 들었다. UIKit도 뷰안에 뷰를 생성하면서 부모-자식 관계를 만들 수 있지 않을까?

**답은 "만들 수 있다"이다.**

하지만 핵심적인 차이가 있다:

| 구분 | UIKit | SwiftUI |
|------|-------|---------|
| 패턴 구현 | 개발자가 **직접 구현** | 프레임워크가 **기본 제공** |
| 업데이트 | 수동으로 **명령** | 자동으로 **반영** |
| 복잡도 | 상태 증가 시 **기하급수적 증가** | 상태 증가 시 **선형적 증가** |

## 실전 예시: 로그인 화면

두 방식의 차이를 실전 코드로 비교해보자.

### UIKit 버전

```swift
class LoginViewController: UIViewController {
    let usernameField = UITextField()
    let passwordField = UITextField()
    let loginButton = UIButton()
    let errorLabel = UILabel()
    let loadingSpinner = UIActivityIndicatorView()

    var isLoading = false {
        didSet { updateLoadingState() }
    }

    var hasError = false {
        didSet { updateErrorState() }
    }

    func updateLoadingState() {
        if isLoading {
            loadingSpinner.startAnimating()
            loginButton.isEnabled = false
            usernameField.isEnabled = false
            passwordField.isEnabled = false
        } else {
            loadingSpinner.stopAnimating()
            loginButton.isEnabled = true
            usernameField.isEnabled = true
            passwordField.isEnabled = true
        }
    }

    func updateErrorState() {
        errorLabel.isHidden = !hasError
        if hasError {
            errorLabel.text = "로그인 실패"
            errorLabel.textColor = .red
        }
    }
}
```

### SwiftUI 버전

```swift
struct LoginView: View {
    @State private var username = ""
    @State private var password = ""
    @State private var isLoading = false
    @State private var hasError = false

    var body: some View {
        VStack {
            TextField("사용자명", text: $username)
                .disabled(isLoading)

            SecureField("비밀번호", text: $password)
                .disabled(isLoading)

            if isLoading {
                ProgressView()
            } else {
                Button("로그인") {
                    login()
                }
            }

            if hasError {
                Text("로그인 실패")
                    .foregroundColor(.red)
            }
        }
    }

    func login() {
        isLoading = true
        // 로그인 로직
    }
}
```

**코드량 차이:**
- UIKit: 약 40줄 + 복잡한 상태 관리
- SwiftUI: 약 25줄 + 명확한 구조

## 다른  프레임워크에서는?

강사님이 말씀하신 **"Everything is dependent on the parents before it"**는 사실 SwiftUI뿐만 아니라 현대의 모든 선언형 UI 프레임워크를 관통하는 핵심 원칙이다. 이를 보통 **단방향 데이터 흐름(One-Way Data Flow)**이라고 부른다.

다른 프레임워크들도 용어만 다를 뿐, 같은 철학을 공유하고 있다.

### React (Web)
- **SwiftUI:** `@State` → `View`
- **React:** `useState` → `Component`

**결국 핵심은 같다:**
1. 진실의 원천(Source of Truth)은 하나여야 한다. (부모가 관리)
2. 데이터는 위에서 아래로만 흘러야 추적하기 쉽다. (단방향 흐름)
3. 자식은 "상태를 가진 주체"가 아니라 "데이터를 보여주는 껍데기"여야 한다. (순수 함수)

## 마치며

선언형 프로그래밍의 본질은 결국 **"상태와 UI의 자동 동기화"**다.

UIKit도 물론 부모-자식 패턴을 만들 수 있다. 하지만 SwiftUI는 이것이 프레임워크의 기본 동작이다. 개발자는 **"무엇을 보여줄지"**만 선언하면 되고, **"어떻게 업데이트할지"**는 프레임워크가 처리한다.

강사님의 한 문장:
> "Everything is dependent on the parents before it."

이제 이 말의 의미를 완전히 이해할 수 있다. 자식 View는 부모의 상태에 의존하고, 부모의 상태가 바뀌면 자식은 자동으로 그 변화를 반영한다. 그동안 따라치기 바빳던 코드에 구조가 조금 더 명확히 보인것 같다.

## 참고 자료

- [Swiftful Thinking](https://www.swiftful-thinking.com/)
- [Apple - SwiftUI Documentation](https://developer.apple.com/documentation/swiftui/)
- [WWDC - Introduction to SwiftUI](https://developer.apple.com/videos/play/wwdc2019/204/)
