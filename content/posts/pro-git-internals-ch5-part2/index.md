+++
date = '2026-11-28T09:00:00+09:00'
draft = false
title = 'Git 프로젝트 운영 #9: A...B 비교와 cherry-pick'
description = 'Pro Git 5장 후반부(5.3)를 바탕으로 메인테이너의 Topic 브랜치 검증, 이중 점(A..B)과 삼중 점(A...B) 차이, cherry-pick·rerere·릴리즈 관리를 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Maintainer', 'CodeReview', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#8)]({{< ref "/posts/pro-git-internals-ch5-part1" >}})에서는 **Pro Git 5장 전반부**(5.1~5.2)를 통해 분산형(DVCS) 아키텍처의 본질과 3대 분산 워크플로우, 그리고 **기여자**(Contributor) 입장에서 지켜야 할 커밋 규율과 Topic 브랜치 격리 전략을 살펴보았다.

이번 2부에서는 시선을 정반대 편에 앉아 있는 **프로젝트 메인테이너(Maintainer)와 시니어 코드 리뷰어**로 옮긴다(**Pro Git 5.3 Maintaining a Project**). 매일 쏟아지는 Pull Request와 패치들을 메인 브랜치 안정성을 해치지 않으면서 검증하고, 리뷰어가 진짜 살펴봐야 할 변경분만 정확히 발라내며(`A..B` vs `A...B`), 필요한 커밋만 이식(`git cherry-pick`)하거나 반복 충돌을 자동 해결(`git rerere`)해 깔끔한 릴리즈로 마무리하는 기술까지 깊이 있게 정리한다.

---

## 1. 메인테이너의 격리 검증 원칙과 패치 통합의 진화

기여자로부터 새 코드를 전달받았을 때 메인테이너가 가장 피해야 할 행동은 `master`나 `main` 위에 곧바로 패치를 덮어씌우는 것이다. 실력이 뛰어난 메인테이너는 언제나 **기여자의 이니셜이나 주제명이 붙은 임시 Topic 브랜치**를 로컬에 따로 열어 먼저 테스트한다.

```bash
$ git checkout -b sc/ruby_client master
```

Pro Git 5.3이 쓰인 리눅스 커널 및 초기 Git 생태계에서는 기여자가 이메일 메일링 리스트로 보낸 패치를 적용하기 위해 두 가지 도구를 사용했다.

- **`git apply`**: 단순한 `patch` 명령과 달리 파일 변경·생성·삭제와 권한 변경까지 트랜잭션처럼 검증한다. **`git apply --check`** 를 먼저 실행하면 작업 트리를 조금도 건드리지 않고 패치가 깔끔하게 붙는지 미리 점검할 수 있다(단, 커밋 객체를 자동으로 만들지는 않는다).
- **`git am` (Apply Mailbox)**: `git format-patch`로 생성된 mbox 이메일 포맷을 읽어 변경 사항뿐 아니라 **원작자의 이름, 이메일, 작성 시각, 커밋 메시지까지 그대로 보존해 커밋을 생성**한다. 충돌이 발생하면 3-Way Merge 옵션(`git am -3`)이나 `git am --resolved`로 해결한 뒤 다음 패치로 넘어간다.

오늘날 GitHub · GitLab 환경에서는 이메일 파일 대신 웹 기반 Pull Request가 표준이 되었지만, **로컬에서 독립 Topic 브랜치를 생성해 안전하게 검증하는 본질은 똑같다.** GitHub 저장소에서는 아래 한 줄만으로 남의 PR을 로컬 격리 브랜치로 바로 내려받아 실행·디버깅할 수 있다.

```bash
# 123번 Pull Request를 로컬 sc/ruby_client 브랜치로 가져와 격리 테스트
$ git fetch origin pull/123/head:sc/ruby_client
$ git checkout sc/ruby_client

# GitHub CLI(gh)를 사용하는 현대 단축 명령
$ gh pr checkout 123
```

---

## 2. 기여 코드 정밀 해부: 이중 점(`A..B`) vs 삼중 점(`A...B`)과 `merge-base`

