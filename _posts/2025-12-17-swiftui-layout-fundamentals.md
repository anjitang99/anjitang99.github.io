---
title: 4.SwiftUI 레이아웃 기초 - 부모와 자식의 협상
date: 2025-12-17 20:00:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - Layout
  - SwiftfulThinking
pin: false
---

## 들어가며

SwiftUI 4번째 포스트를 하기전에 주제를 무엇으로 해야될지 굉장히 애먹었다. 본격 UI 파트로 넘어가는데 출신이 백엔드라 그런지 왠지 UI는 자신도 없고 괜히 공부를 계속 미루게 된달까... 사실 layout에 대한 기본기 없이 대충 따라치고 나중에 하자라는 마인드가 컸던거같기도 하다 ㅋㅋ 그럼 이젠 더이상 미룰수 없는 SwiftUI의 UI 기초! 시작해보자.

SwiftUI로 UI를 만들다 보면 이런 경험을 하게 된다:

- "왜 padding을 주면 원의 크기가 작아지지?"
- "ScrollView에 큰 뷰를 넣으면 스크롤이 안 돼"
- "왜 내가 준 크기대로 안 나와?"

처음엔 이게 정말 답답했다. UIKit에서는 Constraint 잡으면 그대로 나왔는데... SwiftUI는 왜 이렇게 제멋대로일까? 🤔

근데 알고 보니 제멋대로가 아니었다. **SwiftUI만의 레이아웃 철학**이 있었던 것. 이번 포스트에서는 SwiftUI 레이아웃의 핵심 원리를 파헤쳐보려고 한다.

---

## Part 1: SwiftUI 레이아웃의 핵심 원리

### 레이아웃은 '협상'이다

SwiftUI의 레이아웃은 UIKit과 다르다. Constraint를 설정하는 게 아니라, **부모와 자식이 대화하며 크기를 정한다**.

이 대화는 정확히 3단계로 이루어진다:

```
1. 부모 → 자식: "너 이만큼 공간 쓸 수 있어"  (Parent proposes a size)
2. 자식 → 부모: "나는 이만큼만 쓸게요"      (Child chooses its size)
3. 부모: "그럼 여기에 배치할게"              (Parent positions the child)
```

