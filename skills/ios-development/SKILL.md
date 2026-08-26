---
name: ios-development
description: "Swift/SwiftUI/UIKit iOS 앱 개발의 아키텍처, 코드 컨벤션, 동시성, 테스트 가능한 설계 가이드. 화면/기능 구현, 리팩토링, Swift 코드 작성, 프로젝트 구조 설계 시 반드시 이 스킬을 사용할 것. 단순 문법 질문에는 불필요하지만, 실제 기능 구현이나 아키텍처 결정이 필요하면 사용."
---

# iOS 개발 스킬

Swift/SwiftUI/UIKit 프로젝트를 테스트 가능하고 일관된 구조로 구현하기 위한 가이드. 이 스킬은 특정 아키텍처(MVVM/TCA/Clean Architecture)나 UI 프레임워크(SwiftUI/UIKit)에 종속되지 않는다 — 팀마다 조합이 다르고(MVVM+Clean처럼 혼용도 흔함) 그 경우의 수를 스킬이 미리 다 예측할 수 없기 때문이다. 대신 아래 순서로 실제 사용할 스택을 판별한다.

## 0. 프로젝트 컨벤션 확인 (가장 먼저)

1. 프로젝트 루트의 `CLAUDE.md`에 `## 프로젝트 컨벤션` 섹션이 있는지 확인한다. 값이 직접 적혀 있으면 그대로 따른다. 값 대신 기존 아키텍처 문서의 경로/링크만 적혀 있으면(예: "아키텍처: `docs/ARCHITECTURE.md` 참조") 그 문서를 반드시 `Read`로 열어 실제 내용을 확인한다 — 링크만 보고 넘어가지 않는다.
2. `CLAUDE.md`에 컨벤션 섹션 자체가 없으면, 저장소에 이미 문서화된 아키텍처/컨벤션 문서가 있는지 먼저 찾는다. 파일명은 프로젝트마다 다르므로 `find . -iname "ARCHITECTURE*" -o -iname "CONVENTIONS*" -o -path "*/docs/*.md"` 등으로 탐색하고, `README.md`에 아키텍처 절이 있는지도 확인한다. 찾았으면 그 내용을 최우선으로 따른다.
3. 위 두 가지 모두 없으면 기존 코드베이스를 탐색해 추론한다: `import SwiftUI` vs `import UIKit` 비중, `Reducer`/`ComposableArchitecture` 사용 여부(TCA), `Domain`/`Data`/`Presentation` 폴더 구조(Clean Architecture), `Tuist/`·`Project.swift`(Tuist 모듈화) 또는 하위 다중 `Package.swift`(SPM 모듈화) 존재 여부.
4. 셋 다 없는 완전 신규 프로젝트라면 아래 절의 기본값(SwiftUI + MVVM, 단일 타겟)을 적용하되, **이 가정을 결과 보고에 명시**한다. 동시에 사용자에게 `templates/PROJECT_CONVENTIONS.md`를 프로젝트 `CLAUDE.md`에 채워넣도록 안내한다 — 이미 별도 아키텍처 문서가 있다면 템플릿 전체를 다시 쓰지 말고 `CLAUDE.md`에 그 문서 경로만 링크해도 충분하다.
5. TCA, Tuist처럼 문법이 자주 바뀌는 라이브러리를 다룰 때 최신 API가 확실하지 않으면, 정적 지식에 의존하지 말고 `context7` MCP 도구로 직접 확인한다. 단발성 확인(함수 시그니처 하나 등)이면 직접 조회하고, 새 SDK 도입 검토나 여러 라이브러리 비교처럼 조사 자체가 무거우면 `ios-researcher` 에이전트에게 위임한다.

## 프로젝트 구조 파악

새 코드를 작성하기 전, 기존 프로젝트 구조를 먼저 확인한다. `*.xcodeproj`/`*.xcworkspace`/`Package.swift`/`Tuist/`의 존재, `Sources/`, `Features/`, `Models/` 등 폴더 관례를 파악하고 그 패턴을 따른다. 프로젝트가 아직 없다면(위 0단계에서 신규로 판별된 경우) 아래 기본 구조를 제안한다:

```
Sources/
  App/                 - App entry point, DI 조립
  Features/{Feature}/  - View + ViewModel + 관련 모델을 기능 단위로 묶음
  Core/
    Networking/        - API 클라이언트, DTO
    Persistence/       - 로컬 저장소
    Extensions/
  Resources/           - Assets, 로컬라이제이션
Tests/
  {Feature}Tests/
```

기능 단위(vertical slice)로 폴더를 나누면 관련 파일을 함께 찾기 쉽고, 기능 삭제 시 폴더째 제거할 수 있다.