이제 격리된 Topic 브랜치(`contrib` 또는 `feature`)에 올라온 코드를 본격적으로 리뷰할 차례다. 여기서 수많은 개발자가 실수하는 함정이 하나 있다. 바로 **이중 점(`..`)과 삼중 점(`...`) 표기법이 `git log`에서와 `git diff`에서 정반대에 가까울 정도로 다르게 동작한다**는 사실이다.

{{< figure src="dot-ranges-and-maintainer-branches.svg" alt="git log와 git diff에서의 이중 점(A..B)과 삼중 점(A...B) 범위 비교 및 대규모 프로젝트 메인테이너 4단계 통합 브랜치 다이어그램" caption="공통 조상 C2 기준 git log vs git diff의 점(..) 표기 해석 차이(상단)와 Git 코어 프로젝트의 4단계 통합 브랜치 파이프라인(하단) (Pro Git 5.3 재구성)" width="820" height="540" >}}

`C2`를 공통 조상(`merge-base`)으로 삼아 `master`는 `C3 ← C5`로, `feature`는 `C4 ← C6`으로 분기한 그래프를 기준으로 두 명령어의 차이를 비교해 보자.

| 명령어 | 내부 해석 의미 | 결과 / 출력 대상 | 실무 리뷰 용도 |
| :--- | :--- | :--- | :--- |
| **`git log master..feature`** | `feature`에는 도달 가능하지만 `master`에는 없는 커밋 집합 (`^master feature`) | `{C4, C6}` | **이 PR로 새로 도입될 커밋 목록 확인** |
| **`git log --left-right master...feature`** | 공통 조상 `C2` 이후 한쪽에만 속한 대칭 차집합(Symmetric Difference) | `< C3, < C5` (`master`) / `> C4, > C6` (`feature`) | 양쪽 브랜치가 갈라진 뒤 각자 무엇을 했는지 대조 |
| **`git diff master..feature`** | 두 브랜치 **끝점(`C5` vs `C6`) 스냅샷의 직접 차이** | `C6 - C5` (역전된 `master` 수정분까지 섞임) | 양 끝의 최종 코드 상태만 비교할 때 (PR 리뷰 부적합) |
| **`git diff master...feature`** | **공통 조상 `merge-base(C2)`와 `feature(C6)`의 차이** | `C6 - C2` (**`feature`가 바꾼 순수 변경분**) | **PR 코드 리뷰의 표준 (GitHub Files changed 원리)** |

### 왜 PR 리뷰에는 `git diff master..feature`가 아니라 `master...feature`를 쓰는가?

내가 `feature` 브랜치를 딴 뒤 며칠 동안 `master`에 다른 동료들의 커밋 `C3`, `C5`가 먼저 머지되었다고 가정해 보자. 이 상태에서 무심코 `git diff master..feature`를 치면, Git은 `master`의 최신 스냅샷 `C5`와 `feature`의 끝점 `C6`를 그냥 빼 버린다. 그 결과 리뷰 화면에는 **내가 만든 변경분(`C4`, `C6`)뿐 아니라, 내가 아직 반영하지 않은 동료의 코드(`C3`, `C5`)를 마치 내가 삭제한 것처럼 뒤섞인 diff**가 나타난다.

반면 `git diff master...feature`처럼 **점 3개(`...`)** 를 찍으면 Git은 먼저 `git merge-base master feature`를 계산해 두 브랜치가 갈라졌던 분기점 `C2`를 찾고, 정확히 `C2`와 `C6` 사이를 비교한다. 즉 아래 두 명령어는 내부적으로 완전히 똑같은 결과를 낸다.

```bash
# 삼중 점 diff로 기여자가 만든 순수 변경분만 확인
$ git diff master...feature

# 내부적으로 Git이 수행하는 실제 동치 명령
$ git diff $(git merge-base master feature) feature
```

바로 이 **`git diff base...head`** 메커니즘이 오늘날 GitHub Pull Request의 **"Files changed" 탭이 동작하는 핵심 원리**다. 메인 브랜치에 새 커밋이 계속 쌓이더라도 PR 리뷰어가 오직 기여자가 작성한 순수 변경분만 볼 수 있는 이유가 여기에 있다.

---

## 3. 대규모 통합 파이프라인과 `git cherry-pick` · `git rerere`의 내부 원리

### ① 대규모 프로젝트 메인테이너의 4단계 통합 브랜치 (`maint` · `master` · `next` · `seen`)

