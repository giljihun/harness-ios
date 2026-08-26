---
name: ios-unit-testing
description: "iOS Swift 코드의 단위 테스트 작성 가이드. XCTest와 Swift Testing 프레임워크, mock/stub 작성, 비동기 코드 테스트, ViewModel 테스트 패턴을 다룬다. '단위 테스트', '유닛테스트', 'XCTest', '테스트 커버리지', '테스트 작성해줘' 요청 시 반드시 사용."
---

# iOS 단위 테스트 스킬

Swift 코드에 대해 결정적이고 의미 있는 단위 테스트를 작성하기 위한 가이드.

## 프로젝트 컨벤션 확인 (가장 먼저)

프로젝트 `CLAUDE.md`의 `## 프로젝트 컨벤션`에 테스트 프레임워크/아키텍처가 명시되어 있으면 최우선으로 따른다. 값 대신 기존 문서(`docs/ARCHITECTURE.md` 등)의 경로/링크만 있으면 그 문서를 `Read`로 열어 확인한다. `CLAUDE.md`에 컨벤션 섹션이 아예 없으면, 코드 탐색으로 넘어가기 전에 저장소에 이미 아키텍처/테스트 전략 문서가 있는지부터 찾는다(`find . -iname "ARCHITECTURE*" -o -iname "TESTING*" -o -path "*/docs/*.md"`). 특히 아키텍처 패턴에 따라 테스트 방법론 자체가 달라진다:

- **TCA 프로젝트** — ViewModel+mock 방식이 아니라 `TestStore`로 Action을 보내고 State 변화를 assert하는 방식을 쓴다. `TestStore`/`@Dependency` 최신 문법이 확실하지 않으면 `context7` MCP로 공식 문서를 조회한다.
- **MVVM 프로젝트** — 아래 절의 XCTest/Swift Testing + mock 주입 방식을 따른다.
- **Clean Architecture** — UseCase/Repository 단위로 테스트하며, 레이어 경계(Domain이 Data를 참조하지 않는지)를 mock 설계에도 반영한다.

컨벤션이 없으면 기존 테스트 파일의 패턴을 그대로 따르고, 그마저 없는 신규 프로젝트면 아래 MVVM 기준 가이드를 기본값으로 쓴다.

## 프레임워크 선택: XCTest vs Swift Testing

기존 프로젝트에 이미 사용 중인 프레임워크를 그대로 따른다(테스트 파일 상단의 `import XCTest` 또는 `import Testing` 확인). 새 프로젝트이고 Swift 6 / Xcode 16 이상이면 Swift Testing을 기본으로 제안한다 — 문법이 더 간결하고 병렬 실행을 기본 지원하기 때문이다. UI 테스트(`XCUITest`)는 이 스킬의 범위가 아니다(단위 테스트 전용).

**XCTest:**
```swift
import XCTest
@testable import MyApp

final class ProfileViewModelTests: XCTestCase {
    func test_load_success_setsUser() async {
        let sut = ProfileViewModel(repository: MockUserRepository(result: .success(.fixture())))
        await sut.load(id: "1")
        XCTAssertEqual(sut.user?.id, "1")
    }
}
```

**Swift Testing:**
```swift
import Testing
@testable import MyApp

@Suite("ProfileViewModel")
struct ProfileViewModelTests {
    @Test("로드 성공 시 user가 설정된다")
    func loadSuccessSetsUser() async {
        let sut = ProfileViewModel(repository: MockUserRepository(result: .success(.fixture())))
        await sut.load(id: "1")
        #expect(sut.user?.id == "1")
    }
}
```

## Given-When-Then 구조

테스트 본문은 준비(Given) → 실행(When) → 검증(Then) 순서로 구성한다. 테스트명은 `test_{대상}_{조건}_{기대결과}` 패턴을 따르면 실패 시 무엇이 깨졌는지 이름만으로 파악할 수 있다.

## Mock/Stub 작성 원칙

테스트 대상이 의존하는 protocol마다 제어 가능한 mock을 만든다. mock은 실제 동작을 흉내내지 않고, **테스트가 지정한 값을 그대로 반환**해야 한다.

```swift
final class MockUserRepository: UserRepository {
    var result: Result<User, Error>
    private(set) var fetchCallCount = 0

    init(result: Result<User, Error>) { self.result = result }

    func fetchUser(id: String) async throws -> User {
        fetchCallCount += 1
        return try result.get()
    }
}
```

`fetchCallCount` 같은 호출 횟수 추적은 "이 함수가 정확히 몇 번 호출됐는가"가 실제 요구사항일 때만 추가한다 — 모든 mock에 습관적으로 넣지 않는다.

## 비동기 코드 테스트

- `async`로 선언된 대상은 테스트 함수도 `async`로 선언하고 `await`로 직접 호출한다. `XCTestExpectation` + `wait(for:)`는 콜백 기반 레거시 API를 테스트할 때만 사용한다.
- `@MainActor`로 격리된 타입을 테스트할 때는 테스트 클래스/구조체 또는 함수에도 `@MainActor`를 붙여야 컴파일 에러 없이 접근할 수 있다.
- 타이밍에 의존하는 테스트(`sleep`, 임의의 딜레이)는 작성하지 않는다 — 대신 mock이 완료 시점을 제어하게 하거나 `Task`의 완료를 직접 `await`한다. 시간 기반 테스트는 CI에서 불안정(flaky)해지는 주요 원인이다.

## ViewModel 테스트 패턴

ViewModel 테스트는 "입력(함수 호출/프로퍼티 변경) → `@Published` 상태 변화"를 검증한다. View를 렌더링하거나 SwiftUI 프리뷰를 구동할 필요가 없다 — 이것이 MVVM으로 분리하는 핵심 이유다.

검증 우선순위:
1. 비즈니스 로직 (계산, 검증 규칙, 상태 전이 조건)
2. 상태 전이 (로딩 → 성공/실패, 빈 상태 처리)
3. 에러 경로 (의존성이 실패를 반환했을 때 사용자에게 노출되는 상태)
4. 유틸리티 함수 (포맷터, 파서 등 순수 함수)

getter만 호출하는 테스트, 프레임워크 자체 동작(예: `@Published`가 값을 저장하는지)을 검증하는 테스트는 작성하지 않는다 — 의미 있는 커버리지가 아니다.

## 테스트 실행

```bash
xcodebuild test -scheme {SchemeName} -destination 'platform=iOS Simulator,name=iPhone 16'
```

Swift Package: `swift test`

테스트 작성 후 반드시 실행해 통과를 확인한다. 실행 환경이 없다면 정적 검토로 대체하되 그 사실을 명시한다.