## 아키텍처: 프로젝트 컨벤션이 없을 때의 기본값 — MVVM

위 0단계에서 프로젝트가 이미 TCA/Clean Architecture 등 특정 패턴을 쓰기로 정해져 있다면 그 규칙을 따른다(예: TCA는 Reducer/State/Action/Store, `@Dependency` 기반 주입을 쓴다 — 아래 MVVM 예시를 억지로 적용하지 않는다). 아무 컨벤션도 없는 신규 코드에서는 ViewModel이 상태와 로직을 갖고 View는 렌더링만 하는 MVVM을 기본값으로 쓴다 — 이렇게 분리해야 `ios-unit-tester`가 View 없이 ViewModel만 독립적으로 테스트할 수 있다.

```swift
protocol UserRepository {
    func fetchUser(id: String) async throws -> User
}

@MainActor
final class ProfileViewModel: ObservableObject {
    @Published private(set) var user: User?
    @Published private(set) var errorMessage: String?

    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func load(id: String) async {
        do {
            user = try await repository.fetchUser(id: id)
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

핵심: `UserRepository`를 protocol로 추상화하고 생성자 주입했기 때문에, 테스트에서는 네트워크 없이 `MockUserRepository`를 주입할 수 있다. 싱글톤(`.shared`)을 코드 내부에서 직접 참조하면 이 대체가 불가능해진다 — 왜냐하면 테스트가 실제 싱글톤 인스턴스에 종속되어 결정적이지 않은 테스트가 되기 때문이다.

## UI 프레임워크: 컨벤션이 없을 때는 SwiftUI 기본, UIKit은 필요할 때만

프로젝트가 UIKit 전용/하이브리드로 정해져 있다면 그 규칙을 따른다. 정해진 게 없는 신규 화면은 SwiftUI로 작성한다. 다음 경우에만 UIKit을 사용하거나 `UIViewRepresentable`/`UIViewControllerRepresentable`로 래핑한다:
- 레거시 UIKit 화면과의 점진적 통합
- SwiftUI가 아직 지원하지 않는 세밀한 제어(예: 특정 제스처 조합, 커스텀 텍스트 레이아웃)
- 서드파티 SDK가 UIKit 기반 뷰만 제공하는 경우

## Swift Concurrency 원칙

- UI 상태를 다루는 타입(ViewModel)은 `@MainActor`로 격리해 메인 스레드 접근을 컴파일 타임에 보장한다.
- 비동기 작업은 `async/await`를 우선 사용한다. completion handler 기반 API는 새로 작성하지 않는다.
- `Task { }`를 View 안에서 직접 생성하기보다 `.task { }` modifier를 사용해 View 생명주기와 동기화한다.
- 여러 비동기 작업을 병렬로 실행할 때는 `async let` 또는 `TaskGroup`을 사용한다.

## 에러 핸들링

- 복구 가능한 실패는 `throws` + 커스텀 `Error` enum으로 표현한다. `Bool` 반환이나 옵셔널로 실패를 숨기지 않는다.
- 강제 언래핑(`!`)은 로직상 절대 nil이 될 수 없음이 타입 시스템으로 보장될 때만 사용한다. 그렇지 않으면 `guard let`/`if let`으로 처리한다 — force-unwrap 크래시는 QA가 가장 자주 발견하는 결함 유형이다.

## 빌드 검증

구현 후 반드시 빌드를 확인한다:

```bash
xcodebuild build -scheme {SchemeName} -destination 'platform=iOS Simulator,name=iPhone 16' | xcpretty || xcodebuild build -scheme {SchemeName} -destination 'platform=iOS Simulator,name=iPhone 16'
```

Swift Package라면:
```bash
swift build
```

`xcpretty`가 없으면 원본 `xcodebuild` 출력을 그대로 확인해도 무방하다 — 가독성 도구일 뿐 필수는 아니다.

## 코드 스타일

- Swift API Design Guidelines를 따른다: 타입은 `UpperCamelCase`, 변수/함수는 `lowerCamelCase`, 프로토콜명은 능력을 나타내는 형용사(`Codable`, `Equatable`) 또는 명사(`UserRepository`)로 짓는다.
- 함수는 호출부에서 자연스러운 문장처럼 읽히도록 인자 레이블을 설계한다 (`fetchUser(id:)`가 `fetchUser(userId:)`보다 호출부에서 `fetchUser(id: user.id)`로 자연스럽다면 전자를 선택).
- 하나의 타입/파일이 여러 책임을 가지면 분리를 고려한다. 다만 과도한 추상화(3줄짜리 프로토콜 하나만 있는 파일 남발)는 피한다 — 실제로 여러 구현체가 필요하거나 테스트 대체가 필요할 때만 protocol을 도입한다.
