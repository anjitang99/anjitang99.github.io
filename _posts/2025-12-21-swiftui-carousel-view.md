---
title: 5.SwiftUI로 재사용 가능한 CarouselView 만들기 - Generic과 ViewBuilder 활용
date: 2025-12-21 14:04:00 +0900
categories:
  - iOS
  - SwiftUI
tags:
  - SwiftUI
  - Carousel
  - Generic
  - ViewBuilder
  - ScrollView
  - Paging
pin: false
---

## 들어가며

이번 포스트에서는 **재사용 가능한 CarouselView**를 만들면서 SwiftUI의 `Generic`, `ViewBuilder`, 그리고 최신 스크롤 API들을 어떻게 활용했는지 정리해보려고 한다. 사실 `Generic`, `ViewBuilder`를 활용하는 방안은 1강에서 다뤘는데 복습할겸 다른 예제로 한번 더 확인해보겠다.

## CarouselView이 뭐임?

### 캐러셀이란?

**Carousel(캐러셀)**은 여러 아이템을 수평으로 스와이프하면서 볼 수 있는 UI 패턴이다.
보통 앱 온보딩, 상품 목록, 이미지 갤러리 등에서 많이 사용된다.

**핵심 기능**:
- 수평 스크롤
- 페이징 (한 아이템씩 딱딱 넘어감)
- 페이지 인디케이터 (하단 점 표시)
- 부드러운 애니메이션

> SwiftUI의 선언형 방식으로 재사용 가능한 컴포넌트를 만들 수 있다! 🎉
{: .prompt-tip }

## CarouselView 구조 분석

CarouselView는 크게 3가지 요소로 구성된다:
1. **Generic 타입 정의** - 재사용 가능한 구조
2. **ScrollView + 페이징** - 수평 스크롤과 스냅 효과
3. **페이지 인디케이터** - 현재 위치 표시

이제 각 핵심 개념을 하나씩 살펴보자 🔍

## 핵심 개념 1: Generic과 ViewBuilder

### Generic 타입 파라미터

```swift
struct CarouselView<Content: View, T: Hashable>: View {
    var items: [T]
    @ViewBuilder var content: (T) -> Content
}
```

**Generic을 사용하는 이유**:
- `T: Hashable`: 어떤 타입의 배열이든 받을 수 있음
- `Content: View`: 어떤 뷰든 컨텐츠로 사용 가능
- **재사용성 극대화!**

> Generic 덕분에 한 번 만들어두면 어디서든 재사용 가능! 🚀
{: .prompt-info }

### @ViewBuilder - 컨텐츠 주입의 마법

```swift
@ViewBuilder var content: (T) -> Content
```

**@ViewBuilder란?**
- SwiftUI의 특별한 어트리뷰트
- 클로저 안에서 여러 뷰를 선언형으로 작성 가능
- 내부적으로 ViewBuilder DSL로 변환됨

**ViewBuilder의 장점**:

```swift
// ViewBuilder 사용
CarouselView(items: avatars) { avatar in
    HeroCellView(
        title: avatar.name,
        subtitle: avatar.description,
        imageName: avatar.image
    )
}

// 문자열 배열도 가능
CarouselView(items: ["Apple", "Banana", "Cherry"]) { fruit in
    Text(fruit)
        .font(.largeTitle)
        .foregroundColor(.white)
}

// Int 배열도 가능
CarouselView(items: [1, 2, 3, 4]) { number in
    Text("\(number)")
        .frame(maxWidth: .infinity)
        .background(Color.blue)
}
```

ViewBuilder 덕분에 **컨텐츠 타입에 상관없이** 동일한 CarouselView를 재사용할 수 있다!

## 핵심 개념 2: 최신 스크롤 API

### containerRelativeFrame - 페이징의 핵심

```swift
.containerRelativeFrame(.horizontal, alignment: .center)
```

**역할**:
- 각 아이템이 **컨테이너(ScrollView) 전체 너비**를 차지하도록 강제
- 페이징 효과를 위한 필수 요소

**동작 원리**:

```
[     아이템1     ] [     아이템2     ] [     아이템3     ]
 ← 화면 너비만큼 →  ← 화면 너비만큼 →  ← 화면 너비만큼 →
```

이게 없으면 각 아이템이 content 크기만큼만 차지해서 오밀조밀하게 붙어버린다.

> containerRelativeFrame = "컨테이너 기준으로 프레임을 잡아줘!"
{: .prompt-info }

### scrollTargetLayout + scrollTargetBehavior

```swift
.scrollTargetLayout()           // "이 뷰들을 스크롤 타겟으로 사용해!"
.scrollTargetBehavior(.paging)  // "페이징 방식으로 멈춰!"
```

**scrollTargetLayout()**:
- ForEach 안의 각 아이템을 **스크롤 멈춤 지점**으로 등록
- 일종의 앵커 포인트

**scrollTargetBehavior(.paging)**:
- 스크롤을 놓으면 가장 가까운 타겟으로 **스냅(snap)**
- UIScrollView의 `isPagingEnabled = true`와 동일

**함께 사용해야 하는 이유**:

```swift
// ❌ scrollTargetBehavior만 있으면
.scrollTargetBehavior(.paging)
// → "뭘 기준으로 페이징할지 모르겠는데?" → 작동 안 함

// ✅ 둘 다 있어야 완성
.scrollTargetLayout()
.scrollTargetBehavior(.paging)
// → "ForEach 아이템 기준으로 페이징!" → 완벽 작동
```

### scrollPosition - 현재 위치 추적

```swift
@State private var selection: T?

ScrollView(.horizontal) {
    // ...
}
.scrollPosition(id: $selection)
```

