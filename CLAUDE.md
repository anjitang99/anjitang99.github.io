# Wiked Blog - iOS 개발 기술 블로그

## 프로젝트 개요

**Wiked blog**는 iOS 개발, 특히 SwiftUI에 대한 학습과 경험을 공유하는 기술 블로그입니다.

### 저자 정보
- **이름**: 안지훈 (anjitang99)
- **배경**: 백엔드 개발자에서 iOS 개발자로 전향
- **학습 중**: Swiftful Thinking의 SwiftUI 강의 (60+ 강의 수강 완료)
- **목표**: 세계 최강의 iOS 개발자

## 블로그 특성

### 주요 주제
- **iOS 개발**: Swift, SwiftUI, UIKit
- **아키텍처**: AppState 관리, 의존성 주입, MVVM 등
- **메모리 관리**: ARC, 순환 참조 해결
- **선언형 프로그래밍**: SwiftUI 패러다임
- **비교 학습**: Swift vs Java (백엔드 경험 활용)

### 글쓰기 스타일
- **톤**:
  - **반말 사용** (존댓말 X)
  - 친근하고 편안한 말투
  - 이모지 적절히 사용
  - 예: "이렇게 하면 된다", "알아보자", "깔끔하지?"
- **구조**: 명확한 섹션 구분 (들어가며, 핵심 개념, 예제, 마치며)
- **설명 방식**:
  - 개념 → 코드 예제 → 실제 사례
  - 비교 설명 (ARC vs GC, SwiftUI vs UIKit 등)
  - **디자인 패턴 비교 설명 지양** (예: Adapter Pattern, Strategy Pattern 등)
  - **백엔드 개발 비교 설명 지양** (예: Java, Spring 비교 등)
- **깊이**: 표면적 설명보다는 내부 동작 원리까지 파고듦
- **코드 블록**: 풍부한 Swift 코드 예제, 주석 포함

## 기술 스택

### 블로그 플랫폼
- **Static Site Generator**: Jekyll 4.x
- **테마**: jekyll-theme-chirpy
- **호스팅**: GitHub Pages (https://anjitang99.github.io)
- **언어**: 한국어 (ko)
- **타임존**: Asia/Seoul

### 개발 환경
- **로컬 서버**: `bundle exec jekyll serve --livereload`
- **빌드**: `bundle exec jekyll build`

## 파일 구조

```
anjitang99.github.io/
├── _config.yml          # Jekyll 설정
├── _posts/              # 블로그 포스트 (Markdown)
├── _tabs/               # About, Archives 등 탭 페이지
├── _data/               # 데이터 파일
├── assets/
│   └── img/             # 이미지 리소스
├── Gemfile              # Ruby 의존성
└── .claude/             # Claude Code 설정
```

## 포스트 작성 가이드

### Front Matter 템플릿

```yaml
---
title: 포스트 제목
date: YYYY-MM-DD HH:MM:SS +0900
categories:
  - iOS
  - SwiftUI
tags:
  - 태그1
  - 태그2
pin: false
---
```

### 카테고리 규칙
- **주 카테고리**: iOS, Swift
- **부 카테고리**: SwiftUI, UIKit, Architecture 등
- 계층 구조: 대분류 → 소분류

### 태그 규칙
- 구체적인 기술/개념 (SwiftUI, ARC, Environment 등)
- CamelCase 사용
- 강의명: SwiftfulThinking

### 이미지 사용
- 경로: `/assets/img/파일명.확장자`
- Chirpy 테마 이미지 문법:
  ```markdown
  ![설명](/assets/img/image.png){: width="400" }
  ```

### 특수 블록
```markdown
> 중요한 정보
{: .prompt-info }

> 팁
{: .prompt-tip }

> 경고
{: .prompt-warning }
```

### 각주 사용
```markdown
본문에서 용어[^1] 사용

---
[^1]: 각주 설명
```

## 포스트 시리즈

### SwiftUI 강의 정리 시리즈
1. SwiftUI 인트로
2. AppRoot - TabBar/Onboarding 아키텍처
3. AppState 아키텍처 - 전역 상태 관리
4. SwiftUI Layout Fundamentals

### 메모리 관리 시리즈
1. ARC vs GC - 메모리 관리 방식 비교
2. (예정) ARC 메모리 누수 해결

## 작성 시 주의사항

### 코드 예제
- Swift 코드는 명확한 주석 포함
- 비교 설명 시 Swift와 Java/Kotlin 코드 병행 제시
- 실제 동작하는 예제 코드 작성
- 나쁜 예/좋은 예 비교

### 설명 방식
- 백엔드 개발 경험과 연결하여 설명
- "왜?"에 대한 답변 포함
- 컴파일 타임 vs 런타임 구분 명확히
- 아키텍처 다이어그램 ASCII로 표현

### 참고 자료
- 공식 문서 링크 필수
- WWDC 세션 영상 링크
- 강의 출처 명시

## 로컬 개발

### 서버 실행
```bash
bundle exec jekyll serve --livereload
```
- 주소: http://127.0.0.1:4000/
- LiveReload: 파일 저장 시 자동 새로고침

### 새 포스트 작성
1. `_posts/` 폴더에 `YYYY-MM-DD-제목.md` 생성
2. Front Matter 작성
3. 본문 작성
4. 로컬 서버에서 미리보기
5. git commit & push

## Git 커밋 메시지 스타일

```
포스트 작성: SwiftUI 개념 정리
수정: ARC 설명 보완
이미지 추가: 스크린샷 첨부
```

## 유용한 명령어

```bash
# 로컬 서버 실행 (LiveReload 포함)
bundle exec jekyll serve --livereload

# 빌드만 수행
bundle exec jekyll build

# 의존성 설치
bundle install

# 캐시 정리
rm -rf .jekyll-cache _site
```

## 참고 링크

- **블로그 URL**: https://anjitang99.github.io
- **GitHub**: https://github.com/anjitang99
- **Chirpy 테마 문서**: https://github.com/cotes2020/jekyll-theme-chirpy
- **Swiftful Thinking**: https://www.youtube.com/@SwiftfulThinking