이건 내가 막 만든 비유가 아니라, **[WWDC19 - Building Custom Views with SwiftUI](https://developer.apple.com/videos/play/wwdc2019/237/)** 세션에서 애플이 공식적으로 설명하는 방식이다.

SwiftUI의 핵심 철학이 하나 있다:
> "In SwiftUI there's no way to force a size on your child, **the parent has to respect that choice**"

부모가 자식에게 크기를 강제할 수 없고, 자식이 선택한 크기를 존중해야 한다는 것. 이게 UIKit과의 가장 큰 차이점이다.

### 실제로 보기

```swift
VStack {
    Text("Hello")
    Circle()
}
.frame(width: 300, height: 400)
```

**1단계: 부모의 제안**
```
VStack이 300×400 크기를 받음
↓
Text에게: "너 300×200까지 써도 돼"
Circle에게: "너도 300×200까지 써도 돼"
```

**2단계: 자식의 응답**
```
Text("Hello"): "저는 60×20만 필요해요"
Circle(): "저는 정사각형이라 200×200 쓸게요"
```

**3단계: 부모의 배치**
```
VStack:
- Text를 60×20 크기로 배치
- Circle을 200×200 크기로 배치
- 남은 공간은 Spacer처럼 분배
```

이게 SwiftUI 레이아웃의 전부다. 모든 것이 이 3단계로 동작한다.

---

## Part 2: 실전에서 만나는 문제들

### 문제 1: padding을 주면 왜 안의 뷰가 작아질까?

```swift
LazyVGrid(
    columns: Array(repeating: GridItem(.flexible()), count: 3)
) {
    ForEach(1...9, id: \.self) { _ in
        Circle()
    }
}
.padding(.horizontal, 24)
```

**왜 Circle이 작아질까?**

3단계 원리로 분석해보자:

**padding 없을 때:**
```
1. ScrollView → LazyVGrid: "390pt 너비 써"
2. LazyVGrid: "3개 컬럼, 간격 16pt씩..."
   각 Circle: (390 - 32) ÷ 3 = 119pt
```

**padding 있을 때:**
```
1. ScrollView → padding: "390pt 너비 써"
2. padding → LazyVGrid: "좌우 24pt 빼고... 342pt 써"
3. LazyVGrid: "3개 컬럼, 간격 16pt씩..."
   각 Circle: (342 - 32) ÷ 3 = 103pt
```

**결론:** padding이 부모의 제안 크기를 줄였고, LazyVGrid는 그 안에서 공간을 분배했다.

### 문제 2: ScrollView에 큰 뷰를 넣었는데 스크롤이 안 돼

```swift
ScrollView {  // 세로 스크롤
    Rectangle()
        .frame(width: 500)  // 화면보다 넓음
}
```

이건 스크롤이 안 된다. **왜일까?**

ScrollView의 비밀은 **방향에 따라 제안이 다르다**는 것이다:

```swift
ScrollView(.vertical) {
    // 세로 방향: "무제한으로 써도 돼" ✅
    // 가로 방향: "부모 크기만큼만 써" ⚠️
}

ScrollView(.horizontal) {
    // 가로 방향: "무제한으로 써도 돼" ✅
    // 세로 방향: "부모 크기만큼만 써" ⚠️
}
```

그래서 세로 ScrollView에 `width: 500`을 주면:
```
1. ScrollView → Rectangle: "가로는 390pt만, 세로는 무제한"
2. Rectangle: "width: 500을 원하지만... 390pt만 받음"
```

**올바른 사용:**
```swift
// 세로 스크롤이 필요하면 height를 크게
ScrollView(.vertical) {
    Rectangle()
        .frame(height: 2000)  // ✅
}

// 가로 스크롤이 필요하면 width를 크게
ScrollView(.horizontal) {
    Rectangle()
        .frame(width: 2000)  // ✅
}
```

---

## Part 3: 자식이 고집부리면?

때로는 자식이 부모의 제안보다 **더 큰 크기**를 원할 수 있다.

### 허용하는 부모들

**ScrollView:** "더 크게? 좋아, 스크롤할게"
```swift
ScrollView {
    VStack {
        ForEach(1...100, id: \.self) { i in
            Text("Item \(i)")
        }
    }
    // 화면보다 훨씬 커도 OK → 스크롤 가능
}
```

**ZStack:** "더 크게? 좋아, 잘릴 뿐"
```swift
ZStack {
    Circle()
        .frame(width: 1000)
    // 화면보다 커도 OK → 화면 밖으로 잘림
}
```

### 제한하는 부모들

**VStack, HStack:** "어떻게든 맞출게... 압축!"
```swift
HStack {
    Rectangle().frame(width: 300)
    Rectangle().frame(width: 300)
}
.frame(width: 400)
// 600pt를 요구하지만 400pt만 있음
// → 각각 200pt로 압축
```

**해결 방법:**
```swift
ScrollView(.horizontal) {  // ScrollView로 감싸기
    HStack {
        Rectangle().frame(width: 300)
        Rectangle().frame(width: 300)
    }
}
// → 가로 스크롤 가능
```

---

## Part 4: 크기를 다루는 도구들

이제 크기를 조절하는 주요 도구들을 알아보자.

### frame: "이만큼 쓰세요"

```swift
// 정확한 크기
Text("Hi")
    .frame(width: 200, height: 100)
// → 200×100 공간 차지

// 최대한 확장
Text("Hi")
    .frame(maxWidth: .infinity)
// → 부모가 주는 만큼 다 사용

// 범위 지정
Text("Hi")
    .frame(minWidth: 100, maxWidth: 200)
// → 100~200pt 사이에서 결정
```

### padding: "여백을 추가해요"

```swift
Text("Hi")
    .padding(20)
// → Text 주위에 20pt 공간 추가
// → 레이아웃 공간이 커짐
```

### offset: "시각적으로만 이동해요"

```swift
VStack {
    Text("1")
    Text("2").offset(y: 20)  // 아래로 이동
    Text("3")  // 원래 위치 (안 밀림)
}
```

offset은 **레이아웃에 영향을 주지 않는다**. 시각적으로만 이동한다.

**비교:**
```swift
// padding: 레이아웃에 영향 O
VStack {
    Text("1")
    Text("2").padding(.top, 20)  // 아래로 이동
    Text("3")  // 같이 밀림 ✅
}

// offset: 레이아웃에 영향 X
VStack {
    Text("1")
    Text("2").offset(y: 20)  // 아래로 이동
    Text("3")  // 안 밀림 ⚠️
}
```

---

## Part 5: GridItem의 크기 전략

LazyVGrid/LazyHGrid를 사용할 때 GridItem의 크기 타입을 이해해야 한다.

### .flexible() - "주는 만큼 다 쓸게요"

```swift
GridItem(.flexible())
```
- 부모가 제안한 크기를 **최대한 사용**
- 여러 개 있으면 **균등 분배**

```swift
LazyVGrid(
    columns: Array(repeating: GridItem(.flexible()), count: 3)
) {
    // 3개 컬럼이 화면을 균등 분배
}
```

### .fixed(100) - "무조건 100pt"

```swift
GridItem(.fixed(100))
```
- 항상 100pt 고정
- 부모 제안 무시

### .adaptive(minimum: 100) - "최소 100pt 유지하며 최대한 많이"

```swift
LazyVGrid(
    columns: [GridItem(.adaptive(minimum: 100))]
) {
    // 390pt 화면: 3개 컬럼 (각 130pt)
    // 768pt 화면: 7개 컬럼 (각 109pt)
}
```

가장 유연한 방식이다. 화면 크기에 따라 컬럼 개수가 자동으로 조정된다.

---

## Part 6: 디버깅 팁

레이아웃이 예상과 다를 때 유용한 기법들을 정리했다:

### 1. border로 크기 확인
```swift
Text("Hello")
    .border(Color.red)
    .padding()
    .border(Color.blue)
```
각 단계의 크기를 시각적으로 확인할 수 있다.

### 2. background로 영역 확인
```swift
Text("Hello")
    .background(Color.red.opacity(0.3))
```

### 3. fixedSize로 압축 방지
```swift
Text("긴 텍스트가 잘리지 않게")
    .fixedSize()
```
부모의 제안을 무시하고 자신의 크기를 유지한다.

### 4. layoutPriority로 우선순위 지정
```swift
HStack {
    Text("중요한 긴 텍스트")
        .layoutPriority(1)
    Text("압축 가능")
}
```
공간이 부족할 때 우선순위가 낮은 뷰부터 압축된다.

---

## 마치며

SwiftUI의 레이아웃은 결국 **대화**다.

```
부모: "이만큼 줄 수 있어"
자식: "나는 이만큼 쓸게"
부모: "알겠어, 여기 배치할게"
```

이 간단한 원칙만 기억하면:
- padding이 왜 안의 뷰를 작게 만드는지
- ScrollView가 왜 특정 방향만 무제한인지
- Grid의 아이템이 왜 그런 크기를 갖는지

모두 명쾌하게 이해할 수 있다.

처음에는 UIKit의 Constraint가 그리웠지만, 지금은 SwiftUI의 이 협상 시스템이 더 직관적으로 느껴진다. 부모와 자식이 대화한다는 개념만 잡으면, 복잡한 레이아웃도 머릿속에서 시뮬레이션이 가능하다.

이제 레이아웃 문제를 만나면 당황하지 말고, 3단계 원리로 차근차근 생각해보자.

"부모가 뭘 제안했고, 자식이 뭘 선택했나?"
