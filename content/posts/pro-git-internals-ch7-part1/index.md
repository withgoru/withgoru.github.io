+++
date = '2026-11-30T09:00:00+09:00'
draft = false
title = 'Git 실전 도구 #11: Reflog·Stash·코드 검색'
description = 'Pro Git 7장 1부(7.1~7.5)를 바탕으로 Reflog와 ^/~ 탐색, add -p, Stash·Clean, grep과 -S/-G/-L 검색, SSH 서명을 정리한다.'
tags = ['Git', 'ProGit', 'Reflog', 'Stash', 'GitTools', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

지금까지 **Pro Git 1~6장**을 통해 Git의 객체 모델, 브랜치와 리베이스, 서버 프로토콜, 분산 워크플로우, 그리고 현대적인 원격 호스팅 플랫폼까지 큰 뼈대를 모두 완성했다. 이제 책에서 가장 방대하고 실무 밀도가 높은 **Pro Git 7장(Git Tools)** 으로 들어갈 차례다. 7장은 총 4부작(`#11`~`#14`)으로 나누어 하나씩 깊게 파헤친다.

이번 1부(**7.1~7.5**)에서는 개발자가 매일 마주치는 다섯 가지 핵심 도구인 **리비전 선택(`^`/`~`, Reflog)**, **대화형 스테이징(`git add -p`)**, **임시 저장과 청소(`stash`, `clean`)**, **정밀 코드·이력 검색(`grep`, `log -S/-G/-L`)**, 그리고 **SSH 기반 커밋 서명**을 정리한다.

---

## 1. 리비전 선택(Revision Selection): 조상 탐색과 Reflog 안전망 (7.1)

Git의 거의 모든 명령은 특정 커밋을 가리키는 리비전 표현식을 인자로 받는다. 40자리 SHA-1 해시를 일일이 복사하지 않고도 원하는 지점을 정확히 짚어 내는 두 가지 핵심 문법이 **조상 탐색(`^`, `~`)** 과 **Reflog(`HEAD@{n}`)** 다.

{{< figure src="reflog-ancestry-and-stash.svg" alt="Git 조상 탐색 캐럿(^)과 틸드(~), Reflog(HEAD@{n}) 안전망, Stash 스택 및 -S/-G/-L 검색 비교 다이어그램" caption="병합 커밋 조상 탐색(^/~) 및 Reflog 복구 원리(상단)와 git stash 수명 주기 및 정밀 검색 4종 비교(하단) (Pro Git 7.1~7.5 재구성)" width="820" height="540" >}}

### 캐럿(`^`) vs 틸드(`~`) : 병합 커밋에서 갈라지는 운명

직선 히스토리에서는 `HEAD^`와 `HEAD~`가 똑같이 직전 부모 커밋을 가리킨다. 하지만 두 브랜치를 합친 **Merge 커밋 `M`** 앞에서는 동작이 완전히 달라진다.

- **틸드(`~N`)**: 항상 **첫 번째 부모(First-parent, 병합 당시 내가 머물던 본류 브랜치)** 계보를 따라 `N`세대 위로 거슬러 올라간다. 예를 들어 `M~2`는 `M`의 직전 부모 `C3`를 거쳐 그 부모인 `C2`를 가리킨다(`M^^`와 동일).
- **캐럿(`^N`)**: 해당 커밋이 가진 **N번째 부모**를 선택한다. Merge 커밋 `M`에서 `M^`(또는 `M^1`)는 본류 부모 `C3`지만, **`M^2`는 머지되어 들어온 브랜치의 마지막 커밋 `B2`** 를 가리킨다.

### 로컬 실수 복구의 최후 보루: `git reflog`

실수로 `git reset --hard HEAD~2`를 실행해 방금 만든 커밋들이 브랜치에서 사라졌을 때, 당황하지 말고 **`git reflog`** 를 치면 된다. Git은 작업 중인 내 로컬 저장소에서 **`HEAD`나 브랜치 포인터가 움직인 모든 순간을 `.git/logs/` 아래 기록해 둔다.**

```bash
$ git reflog
1a2b3c4 (HEAD -> main) HEAD@{0}: reset: moving to HEAD~2
9f8e7d6 HEAD@{1}: commit: feat(auth): add JWT refresh token
4c5d6e7 HEAD@{2}: checkout: moving from develop to main

# 실수 직전 시점(HEAD@{1})으로 즉시 완벽 복구
$ git reset --hard HEAD@{1}
```

단, **`reflog`는 철저히 내 로컬 PC 안에만 존재하며 `git push`나 `clone` 시 서버로 전송되지 않는다.** 또한 기본 만료 기간(보통 90일, 도달 불가능한 커밋은 30일)이 지나 `git gc`가 돌기 전까지만 유효하다.

*(범위 표기법 `A..B` vs `A...B`의 차이는 [지난 글(#9)]({{< ref "/posts/pro-git-internals-ch5-part2" >}})에서 상세히 다루었으므로 7.1 복습 시 함께 참고하면 좋다.)*

---

## 2. 대화형 스테이징: `git add -p`로 원자적 커밋 조각하기 (7.2)

한 파일 안에서 두 가지 다른 버그를 고쳤거나, 디버깅용 `console.log`가 실제 로직 수정과 섞여 있을 때 `git add .`를 치면 지저분한 단일 커밋이 만들어진다. **`git add -p`**(`--patch`)는 파일 단위가 아니라 변경 블록인 **헝크(Hunk)** 단위로 쪼개어 Staging Area에 올릴지 말지 물어본다.

```bash
$ git add -p src/auth.ts
(1/3) Stage this hunk [y,n,q,a,d,j,J,g,/,e,s,?]?
```

실무에서 가장 자주 쓰는 응답 키 다섯 가지만 익히면 충분하다.

- **`y` / `n`**: 이 변경 헝크를 다음 커밋에 포함(`y`) 또는 제외(`n`)한다.
- **`s` (Split)**: 한 헝크에 서로 무관한 두 변경이 붙어 있을 때 더 작은 헝크로 자동 분할한다.
- **`e` (Edit)**: 자동 분할조차 안 될 만큼 바로 인접한 줄이 섞였다면 에디터를 열어 `+`/`-` 줄을 수동으로 골라 스테이징한다.
- **`q` (Quit)**: 남은 헝크 검토를 즉시 종료하고 지금까지 `y`로 담은 것만 유지한다.

---

## 3. Stashing과 Cleaning: 컨텍스트 스위칭과 작업 디렉토리 청소 (7.3)

작업 도중 급하게 다른 브랜치로 체크아웃해 핫픽스를 봐야 할 때, 미완성 코드를 억지로 커밋하지 않고 스택에 잠시 치워 두는 도구가 **`git stash`** 다.

```bash
# 1. 추천 방식: 메시지를 붙이고 Untracked 새 파일까지 함께(-u) 임시 저장
$ git stash push -u -m "WIP: oauth login middleware"

# 2. 저장된 Stash 목록 확인
$ git stash list
stash@{0}: On feat/oauth: WIP: oauth login middleware

# 3. 복원 (apply는 스택에 남기고, pop은 복원과 동시에 스택에서 삭제)
$ git stash pop

# 4. (중요) Stash 복원 시 충돌이 나거나 현재 브랜치와 너무 멀어졌을 때: 새 브랜치로 안전 탈출
$ git stash branch fix/oauth-recovery stash@{0}
```

특히 4번 **`git stash branch <새브랜치명>`** 은 프로덕션 협업에서 매우 유용하다. Stash를 만들었던 원래 커밋 지점에서 새 브랜치를 따서 Stash를 풀고 성공 시 목록에서 지워 주므로, 현재 브랜치와 충돌로 고생할 위험을 피할 수 있다.

> **최신 Git 참고 (Git 2.51 Stash 표현 확장)**: **Git 2.51(2025.08)** 부터는 여러 개의 Stash 항목을 직전 Stash 커밋과 부모 포인터로 잇는 **커밋 시퀀스 기반 내부 표현 및 내보내기/가져오기(`export`/`import`) 형식**이 도입되어, 다른 머신이나 원격 저장소로 Stash 목록을 깔끔하게 주고받기 한층 수월해졌다.

반면 Stash가 아니라 **아예 추적되지 않는 빌드 잔여물이나 임시 파일을 지우고 싶다면 `git clean`** 을 쓴다. 단, 한 번 지운 Untracked 파일은 `reflog`로도 살릴 수 없으므로 반드시 **`-n`(`--dry-run`)으로 삭제 대상을 먼저 눈으로 확인한 뒤** `-fd`를 실행해야 안전하다.

```bash
# 1. 무엇이 지워질지 미리 시뮬레이션 (-d: 빈 폴더 포함, -x: .gitignore 파일까지 포함)
$ git clean -n -d

# 2. 확인 완료 후 실제 삭제 실행
$ git clean -f -d
```

---

## 4. 코드와 히스토리 정밀 검색: `git grep`과 `git log -S / -G / -L` (7.5)

IDE 검색이나 일반 `grep`은 `.git` 디렉토리 내부까지 훑거나 과거 커밋 시점의 코드를 찾지 못한다. Git 네이티브 검색 도구는 **색인 기반 초고속 검색**과 **특정 함수 단위 이력 추적**을 제공한다.

### ① 현재 및 과거 스냅샷 초고속 검색: `git grep`

```bash
# 줄 번호(-n)와 해당 줄이 속한 함수명(-p)까지 함께 출력
$ git grep -n -p "verifyToken"

# 특정 과거 태그(v1.8.0) 시점의 코드베이스 직접 검색
$ git grep -n "verifyToken" v1.8.0
```

### ② 언제 이 코드가 태어나고 사라졌는가: Pickaxe `-S` vs 정규식 `-G` vs 라인 `-L`

| 검색 명령어 | 동작 원리 | 실무 활용 상황 |
| :--- | :--- | :--- |
| **`git log -S "verifyJWT"`** | 해당 문자열의 **등장 횟수(Count) 자체가 증가하거나 감소한 커밋**만 필터링 | 특정 함수·변수가 **최초 도입되거나 완전히 삭제된 커밋** 찾기 |
| **`git log -G "JWT_.*_EXPIRE"`** | Diff의 추가(`+`)·삭제(`-`) 줄에 **정규식 패턴이 매칭되는 커밋 전부** 출력 | 단순 리팩토링·들여쓰기 수정 포함 정규식 변경 전수 추적 |
| **`git log -L :parseConfig:config.go`** | 파일 내 **특정 함수(`:함수명:경로`) 또는 줄 번호 범위(`10,40:경로`)** 만 추적 | 특정 함수가 만들어져서 지금까지 어떻게 진화했는지 서사 읽기 |

`-S`는 예를 들어 `foo`를 한 줄 지우고 다른 줄에 똑같이 `foo`를 옮겼다면 총 등장 횟수가 변하지 않으므로 건너뛴다. 반면 `-G`는 양쪽 모두 잡아낸다. 그리고 **`git log -L :함수명:파일경로`** 는 파일 이름이 바뀌어도 함수를 추적해 주므로 레거시 코드 파악 시 가장 강력한 무기다.

---

## 5. 내 작업에 신뢰 서명 남기기: GPG에서 SSH 커밋 서명 시대로 (7.4)

오픈소스 메인테이너나 보안 민감 조직에서는 "이 커밋과 태그를 진짜 본인 계정이 작성했는가"를 암호학적으로 증명해야 한다. 과거 Pro Git 2판에서는 복잡한 GPG 키쌍 발급(`gpg --gen-key`)과 `git config user.signingkey` 설정이 표준이었다.

하지만 Git `2.34` 버전 이후부터는 이미 GitHub 인증에 쓰는 **SSH 키를 그대로 커밋·태그 서명 키로 재사용**할 수 있게 되었다. 별도의 GPG 데몬이나 키 만료 관리 없이 3줄 설정만으로 GitHub 커밋 옆에 초록색 **`Verified`** 배지를 띄울 수 있다.

```bash
# 1. 서명 포맷을 GPG 대신 SSH로 전환 (Git 2.34+)
$ git config --global gpg.format ssh

# 2. 사용할 SSH 공개키 경로 지정
$ git config --global user.signingkey ~/.ssh/id_ed25519.pub

# 3. 모든 커밋과 태그 생성 시 자동 서명 (-S / -s 생략 가능)
$ git config --global commit.gpgsign true
$ git config --global tag.gpgsign true
```

*(단, GitHub 설정 > SSH and GPG keys 메뉴에 해당 공개키를 **Signing Key** 타입으로도 한 번 더 등록해야 검증 배지가 활성화된다.)*

---

## 마치며

이번 **7장 1부(`#11`)** 에서는 일상 업무 속도를 끌어올리고 실수를 예방하는 다섯 가지 도구(조상 탐색과 `reflog`, `git add -p`, `stash`/`clean`, `grep` & `log -S/-G/-L`, SSH 커밋 서명)를 정리했다.

이어지는 [7장 2부(#12)](/posts/pro-git-internals-ch7-part2/)에서는 수많은 개발자가 가장 헷갈려 하는 **`git reset`의 동작 원리를 Git의 3가지 트리(`HEAD` · `Index` · `Working Tree`) 관점에서 완벽 해부**하고, 히스토리 재작성의 꽃인 **`git rebase -i`** 와 현대 대용량 히스토리 정화 도구 **`git filter-repo`** 까지 파헤쳐 본다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 7.1~7.5 Git Tools](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection)
- [Git Documentation - git-reflog](https://git-scm.com/docs/git-reflog)
- [GitHub Docs - Telling Git about your SSH signing key](https://docs.github.com/en/authentication/managing-commit-signature-verification/telling-git-about-your-signing-key)