Git 코어 프로젝트나 리눅스 커널처럼 수백 개의 Topic 브랜치가 동시에 들어오는 거대 프로젝트에서는 Pro Git 3.4([3장 2부]({{< ref "/posts/pro-git-internals-ch3-part2" >}}))에서 보았던 Long-Running 브랜치 전략을 한 단계 더 정교하게 확장한다.

- **`seen` (과거 이름 `pu`, Proposed Updates)**: 아직 완전히 검증되지 않았지만 일단 모아 두는 실험적 통합 브랜치다. 메인테이너는 매일 수많은 PR을 여기에 시험 삼아 합쳐 보며 **여러 토픽 간에 어떤 파일에서 충돌이 나는지** 조기에 파악한다. 주기적으로 이력을 되감아(Rewind/Rebase) 재구축하므로 외부 기여자가 기반 브랜치로 삼아서는 안 된다.
- **`next`**: 1차 코드 리뷰를 통과하고 다음 정식 버전 후보로 올라온 토픽들이 모이는 통합 검증 브랜치다. 자동화 CI 테스트와 커뮤니티 베타 테스터들의 실사용 검증이 여기에서 이루어진다.
- **`master`**: 검증이 완전히 끝나 **언제든 릴리즈할 수 있는 안정 상태**의 토픽만 최종 진급(Graduate)하는 기준선이다.
- **`maint` (Maintenance의 약자, 구버전 유지보수 브랜치)**: `main`이나 `master`의 오타가 아니라, 직전 정식 릴리즈에서 분기해 유지하는 **장기 유지보수(Maintenance) 전용 브랜치**다. 새 기능을 넣지 않고 구버전 사용자들을 위한 치명적 버그 수정과 보안 패치만 반영해 포인트 릴리즈(`v2.x.1`)를 발행한다.

---

### ② 핀셋 커밋 이식: `git cherry-pick`의 내부 원리, 실무 옵션, 그리고 터미널 실습

기여자가 보낸 Topic 브랜치 전체를 아직 병합할 수는 없지만, 그중 **특정 버그 수정 커밋 1개(`e43a6fd`)만 먼저 본류 브랜치(`main` / `master`)나 구버전 유지보수·릴리즈 브랜치(`maint`, `release/1.x`)로 가져오고 싶을 때**는 `git cherry-pick`을 사용한다.

```bash
# main(또는 master) 브랜치로 이동해 특정 커밋 e43a6fd만 핀셋 이식
$ git switch main
$ git cherry-pick e43a6fd
[main a1b2c3d] Fix race condition in cache lookup
 1 file changed, 12 insertions(+), 3 deletions(-)
```

실무에서 `git cherry-pick`이 가장 자주 쓰이는 세 가지 대표 시나리오는 다음과 같다.

1. **구버전 릴리즈(`maint`, `release/1.x`)로 핫픽스 백포팅**: 본류(`main`)에서 보안 패치나 치명적 버그를 고친 커밋만 골라, 신규 기능(`C3`, `C4`) 유입 없이 과거 LTS 버전에 이식할 때 사용한다.
2. **장기 `feature` 브랜치 속 공통 버그 수정 선반영**: 아직 완성되지 않아 머지할 수 없는 기능 브랜치 안에서 우연히 함께 고친 공통 유틸·설정 버그 커밋만 `main`에 먼저 합류시킬 때 유용하다.
3. **실수로 잘못된 브랜치에 만든 커밋 복구**: `main`에 올려야 할 커밋을 다른 브랜치에 잘못 커밋했을 때, 해당 커밋을 올바른 브랜치로 `cherry-pick`한 뒤 원래 브랜치에서는 `git reset --hard HEAD~1`로 되돌린다.

{{< figure src="cherry-pick-rerere-and-describe.svg" alt="git cherry-pick 단일 커밋 델타 이식, git rerere 충돌 해시 캐시 자동 해결, 그리고 git describe 빌드 식별자 해부 다이어그램" caption="git cherry-pick(좌상단)과 git rerere 자동 충돌 재적용 메커니즘(우상단), 그리고 git describe 빌드 번호 해부 및 릴리즈 패키징 도구 체인(하단) (Pro Git 5.3 재구성)" width="820" height="540" >}}

