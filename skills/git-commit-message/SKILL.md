---
name: git-commit-message
description: "git 커밋 메시지 작성 형식과 예시 가이드. diff 분석 절차, Conventional Commits 여부 판단, 왜(why) 중심 메시지 작성법을 다룬다. '커밋 메시지', '커밋해줘', 'git commit' 요청 시 사용."
---

# Git 커밋 메시지 작성 스킬

diff를 분석해 "무엇을"보다 "왜" 바꿨는지가 드러나는 커밋 메시지를 작성하기 위한 가이드.

## 작성 절차

1. `git status`로 변경/미추적 파일을 확인한다.
2. `git diff`(unstaged)와 `git diff --staged`로 실제 변경 내용을 읽는다 — 파일명만 보고 추측하지 않는다.
3. `git log --oneline -10`으로 이 저장소의 기존 메시지 스타일(Conventional Commits 접두어 사용 여부, 언어, 평서문/명령문)을 파악하고 따른다.
4. 변경의 성격을 판단한다: 새 기능(feat) / 버그 수정(fix) / 리팩토링(refactor) / 테스트(test) / 문서(docs) / 스타일(style) / 빌드/설정(chore).

## 메시지 형식

기존 저장소에 Conventional Commits 스타일(`feat:`, `fix:` 등)이 이미 쓰이고 있으면 그대로 따른다. 스타일이 없는 새 저장소라면 Conventional Commits를 기본으로 제안한다:

```
{type}({scope 선택}): {한 줄 요약, 명령문, 마침표 없이}

{선택: 본문 — 왜 이 변경이 필요했는지, 무엇을 고려했는지}
```

## 예시

**입력:** 사용자 프로필 화면에 프로필 사진 업로드 기능 추가, `UserRepository`에 `uploadAvatar` 메서드 추가, 관련 ViewModel/View 수정
**출력:**
```
feat(profile): 프로필 사진 업로드 기능 추가
```

**입력:** `ProfileViewModel.load`에서 네트워크 실패 시 앱이 크래시하던 강제 언래핑 버그 수정
**출력:**
```
fix(profile): 프로필 로드 실패 시 강제 언래핑으로 인한 크래시 수정
```

**입력:** `NetworkClient`를 protocol로 추상화해 테스트 가능하게 리팩토링 (동작 변경 없음)
**출력:**
```
refactor(networking): NetworkClient를 protocol로 추상화

단위 테스트에서 mock으로 대체할 수 있도록 구체 타입 의존을
제거했다. 외부 동작은 변경되지 않았다.
```

## 원칙

- 제목은 50~72자 내외로, 마침표 없이 명령문으로 쓴다.
- "코드 수정", "업데이트" 같은 내용 없는 요약은 금지 — diff를 봐야 알 수 있는 정보(무엇이 왜 바뀌었는지)를 담는다.
- 여러 관심사가 섞인 변경(기능 추가 + 무관한 포맷팅/파일 이동)이 하나의 diff에 있으면, 커밋을 분리할지 사용자에게 먼저 확인한다.
- `Package.resolved`, `*.xcodeproj/project.pbxproj` 같은 생성/설정 파일 변경은 본문에서 "의존성 버전 갱신" 등으로 요약하고 라인 단위로 설명하지 않는다.
- `.env`, 인증서, provisioning profile, API 키가 포함된 파일은 스테이징 전 반드시 내용을 확인하고 사용자에게 알린다.
