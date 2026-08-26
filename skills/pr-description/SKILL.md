---
name: pr-description
description: "GitHub PR 제목/본문 작성 가이드. 요약, iOS 프로젝트용 테스트 계획 체크리스트, gh CLI를 이용한 PR 생성 절차를 다룬다. 'PR 작성', '풀리퀘스트 만들어줘', 'pull request 생성' 요청 시 사용."
---

# PR 설명 작성 스킬

브랜치의 전체 변경 이력을 리뷰어가 빠르게 파악할 수 있는 PR 제목/본문 작성 가이드.

## 작성 전 확인 절차

1. `git status` — 커밋되지 않은 변경이 없는지 확인
2. 현재 브랜치가 원격을 추적 중인지, push가 필요한지 확인
3. `git log {base}..HEAD`와 `git diff {base}...HEAD` — **마지막 커밋만이 아니라 브랜치의 모든 커밋**을 검토
4. `.github/PULL_REQUEST_TEMPLATE.md` 존재 여부 확인 — 있으면 그 구조를 우선 따른다

## PR 제목

70자 이내, 무엇이 바뀌었는지 한눈에 드러나게. 상세는 본문에 담는다.

- 좋음: `프로필 화면에 아바타 업로드 기능 추가`
- 나쁨: `여러 수정사항` — 무엇이 바뀌었는지 알 수 없음

## PR 본문 구조

```markdown
## Summary
- {핵심 변경 1}
- {핵심 변경 2}
- {핵심 변경 3, 필요 시}

## Test plan
- [ ] `xcodebuild build`/`swift build` 성공 확인
- [ ] 단위 테스트 통과 (`xcodebuild test`/`swift test`)
- [ ] 시뮬레이터에서 {구체적 화면/플로우} 수동 확인
- [ ] {영향받을 수 있는 다른 화면/기능} 회귀 확인
```

Test plan은 이 프로젝트에서 실제로 수행 가능한 항목만 체크리스트로 넣는다 — 수행하지 않을 항목을 형식적으로 나열하지 않는다. QA 리포트가 있다면 그 결과(통과/미검증)를 정직하게 반영한다.

## Summary 작성 원칙

- 커밋 메시지를 그대로 나열하지 않는다 — 리뷰어 관점에서 상위 수준으로 압축한다.
- "왜" 이 변경이 필요했는지 한 줄 포함하면 리뷰 속도가 빨라진다.
- 스크린샷/화면 녹화가 유용한 UI 변경이면, 사용자에게 첨부를 요청하거나 위치를 안내한다 (이 스킬 자체는 이미지를 생성하지 않는다).

## gh CLI로 PR 생성

```bash
git push -u origin {branch-name}   # 필요 시
gh pr create --title "{제목}" --body "$(cat <<'EOF'
## Summary
- ...

## Test plan
- [ ] ...
EOF
)"
```

- base 브랜치를 명시해야 하는 경우 `--base {branch}`를 추가한다.
- 원격 저장소가 없거나 `gh auth status`가 실패하면, PR 생성 대신 제목/본문 초안만 제공하고 사용자에게 원격/인증 설정을 안내한다.
- `main`/`master`로 강제 push하지 않는다.