#### 3-Way Merge 기반 내부 동작 원리

`git cherry-pick`은 내부적으로 **`git rebase` 1개 커밋 버전과 동일한 3-Way Merge 메커니즘**으로 동작한다.

1. 대상 커밋 `C4`(`e43a6fd`)와 그 직전 부모 커밋 `C3`의 차이를 비교해 **`C4`가 도입한 변경 델타만 추출**한다(`C3` 자체나 이후 커밋 `C6`는 무시한다).
2. `C3`를 공통 조상(Base)으로 놓고 현재 체크아웃된 브랜치의 끝(`C5`)과 `C4`를 대상으로 **3-Way Merge**를 수행한다.
3. 그 결과를 현재 브랜치 위에 **새로운 커밋 객체 `C4'`(`a1b2c3d`)로 생성**한다.

파일 변경분은 같더라도 부모 커밋과 커미터(Committer) 메타데이터가 달라지므로 원본 `e43a6fd`와 SHA-1 해시는 전혀 다른 별개의 커밋이 된다. 따라서 [Rebase Golden Rule]({{< ref "/posts/pro-git-internals-ch3-part3" >}})에서 살펴본 것처럼, `cherry-pick` 후 나중에 원래 토픽 브랜치 전체를 `merge`하면 같은 변경이 두 커밋에 중복 존재해 충돌하거나 히스토리가 지저분해질 수 있으므로 **백포팅이나 선별 이식 등 목적이 명확할 때 절제해서 사용하는 것**이 좋다.

#### 자주 쓰는 `git cherry-pick` 핵심 옵션과 범위 문법

| 명령어 / 옵션 | 동작 설명 | 실무 활용 팁 |
| :--- | :--- | :--- |
| `git cherry-pick <해시>` | 지정한 단일 커밋의 변경분만 현재 브랜치에 새 커밋으로 적용 | 가장 기본적인 단건 이식 |
| **`git cherry-pick -x <해시>`** | 커밋 메시지 끝에 `(cherry picked from commit <원본해시>)` 자동 추가 | **실무 권장 옵션** · 나중에 원본 출처 추적 가능 |
| `git cherry-pick A B` | 띄어쓰기로 구분한 여러 커밋(`A`, `B`)을 순서대로 이식 | 독립적인 2~3개 패치 동시 적용 |
| `git cherry-pick A..B` | `A`를 **제외하고** `A` 다음 커밋부터 `B`까지 연속 범위 이식 | 연속 구간 백포팅 (`A` 포함 시 **`A^..B`** 사용) |
| **`git cherry-pick -n <해시>`** | 커밋하지 않고 변경 사항만 Working Tree와 Index에 반영 (`--no-commit`) | 여러 커밋을 하나로 합치거나 추가 수정 후 커밋할 때 |
| `git cherry-pick -e <해시>` | 커밋 적용 전 에디터를 열어 커밋 메시지 수정 (`--edit`) | 대상 브랜치 문맥에 맞게 메시지 보완 |
| `--continue` / `--abort` / `--skip` | 충돌 해결 후 진행 / 작업 전 상태로 취소 / 현재 커밋 건너뛰기 | 다중 `cherry-pick` 중 충돌 제어 |

#### 따라 해 보는 `git cherry-pick` 터미널 실습 (`cherry-pick-lab`)

아래 명령을 빈 디렉토리에서 순서대로 실행하면, 미완성 기능 커밋(`C3`, `C5`)은 제외하고 그사이에 낀 버그 수정 커밋(`C4`)만 `main`으로 이식되는 과정과 `-x` 추적 기록, 그리고 충돌 처리까지 한 번에 확인할 수 있다.

```bash
# 1. 실습 저장소 초기화 및 main 기본 커밋 2개 생성 (C1, C2)
$ mkdir cherry-pick-lab && cd cherry-pick-lab
$ git init -b main

$ echo "app-v1" > app.txt
$ git add app.txt && git commit -m "C1: Initial app"

$ echo "timeout=5000" > config.env
$ git add config.env && git commit -m "C2: Add default config"

# 2. feature/login 브랜치에서 미완성 기능(C3, C5) 사이에 버그 수정 커밋(C4) 작성
$ git switch -c feature/login
$ echo "login-wip" > login.txt
$ git add login.txt && git commit -m "C3: Start login feature (WIP)"

$ echo "timeout=10000" > config.env
$ git add config.env && git commit -m "C4: Fix timeout config bug"

# 방금 만든 C4의 7자리 해시 확인 (아래 단계에서 <C4-해시> 자리에 입력)
$ git log -n 1 --oneline
e43a6fd C4: Fix timeout config bug

$ echo "oauth-wip" >> login.txt
$ git add login.txt && git commit -m "C5: Add oauth draft (WIP)"
```