**양방향 바인딩**:
- **스크롤 → 상태**: 사용자가 스크롤하면 `selection` 자동 업데이트
- **상태 → 스크롤**: `selection`을 변경하면 스크롤 위치 이동

**실제 동작**:

```swift
// 사용자가 두 번째 아이템으로 스크롤
// → selection = items[1] (자동)
// → 하단 인디케이터 자동 업데이트!

Circle()
    .fill(item == selection ? .accent : .secondary.opacity(0.5))
    // ↑ selection이 바뀌면 색상 자동 변경
```

## 핵심 개념 3: 인터랙티브 애니메이션

### scrollTransition - 부드러운 전환 효과

```swift
.scrollTransition(
    .interactive.threshold(.visible(0.95)),
    transition: { content, phase in
        content
            .scaleEffect(phase.isIdentity ? 1 : 0.9)
    }
)
```

**역할**:
- 스크롤 진행도에 따라 뷰 변형
- `phase.isIdentity`: 뷰가 정중앙에 있으면 true

**동작 과정**:

```
왼쪽에서 스크롤 시작:
  scale = 0.9 (작음)
         ↓
  화면 중앙 도착:
  scale = 1.0 (원래 크기)
         ↓
  오른쪽으로 벗어남:
  scale = 0.9 (다시 작아짐)
```

이 효과 덕분에 현재 선택된 아이템이 확대되어 보인다! ✨

**threshold(.visible(0.95))**:
- 뷰의 95%가 보일 때부터 transition 시작
- 민감도 조절 가능

## 실전 사용 예제

### 아바타 선택 캐러셀

```swift
struct AvatarSelectionView: View {
    let avatars: [AvatarModel] = AvatarModel.mocks

    var body: some View {
        VStack {
            Text("아바타를 선택하세요")
                .font(.title)

            CarouselView(items: avatars) { avatar in
                HeroCellView(
                    title: avatar.name,
                    subtitle: avatar.characterDescription,
                    imageName: avatar.profileImageName
                )
            }
            .padding()
        }
    }
}
```

**HeroCellView 구조**:

```swift
struct HeroCellView: View {
    var title: String?
    var subtitle: String?
    var imageName: String?

    var body: some View {
        ZStack {
            // 배경 이미지
            if let imageName {
                ImagesLoaderView(urlString: imageName)
            } else {
                Rectangle().fill(.accent)
            }
        }
        .overlay(alignment: .bottomLeading) {
            VStack(alignment: .leading, spacing: 4) {
                if let title {
                    Text(title)
                        .font(.headline)
                }
                if let subtitle {
                    Text(subtitle)
                        .font(.subheadline)
                }
            }
            .foregroundStyle(.white)
            .padding(16)
            .frame(maxWidth: .infinity, alignment: .leading)
            .background(
                // 그라데이션 오버레이
                LinearGradient(
                    colors: [
                        Color.black.opacity(0),
                        Color.black.opacity(0.3),
                        Color.black.opacity(0.8)
                    ],
                    startPoint: .top,
                    endPoint: .bottom
                )
            )
        }
        .cornerRadius(16)
    }
}
```

## 주의사항과 함정 🚨

### 1. `.id(item)` 필수!

```swift
ForEach(items, id: \.self) { item in
    content(item)
        .containerRelativeFrame(.horizontal, alignment: .center)
        .id(item)  // ← 이거 없으면 scrollPosition 작동 안 함!
}
```

`.id(item)`이 없으면:
- `scrollPosition`이 현재 아이템을 식별 못함
- 페이지 인디케이터가 업데이트 안 됨
- 스크롤해도 `selection` 변화 없음

### 2. `T: Hashable` 제약

```swift
struct CarouselView<Content: View, T: Hashable>: View {
    //                                 ↑ 필수!
}
```

**이유**:
- `ForEach(items, id: \.self)`를 사용하려면 Hashable 필요
- Hashable이 아니면 `id: \.someProperty` 사용해야 함

### 3. spacing: 0 필수

```swift
LazyHStack(spacing: 0) {  // ← 반드시 0!
    ForEach(items, id: \.self) { item in
        // ...
    }
}
```

**이유**:
- `containerRelativeFrame`이 화면 전체 너비를 차지
- spacing이 있으면 아이템 사이 빈 공간 생김
- 페이징이 어긋남

> spacing > 0이면 페이징 위치가 어긋나서 이상한 곳에서 멈춘다! 😱
{: .prompt-warning }

## 마치며

오늘은 SwiftUI로 재사용 가능한 CarouselView를 만들어보았다.

**핵심 포인트 정리**:
1. **Generic + ViewBuilder**: 어떤 타입, 어떤 뷰든 사용 가능
2. **containerRelativeFrame**: 페이징의 핵심
3. **scrollTargetLayout + scrollTargetBehavior**: 스냅 효과
4. **scrollPosition**: 양방향 바인딩으로 상태 추적
5. **scrollTransition**: 부드러운 애니메이션

처음에는 "ScrollView 하나면 되지 않나?" 생각했는데, 막상 만들어보니 SwiftUI의 강력한 기능들을 많이 배웠다.
특히 `containerRelativeFrame`이랑 `scrollTargetLayout` 같은 최신 API들은 공식 문서를 여러 번 읽어야 이해가 되더라 😅

진도는 나가고 있는데... 사실 아주 느리다. 하지만 확실히 알때까지 대충 넘어가려는건 하지 않으려고한다. 많은 시행착오를 겪은 끝에 결국 나중에 자세히 봐야지라는 마인드 때문에 사실상 개발자가 아닌 복붙러가 되버린것같다. 다음엔 카테고리뷰다. 천천히 확실히 가보자.