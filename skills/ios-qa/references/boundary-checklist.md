# iOS 통합 정합성 체크리스트

`ios-qa` 스킬 본문에서 참조하는 상세 체크리스트. **SwiftUI + MVVM을 기준으로 작성되어 있다** — 프로젝트 컨벤션이 다르면 항목명을 그대로 찾지 말고 아래 대응표로 개념을 치환해서 적용한다.

| 이 문서의 표현 | TCA에서의 대응 | UIKit에서의 대응 |
|---|---|---|
| `@Published` 프로퍼티 | Reducer의 `State` 필드 | `IBOutlet`/프로퍼티 + delegate 콜백 |
| View ↔ ViewModel 바인딩 | View ↔ `Store` 바인딩 | ViewController ↔ 데이터소스/delegate |
| 상태 전이(enum) | Reducer의 `Action` 처리 완전성 | 커스텀 상태 프로퍼티 + 전이 코드 |
| 네비게이션(`NavigationLink` 등) | Reducer의 destination/child state | Storyboard segue, `UINavigationController.pushViewController` |

모듈화(Tuist/SPM) 프로젝트라면 위 항목에 더해 "모듈 간 의존성 규칙 위반"(예: Feature 모듈이 다른 Feature 모듈을 Interface 없이 직접 import)도 함께 확인한다.

검증 시 이 목록을 순서대로 확인한다.

## View ↔ ViewModel 바인딩

- [ ] ViewModel의 모든 `@Published` 프로퍼티가 실제로 View에서 사용되는가 (미사용이면 죽은 상태)
- [ ] View가 참조하는 `viewModel.x`/`$viewModel.x`가 ViewModel에 실제로 존재하는 프로퍼티명과 정확히 일치하는가 (리네임 후 누락 흔함)
- [ ] `@MainActor` 격리된 ViewModel의 프로퍼티를 non-MainActor 컨텍스트에서 동기적으로 읽으려는 코드가 없는가
- [ ] 로딩/에러/빈 상태(empty state)에 대응하는 분기가 View에 실제로 존재하는가, 아니면 성공 케이스만 렌더링하는가

## API 응답 ↔ 모델(Codable)

- [ ] 서버 응답 필드명(snake_case 등)과 `CodingKeys`의 매핑이 실제 응답과 일치하는가
- [ ] 서버가 nullable인 필드가 모델에서도 Optional로 선언되어 있는가 (아니면 디코딩 전체 실패)
- [ ] 페이지네이션 응답(`{ items, total, page }`)을 배열로 착각해 바로 사용하지 않는가
- [ ] 즉시 응답(예: 202 Accepted)과 최종/비동기 결과의 shape이 다른데 호출부가 이를 구분하지 않고 즉시 접근하지 않는가
- [ ] 날짜/시간 필드의 디코딩 전략(`ISO8601`, epoch 등)이 서버 포맷과 일치하는가

## 네비게이션

- [ ] `NavigationLink(value:)`가 push하는 타입에 대응하는 `.navigationDestination(for:)`이 실제로 등록되어 있는가
- [ ] `.sheet(item:)`/`.fullScreenCover(item:)`에 전달되는 Identifiable 값이 nil이 아닐 때만 의도대로 표시되는가
- [ ] enum 기반 route/화면 case 중 어디서도 push하지 않는 "죽은 case"가 없는가
- [ ] 딥링크(URL Scheme, Universal Link) 파싱 결과가 실제 존재하는 화면으로만 연결되는가

## 상태 전이 (상태머신을 쓰는 경우)

- [ ] enum으로 정의된 모든 상태 전이가 코드 어딘가에서 실제로 실행되는가 (죽은 전이 없음)
- [ ] 코드의 모든 상태 변경(`self.state = ...`)이 정의된 전이 목록에 있는가 (무단 전이 없음)
- [ ] 중간 상태(로딩 중, 처리 중)에서 최종 상태(완료/실패)로의 전환이 누락되어 영원히 로딩 상태로 남는 경로가 없는가
- [ ] 상태 기반 UI 분기(`if case .x = state`)의 모든 case가 실제 도달 가능한가

## 동시성/비동기

- [ ] `Task`가 View 생명주기보다 오래 살아남아 dealloc된 객체를 참조하지 않는가 (`[weak self]` 필요 여부 확인)
- [ ] 여러 비동기 작업이 동일한 `@Published` 프로퍼티를 경쟁적으로 갱신할 때 마지막 요청이 이전 요청 결과를 덮어쓰는 race condition이 없는가 (예: 빠른 연속 검색 요청)
- [ ] `async` 함수 내 에러가 `catch`되지 않고 무시되어 사용자에게 실패가 드러나지 않는 경로가 없는가

## 정적 위험 패턴 검색

```bash
grep -rn '!\s*$\|as!\|try!' Sources/ --include='*.swift'
```

검색된 각 위치에 대해 "이 값이 nil/타입 불일치일 수 있는 입력 경로가 실제로 없는가"를 개별 판단한다. 단순히 개수를 세어 보고하지 않는다.