이제 `main` 브랜치로 돌아와 `feature/login`의 `C3`, `C5`는 가져오지 않고 **`C4` 커밋 하나만 `-x` 옵션으로 이식**해 보자.

```bash
# 3. main으로 복귀 후 C4 해시만 cherry-pick (-x로 원본 해시 기록)
$ git switch main
$ git cherry-pick -x <C4-해시>
[main 9f2d1a8] C4: Fix timeout config bug
 1 file changed, 1 insertion(+), 1 deletion(-)

# 4. 결과 확인: login.txt(C3, C5) 없이 config.env 변경분만 정확히 반영됨
$ ls
app.txt  config.env

$ cat config.env
timeout=10000

# 5. 브랜치 그래프와 새 커밋 해시(9f2d1a8 != e43a6fd) 및 -x 메타데이터 확인
$ git log --oneline --graph --all
* 9f2d1a8 (HEAD -> main) C4: Fix timeout config bug
| * 7c1b302 (feature/login) C5: Add oauth draft (WIP)
| * e43a6fd C4: Fix timeout config bug
| * 4a8d019 C3: Start login feature (WIP)
|/
* 2b9e411 C2: Add default config
* 1a0c500 C1: Initial app

$ git show -s HEAD
...
    C4: Fix timeout config bug

    (cherry picked from commit e43a6fd...)
```

마지막으로 체리픽 도중 **충돌(Conflict)** 이 발생하는 상황도 직접 재현해 보자. 현재 `main`의 `config.env`를 다른 값(`timeout=3000`)으로 먼저 바꾼 뒤, 똑같은 줄을 고친 `C4`를 다시 적용하려 하면 3-Way Merge 충돌이 일어난다.

```bash
# 6. main에서 같은 줄을 다르게 수정해 충돌 조건 만들기
$ echo "timeout=3000" > config.env
$ git add config.env && git commit -m "Set timeout to 3000 on main"

$ git cherry-pick <C4-해시>
CONFLICT (content): Merge conflict in config.env
error: could not apply e43a6fd... C4: Fix timeout config bug
hint: After resolving the conflicts, mark them with
hint: "git add <paths>" or "git rm <paths>"
hint: and run "git cherry-pick --continue"

# 7-A. 충돌 해결 후 계속 진행할 때
$ echo "timeout=10000" > config.env
$ git add config.env
$ git cherry-pick --continue

# 7-B. (또는) 체리픽을 포기하고 직전 상태로 되돌릴 때
$ git cherry-pick --abort
```

---

### ③ 반복되는 충돌 해결의 해방: `git rerere` (Reuse Recorded Resolution)

대규모 프로젝트의 메인테이너가 매일 `seen`이나 `next` 브랜치를 새로 조립할 때, 혹은 기여자가 Rebase 중 충돌을 고치다가 다시 Merge 방식으로 전환할 때 똑같은 충돌을 며칠 연속으로 수동 해결하는 일은 엄청난 시간 낭비다. 이때 빛을 발하는 Git의 숨은 병기가 바로 **`git rerere`**("Reuse Recorded Resolution")다.

```bash
# 한 번만 전역 활성화해 두면 이후 모든 Merge / Rebase 충돌에 자동 작동
$ git config --global rerere.enabled true
```

`rerere`가 켜져 있으면 Git은 프로젝트의 `.git/rr-cache` 디렉토리에 충돌 해결 패턴을 3단계로 학습한다.

