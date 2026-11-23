+++
date = '2026-12-03T09:00:00+09:00'
draft = false
title = 'Git Submodule #14: 원리와 Bundle·Replace'
description = 'Pro Git 7장 4부(7.11~7.14)를 바탕으로 Submodule Gitlink 원리·함정과 Subtree·Monorepo 비교, git bundle·replace·인증 헬퍼를 정리한다.'
tags = ['Git', 'ProGit', 'Submodule', 'Monorepo', 'GitTools', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#13)](/posts/pro-git-internals-ch7-part3/)을 통해 `zdiff3`, `revert -m 1` 재병합 함정, 그리고 `git bisect run` 자동 디버깅을 살펴보았다. 이제 대단원인 **7장 4부(`#14`, Pro Git 7.11~7.14)** 에서 **Submodule(서브모듈)의 내부 원리와 실무 함정**, 현대적 대안 비교, 그리고 **오프라인 `git bundle` · 해시 보존 접합 `git replace` · 안전한 인증 저장소(Credential Helper)** 까지 모두 마치고 Pro Git 7장을 완주한다.

---

## 1. Git Submodule 내부 원리: 왜 Gitlink는 브랜치가 아니라 커밋 해시인가? (7.11)

프로젝트를 진행하다 보면 다른 Git 저장소의 라이브러리를 내 저장소 하위 폴더에 통째로 넣고 싶을 때가 있다. 단순히 폴더를 복사해 커밋하면 외부 라이브러리 업데이트를 따라가기 어렵고 수정 사항도 섞인다. **Submodule**은 하나의 Git 저장소 안에 다른 독립 Git 저장소를 **하위 디렉토리로 연결하되, 두 저장소의 커밋 이력은 분리해 관리**하는 기능이다.

{{< figure src="submodule-and-special-tools.svg" alt="Git Submodule Gitlink 160000 포인터 구조와 Subtree/Monorepo 비교, 그리고 git bundle, replace, credential helper 다이어그램" caption="Superproject의 Gitlink(160000) 고정 원리 및 Submodule vs Subtree vs Monorepo 비교(상단)와 특수 도구 3종(하단) (Pro Git 7.11~7.14 재구성)" width="820" height="540" >}}

### 부모 저장소(Superproject)는 대체 무엇을 저장하는가?

많은 개발자가 Submodule을 추가(`git submodule add`)하면 외부 코드 파일 전체가 부모 저장소에 복사된다고 착각한다. 하지만 `git ls-tree HEAD`로 부모 저장소의 Tree 객체를 들여다보면 정체가 명확히 드러난다.

```text
100644 blob e69de29bb2d1d6434b8b29ae775ad8c2e48c5391    .gitmodules
160000 commit a1b2c3d4e5f60718293a4b5c6d7e8f9012345678  libs/ui
```

- **`.gitmodules`**: `libs/ui` 경로가 어느 원격 URL(`https://...`)과 매핑되는지 기록한 설정 파일이다.
- **Gitlink (`mode 160000 commit`)**: 부모 저장소는 하위 모듈의 실제 소스 파일 대신 **"그 시점에 체크아웃해야 할 40바이트 커밋 해시 포인터 1개"** 만 Tree 엔트리로 저장한다.

그렇기 때문에 부모 저장소가 외부 라이브러리의 `main` 브랜치를 통째로 따라가다가 어느 날 갑자기 빌드가 깨지는 참사 없이, **언제 클론하든 정확히 검증된 특정 커밋(`a1b2c3d`) 상태로 재현**할 수 있다.

### Submodule 실무 필수 명령어 4종 세트

```bash
# 1. Submodule이 포함된 저장소를 처음 클론할 때 (빈 폴더 방지!)
$ git clone --recurse-submodules https://github.com/org/superproject.git

# 이미 그냥 git clone 했다면 아래 1줄로 초기화 + 내려받기
$ git submodule update --init --recursive

# 2. 협업자가 Submodule 버전을 올려서 git pull 받았을 때 하위 폴더 동기화
$ git pull --recurse-submodules

# 3. Submodule 원격 저장소의 최신 커밋으로 한꺼번에 갱신하기
$ git submodule update --remote --merge
```

---

## 2. Submodule의 3대 실무 함정과 현대적 대안 비교 (Subtree vs Monorepo)

Submodule은 원리가 명확한 대신 손이 많이 가는 기능으로도 유명하다. 실무에서 겪는 가장 흔한 함정 세 가지는 다음과 같다.

1. **`Detached HEAD` 함정**: `git submodule update`를 실행하면 Submodule 디렉토리는 특정 브랜치(`main`)가 아니라 특정 커밋 해시에 직접 체크아웃(**Detached HEAD**)된다. 이 상태를 모른 채 Submodule 폴더 안에서 코드를 수정·커밋하고 다음에 `git submodule update`를 실행하면, **방금 만든 커밋이 브랜치 연결 없이 공중에 떠 버려(`orphaned`) 사라질 위험**이 있다. Submodule 내부에서 작업할 때는 반드시 `git switch main` 또는 `--rebase`/`--merge` 옵션을 챙겨야 한다.
2. **하위 커밋 푸시 누락 참사**: 내 로컬에서 Submodule 수정 커밋(`e4f5a6b`)을 만들고 부모 저장소에 `Gitlink`를 올려서 부모만 Push하면, 동료가 `git pull` 받았을 때 원격 Submodule 저장소에 `e4f5a6b` 커밋이 없어 빌드가 완전히 막힌다. 이를 막기 위해 **`git push --recurse-submodules=check`**(미푸시 서브모듈이 있으면 거부) 또는 **`on-demand`**(자동 동반 푸시)를 쓰는 습관이 좋다.
3. **브랜치 전환 시 흔적 충돌**: 브랜치 A에는 Submodule 폴더가 있고 브랜치 B에는 일반 폴더가 있을 때 전환이 꼬일 수 있다.

### 현대 프로젝트에서는 무엇을 쓰나? Submodule vs Subtree vs Monorepo

| 접근 방식 | 동작 구조 | 장점 | 단점 및 현대적 선택 기준 |
| :--- | :--- | :--- | :--- |
| **Git Submodule** | 별도 저장소 유지 + `160000` 커밋 해시 고정 | 저장소 완전 독립, 이력 비대화 방지 | Detached HEAD 등 팀원 Git 숙련도 요구 (C/C++ 서드파티, 펌웨어 SDK 등에 주로 잔존) |
| **Git Subtree** (`7.15`) | 외부 저장소 소스를 내 저장소 하위 폴더로 병합(`--squash`) | 팀원은 `clone` 후 일반 폴더처럼 바로 사용 | `git subtree push/split`으로 변경을 원류에 되돌릴 때 이력 관리가 번거로움 |
| **Monorepo / 패키지 매니저** | 단일 저장소 내 Workspace 구성 또는 `npm`·`Maven`·`Go module` 배포 | 버전·테스트·리팩토링·CI 통합이 가장 매끄러움 | **현대 웹·백엔드 실무 최우선 권장안** (같은 팀 코드는 Monorepo, 외부 의존성은 패키지 레지스트리 사용) |

---

## 3. 알아두면 결정적인 Git 특수 도구 3종: `bundle` · `replace` · `credential`

Pro Git 7장 후반부(7.12~7.14)는 매일 쓰지는 않지만 **특정 제약 상황에서 유일한 탈출구가 되어 주는 세 가지 도구**를 소개한다.

### ① 네트워크가 끊긴 폐쇄망을 뚫는 오프라인 전송: `git bundle` (7.12)

사내 망분리나 Air-gapped 환경, 혹은 대용량 이력을 USB나 메신저 파일 하나로 통째로 전달해야 할 때 디렉토리를 `zip`으로 압축하면 `.git` 내부 파일 권한이나 진행 중인 상태까지 섞여 지저분해진다. **`git bundle`** 은 `git push` 시 네트워크로 보낼 모든 객체와 참조를 **딱 1개의 바이너리 `.bundle` 파일**로 묶어 준다.

```bash
# 1. 전체 main 브랜치 이력을 단일 파일로 패키징
$ git bundle create repo.bundle HEAD main

# 2. 받는 쪽 폐쇄망 PC에서는 일반 원격 URL처럼 그대로 clone 가능!
$ git clone repo.bundle my-project -b main

# (참고) 특정 기준 태그(v1.0) 이후 새로 쌓인 커밋만 증분 번들로 만들 수도 있다
$ git bundle create update.bundle main --not v1.0
$ git pull /path/to/update.bundle main
```

### ② 기존 SHA-1 해시를 깨지 않는 투명한 이력 접합: `git replace` (7.13)

10년 치 수십만 커밋이 쌓인 거대 저장소를 정리하기 위해 최근 1,000개 커밋만 잘라내어 배포하고 싶은데, 나중에 필요할 때만 과거 10년 치 역사를 연결해서 보고 싶다면 어떻게 해야 할까? 부모 포인터를 수정해 `rebase`나 `filter-repo`를 돌리면 **모든 후속 커밋의 SHA-1 해시가 전부 바뀌어 기존 태그와 CI가 깨진다.**

**`git replace`** 는 커밋 객체 자체를 건드리지 않고 `.git/refs/replace/`에 **"A 커밋을 읽을 때 대신 B 커밋(또는 과거 이력을 가진 부모)을 보여 달라"** 고 투명하게 매핑해 둔다. 덕분에 기존 SHA-1 해시와 서명 태그를 단 하나도 깨뜨리지 않은 채, 평소에는 가벼운 저장소로 일하다가 필요할 때만 과거 히스토리 저장소를 `fetch`해 이어 붙일 수 있다.

### ③ 안전한 HTTPS 인증 저장소: OS Keychain과 Git Credential Manager (7.14)

[4장(#7)]({{< ref "/posts/pro-git-internals-ch4" >}})에서 살펴본 것처럼 오늘날 원격 Git 통신의 표준은 Smart HTTPS(+ Personal Access Token)다. 매번 토큰을 입력하지 않기 위해 쓰는 것이 **Credential Helper**다.

- **권장 설정**: macOS의 **`osxkeychain`**, Windows/Linux의 크로스플랫폼 표준인 **`manager`(Git Credential Manager)** 를 사용하면 운영체제의 암호화된 키체인(Keychain / Credential Vault)에 토큰과 OAuth 인증을 안전하게 보관한다.
- **절대 피해야 할 설정**: `git config --global credential.helper store`를 켜면 홈 디렉토리 아래 **`~/.git-credentials` 텍스트 파일에 비밀번호와 PAT 토큰을 암호화 없는 평문으로 저장**하므로 공용 장비나 보안 환경에서는 절대 사용하지 말아야 한다(`cache` 메모리 임시 보관 또는 OS Keychain 사용 권장).

```bash
# 현재 내 Git에 등록된 Credential Helper 확인
$ git config --show-origin --get-all credential.helper
```

---

## 마치며

이로써 총 4부작(`#11`~`#14`)에 걸쳐 Pro Git에서 가장 방대하고 실전적인 **7장(Git Tools)** 완주를 마쳤다.

- **1부 (`#11`)**: 조상 탐색(`^`/`~`)과 `reflog` 복구, `git add -p`, `stash`/`clean`, `grep` & `log -S/-G/-L`, SSH 커밋 서명
- **2부 (`#12`)**: 3가지 트리 관점의 `git reset` 3단계 해부, `rebase -i`, 현대 표준 `git filter-repo`
- **3부 (`#13`)**: `zdiff3` 충돌 맥락, `revert -m 1` 재병합 함정, `git blame -C`, `git bisect run` 자동 디버깅
- **4부 (`#14`)**: Submodule Gitlink 원리와 함정, Monorepo 대안, 오프라인 `git bundle`, 해시 보존 `git replace`, 안전한 Credential 저장소

Git의 내부 객체 그래프와 3가지 트리 모델을 바탕으로 이 도구들의 원리까지 이해했다면, 이제 어떤 장애 복구·대규모 협업·마이그레이션 상황 앞에서도 당황하지 않고 가장 안전한 해법을 선택할 수 있다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 7.11~7.14 Git Tools](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [Git Documentation - git-submodule](https://git-scm.com/docs/git-submodule)
- [Git Documentation - git-bundle & git-replace](https://git-scm.com/docs/git-bundle)
- [Git Credential Manager (GCM) Official Repository](https://github.com/git-ecosystem/git-credential-manager)
