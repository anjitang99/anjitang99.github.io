---
layout: post
title: "Swift 기초 학습 - 옵셔널과 옵셔널 바인딩"
date: 2025-01-13 20:30:00 +0900
categories: swift ios development
---

## Swift를 배우는 이유

금융과 IT를 융합한 Fintech 앱을 만들고 싶어서 Swift를 공부하기 시작했습니다.
오늘은 Swift의 가장 중요한 개념 중 하나인 **옵셔널(Optional)**에 대해 정리합니다.

### 🤔 옵셔널이란?

옵셔널은 "값이 있을 수도, 없을 수도 있다"는 것을 나타내는 타입입니다.

```swift
var userName: String? = "도언"
var userAge: Int? = nil
```

물음표(`?`)가 붙으면 옵셔널 타입이 됩니다.

### 📦 옵셔널 언래핑 방법

#### 1. 강제 언래핑 (Forced Unwrapping)

```swift
let name: String? = "박도언"
print(name!)  // "박도언"
```

⚠️ **주의**: 값이 `nil`일 때 강제 언래핑하면 런타임 에러 발생!

#### 2. 옵셔널 바인딩 (Optional Binding) ✅

가장 안전한 방법입니다:

```swift
if let name = userName {
    print("안녕하세요, \(name)님!")
} else {
    print("이름이 없습니다.")
}
```

#### 3. 가드문 (Guard Statement)

```swift
func greet(name: String?) {
    guard let validName = name else {
        print("이름이 필요합니다.")
        return
    }
    print("안녕하세요, \(validName)님!")
}
```

#### 4. 닐 코얼레싱 (Nil Coalescing)

```swift
let displayName = userName ?? "게스트"
// userName이 nil이면 "게스트" 사용
```

### 💻 실전 예제: 금융 앱 시뮬레이션

```swift
struct BankAccount {
    var accountNumber: String
    var balance: Double?

    func displayBalance() {
        if let balance = balance {
            print("잔액: \(balance)원")
        } else {
            print("잔액 정보를 불러올 수 없습니다.")
        }
    }
}

var myAccount = BankAccount(
    accountNumber: "123-456-789",
    balance: 1_000_000
)

myAccount.displayBalance()  // "잔액: 1000000.0원"
```

### 🎓 오늘 배운 교훈

1. **옵셔널은 안전성을 위한 Swift의 핵심 기능**
2. **강제 언래핑은 가급적 피하기**
3. **옵셔널 바인딩이나 가드문 사용이 베스트 프랙티스**

### 📚 다음 학습 목표

- 클로저(Closure) 개념 이해하기
- SwiftUI로 간단한 UI 만들기
- API 통신 기초 배우기

---

> 💡 **TIP**: Xcode Playground를 사용하면 코드를 바로바로 테스트할 수 있어요!

**#Swift #iOS개발 #Fintech #학습일지**