1. **Pre-image 기록**: Merge나 Rebase 도중 충돌 마커(`<<<<<<< ... ======= ... >>>>>>>`)가 생기면, Git은 해당 충돌 블록 양쪽 내용의 SHA-1 지문을 계산해 `.git/rr-cache/<해시>/preimage`에 저장한다. (`Recorded preimage for '...'` 메시지 출력)
2. **Post-image 기록**: 개발자가 충돌을 해결한 뒤 스테이징하고 커밋을 완료하면, Git은 해결된 최종 코드 블록을 같은 지문 디렉토리 아래 `postimage`로 저장해 둔다. (`Recorded resolution for '...'` 메시지 출력)
3. **자동 재적용**: 며칠 뒤 다른 브랜치 통합이나 Rebase 과정에서 **정확히 같은 충돌 Pre-image 지문이 다시 나타나면**, Git은 `.git/rr-cache`에서 저장된 `postimage`를 꺼내 작업 트리 파일에 즉시 자동 적용한다(`Resolved '...' using previous resolution.` 메시지 출력). 단, 자동으로 `git add`까지 완료해 주지는 않으므로 개발자가 `git diff`로 최종 결과만 한 번 눈으로 검수한 뒤 `git commit`(또는 `git rebase --continue`)으로 확정할 수 있어 안전성까지 챙겼다.

---

## 4. 릴리즈 엔지니어링: 서명 태그, `git describe` 빌드 번호, `git archive`, `git shortlog`

성공적으로 코드 통합을 마쳤다면 이제 사용자에게 정식 버전을 배포할 차례다. Pro Git 5.3은 메인테이너가 릴리즈를 발행할 때 챙겨야 할 네 가지 핵심 엔지니어링 도구를 짚어 준다.

### ① 서명 태그(`git tag -s`)와 신뢰 검증의 현대화

누군가 릴리즈 파일과 태그를 변조하지 않았음을 증명하려면 경량 태그가 아닌 **전자 서명된 Annotated 태그**(`git tag -s v1.5 -m "my signed tag"`)를 남겨야 한다. 과거 Pro Git 2판 집필 당시에는 저장소 자체에 메인테이너의 공개키를 Blob 객체로 커밋해 `git cat-file`로 꺼내 검증하는 수동 패턴(`maintainer-pgp-pub`)도 소개했지만, 오늘날 GitHub/GitLab 환경에서는 계정 설정에 등록된 **GPG 키 또는 SSH 서명 키(`gpg.format = ssh`)** 를 플랫폼이 자동으로 대조해 태그 옆에 **`Verified`** 배지를 띄워 주는 방식으로 훨씬 간편하고 안전하게 진화했다.

### ② SHA-1 해시를 사람이 읽는 빌드 번호로 바꾸는 마법: `git describe`

Git은 Subversion처럼 커밋 번호가 `r1234` 식으로 순차 증가하지 않고 40자리 SHA-1 해시를 쓴다. 그런데 배포 패키지 파일 이름이나 장애 리포트 로그에는 `"지금 빌드된 바이너리가 지난 릴리즈 이후 얼마나 진행되었는지"` 한눈에 파악할 수 있는 **가독성 높은 빌드 식별자**가 필요하다. `git describe`가 바로 그 역할을 한다.

```bash
$ git describe master
v1.6.2-rc1-20-g2414721
```

이 문자열 하나에는 네 가지 핵심 정보가 정교하게 압축되어 있다.

- **`v1.6.2-rc1`**: 현재 커밋에서 부모를 거슬러 올라갈 때 **가장 가깝게 도달 가능한 Annotated 태그** 이름이다. (주의: 일반 Lightweight 태그는 기본적으로 무시하며, 포함하려면 `--tags` 플래그를 주어야 한다. 만약 현재 `HEAD`가 태그가 달린 커밋 정확히 그 위를 가리키고 있다면 뒤의 숫자와 해시 없이 태그 이름 `v1.6.2-rc1`만 단독 출력된다.)
- **`20`**: 해당 태그(`v1.6.2-rc1`) 이후 현재 커밋(`master`)까지 도달하기 위해 추가로 쌓인 **커밋 개수**다.
- **`g`**: 뒤의 16진수 문자열이 다른 VCS가 아닌 **Git**의 커밋 해시임을 알리는 고정 접두사다.
- **`2414721`**: 현재 체크아웃된 커밋의 **7자리 축약 SHA-1 해시**다.

실무 빌드 파이프라인(Makefile, Gradle, Go `-ldflags`)에서는 종종 `git describe --tags --always --dirty` 옵션을 결합해 사용한다. `--always`는 저장소에 태그가 하나도 없을 때도 최소한 축약 해시를 출력해 빌드가 깨지지 않게 막아 주며, `--dirty`는 커밋되지 않은 로컬 수정 상태에서 빌드했을 때 끝에 `-dirty` 꼬리표를 붙여 추적을 돕는다.

### ③ 배포용 순수 아카이브와 자동 체인지로그: `git archive` & `git shortlog`

릴리즈 소스 파일을 배포할 때 저장소 디렉토리를 그대로 압축하면 수백 MB에 달하는 `.git` 히스토리 폴더까지 함께 묶여 나간다. **`git archive`** 는 특정 커밋 스냅샷에서 `.git` 메타데이터와 무시 파일(`export-ignore`)을 깔끔히 제외한 순수 소스 아카이브를 즉시 생성한다(GitHub Releases 페이지에서 자동 제공하는 *Source code (zip/tar.gz)* 다운로드 링크가 내부적으로 사용하는 메커니즘이다).

```bash
# 릴리즈용 .tar.gz 및 .zip 소스 아카이브 생성 (--prefix로 압축 해제 폴더명 지정)
$ git archive master --prefix='project/' | gzip > $(git describe master).tar.gz
$ git archive master --prefix='project/' --format=zip > $(git describe master).zip
```

마지막으로 지난 정식 릴리즈(`v1.0.1`) 이후 누가 어떤 개선 사항을 반영했는지 정리해 릴리즈 노트를 쓸 때는 **`git shortlog`** 를 활용한다. `--no-merges` 옵션을 주면 병합 커밋 잡음을 거르고 기여자(Author) 이름별로 알파벳 순 정렬된 깔끔한 변경 요약을 한 줄 명령으로 뽑아낼 수 있다.

```bash
# v1.0.1 이후 v1.1.0까지 기여자별 순수 변경 내역 요약
$ git shortlog --no-merges master --not v1.0.1
Chris Wanstrath (6):
      Add support for ruby 1.9
      Fix edge case in config parser
...
Scott Chacon (8):
      Update documentation for CLI options
      ...
```

---

## 마치며

이로써 2부작에 걸쳐 **Pro Git 5장(Distributed Git)** 전체를 **기여자의 시각(#8)** 과 **메인테이너의 시각(#9)** 양쪽에서 모두 해부했다.

- **프로젝트 운영의 핵심은 안전 격리와 정밀 검증**이다. 메인테이너는 모든 기여 PR을 별도 Topic 브랜치로 격리해 검증하고, `git diff master...feature`(`merge-base` 비교)로 순수 변경분만 리뷰해야 한다.
- **규모가 커질수록 도구가 노동을 대신한다**. `seen · next · master · maint` 다층 파이프라인과 `git cherry-pick` 백포팅, `.git/rr-cache`에 충돌 해법을 학습하는 `git rerere`를 결합하면 하루 수십 건의 통합 작업도 일관되게 처리할 수 있다.
- **릴리즈 엔지니어링까지 Git 네이티브 도구로 완결된다**. 서명 태그(`git tag -s`), 사람이 읽는 빌드 번호(`git describe`), 순수 소스 패키징(`git archive`), 기여자별 체인지로그(`git shortlog`)가 결합해 투명하고 재현 가능한 배포 파이프라인을 완성한다.

1장에서 다룬 스냅샷 객체 모델부터 5장의 분산 협업 및 프로젝트 운영까지 이해했다면, 이제 Git을 단순한 백업 명령어 모음이 아니라 하나의 완성된 소프트웨어 엔지니어링 플랫폼으로 다룰 준비가 끝났다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 5.3 Distributed Git - Maintaining a Project](https://git-scm.com/book/en/v2/Distributed-Git-Maintaining-a-Project)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 5.4 Distributed Git - Summary](https://git-scm.com/book/en/v2/Distributed-Git-Summary)
- [Git Documentation - git-rev-parse (Specifying Ranges: `..` and `...`)](https://git-scm.com/docs/git-rev-parse#_specifying_ranges)
- [Git Documentation - gitworkflows (Graduation of Topic Branches)](https://git-scm.com/docs/gitworkflows)
