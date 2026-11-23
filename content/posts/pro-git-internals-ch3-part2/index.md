+++
date = '2026-11-23T09:00:00+09:00'
draft = false
title = 'Git 브랜치 원리 #5: 리모트 트래킹 브랜치와 업스트림 추적'
description = 'Pro Git 3장 중반부를 바탕으로 git branch --merged 필터링, Long-Running·Topic 브랜치, 리모트 트래킹 브랜치(origin)와 업스트림 추적을 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글]({{< ref "/posts/pro-git-internals-ch3-part1" >}})에서는 Git 브랜치가 `.git/refs/heads/` 아래 놓인 41바이트짜리 포인터 파일에 불과하다는 사실과 함께, 이력이 직선일 때 포인터만 밀어 올리는 Fast-forward와 분기된 양쪽 끝을 공통 조상과 대조하는 3-Way Merge의 동작 원리를 살펴보았다.

이번 글에서는 **Pro Git 3장 중반부**(3.3~3.5)를 다룬다. 실무에서 수십 개의 브랜치가 쌓였을 때 병합 완료 여부에 따라 안전하게 정리하고 이름을 바꾸는 **브랜치 관리 기법**, Git의 가벼운 브랜치 모델을 극대화하는 **Long-Running 및 Topic 브랜칭 워크플로우**, 그리고 로컬 저장소와 원격 저장소를 연결하는 핵심 장치인 **리모트 트래킹 브랜치**(Remote-tracking Branch)와 업스트림 추적 원리까지 깊이 있게 정리한다.

---

## 1. 실전 브랜치 관리: 병합 상태 필터링과 안전한 이름 변경 (Pro Git 3.3)

프로젝트가 진행될수록 브랜치는 빠르게 불어난다. 단순히 브랜치를 만들고 지우는 것을 넘어, 어떤 브랜치가 이미 통합되었고 어떤 브랜치에 아직 합치지 못한 커밋이 남아 있는지 파악하는 도구가 필요하다.

### 브랜치 목록 조회와 병합 여부 필터링: `-v`, `--merged`, `--no-merged`

아무 인자 없이 `git branch`를 실행하면 현재 로컬 브랜치 목록을 보여 주며, **`HEAD`가 가리키는 현재 체크아웃된 브랜치** 앞에는 `*` 기호가 표시된다. 각 브랜치가 가리키는 마지막 커밋 해시와 메시지까지 함께 보려면 `-v` 옵션을 붙인다.

```bash
$ git branch -v
  iss53   93b412c Fix javascript issue
* master  7a98805 Merge branch 'iss53'
  testing 782fd34 Add scott to the author list in the readme
```

여기서 브랜치 청소를 안전하게 돕는 두 가지 핵심 필터 옵션이 `--merged`와 `--no-merged`다.

1. **`git branch --merged`**: 현재 브랜치에 **이미 병합이 완료된** 브랜치만 골라 출력한다. 이 목록에서 `*` 표시가 없는 브랜치는 다른 브랜치에 이미 모든 이력이 통합된 상태이므로, `git branch -d`로 삭제해도 작업 내용을 잃을 염려가 없다.
2. **`git branch --no-merged`**: 현재 브랜치에 **아직 병합되지 않은 커밋**이 남아 있는 브랜치만 필터링한다. 이 목록에 있는 브랜치는 `git branch -d`로 삭제하려고 해도 Git이 데이터 손실을 막기 위해 거부하며, 정말 버려도 되는 실험 브랜치라면 강제 삭제 옵션인 `-D`를 명시해야 한다.

```bash
$ git branch --no-merged
  testing

$ git branch -d testing
error: The branch 'testing' is not fully merged.
If you are sure you want to delete it, run 'git branch -D testing'.
```

참고로 다른 브랜치로 굳이 체크아웃하지 않더라도, `--merged`나 `--no-merged` 뒤에 기준 브랜치 이름을 인자로 넘겨 특정 브랜치 기준으로 병합 여부를 조회할 수 있다. 예를 들어 `testing` 브랜치에 머무른 상태에서 `master`에 아직 반영되지 않은 브랜치를 확인하려면 다음과 같이 실행한다.

```bash
$ git branch --no-merged master
  topicA
  featureB
```

### 로컬 및 원격 브랜치 이름 변경 절차: `--move`

잘못 지은 로컬 브랜치 이름(`bad-branch-name`)을 히스토리를 그대로 유지한 채 `corrected-branch-name`으로 바꾸고 싶다면 `git branch --move`(또는 `-m`)를 사용한다. 단, **다른 협업자가 함께 사용 중인 원격 브랜치나 기본 브랜치(`master`, `main`) 이름을 바꿀 때는 각별한 주의가 필요**하다.

```bash
# 1. 로컬 브랜치 이름 변경
$ git branch --move bad-branch-name corrected-branch-name

# 2. 새 이름의 브랜치를 원격에 푸시하고 업스트림 연결
$ git push --set-upstream origin corrected-branch-name

# 3. 원격에 남아 있는 옛 이름의 브랜치 삭제
$ git push origin --delete bad-branch-name
```

만약 기존 기본 브랜치였던 `master`를 `main`으로 전환하려는 경우에는 `git branch --move master main` 및 `git push --set-upstream origin main` 이후에도 원격 저장소 호스팅 설정(기본 브랜치 지정, 브랜치 보호 규칙), CI/CD 빌드·배포 스크립트, 테스트 설정, 문서 내 브랜치 참조, 열린 Pull Request의 타겟 브랜치까지 모두 갱신한 뒤에 마지막으로 `git push origin --delete master`를 실행해야 장애를 피할 수 있다.

---

## 2. 브랜칭 워크플로우: Long-Running vs Topic 브랜치 (Pro Git 3.4)

브랜치 생성과 3-Way Merge 비용이 사실상 0에 수렴하는 Git의 특성은 팀의 개발 사이클을 전혀 다른 형태로 바꾼다. Pro Git에서는 이를 크게 **Long-Running 브랜치**와 **Topic 브랜치** 모델로 설명한다.

### Long-Running 브랜치: 안정성 단계(Silo)에 따른 포인터 진급

Git은 3-Way Merge를 통해 한 브랜치에서 다른 브랜치로 장기간에 걸쳐 반복 병합하는 작업을 매우 깔끔하게 처리한다. 덕분에 개발 파이프라인의 **안정성 수준(Stability Level)에 따라 항상 열어 두는 영속 브랜치**를 여러 개 두는 방식이 가능하다.

- **`master` 브랜치**: 배포되었거나 곧 배포할 만큼 **완벽히 검증되고 안정적인 코드**만 유지한다.
- **`develop` (또는 `next`) 브랜치**: 기능 테스트와 통합을 검증하는 단계다. 항상 안정적이지는 않지만, 테스트를 통과해 안정 상태에 도달할 때마다 `master`로 병합한다.
- **`pu` (Proposed Updates) 브랜치**: 규모가 큰 프로젝트에서는 아직 `develop`이나 `master`에 들어갈 준비가 덜 된 실험적 통합 브랜치를 한 단계 더 앞단에 두기도 한다.

내부적으로 보면 이는 커밋 그래프를 따라 왼쪽에서 오른쪽으로 커밋이 쌓일 때, 검증이 끝난 지점까지 `master` → `develop` → `pu` 포인터들이 차례로 앞을 쫓아 **진급**(Graduate)하는 구조다.

### Topic 브랜치: 빠른 컨텍스트 스위칭과 격리된 실험

**Topic 브랜치**는 프로젝트 규모와 무관하게 모든 팀에서 빛을 발하는 단기 수명(Short-lived) 브랜치다. 단일 이슈나 특정 기능 단위로 브랜치를 만들고, 작업과 코드 리뷰가 끝나 병합하면 즉시 삭제한다.

예를 들어 `master`에서 `iss91` 이슈 브랜치를 따서 작업(`C4`)하다가 다른 접근 방식을 시험해 보기 위해 `iss91v2`(`C7 ← C8`)를 분기하고, 그사이에 `master`로 돌아와 또 다른 아이디어용 브랜치 `dumbidea`(`C9`)를 실험하는 상황을 생각해 보자.

검토 결과 원래 `iss91`(`C5 ← C6`)보다 두 번째 대안인 `iss91v2`가 낫고 `dumbidea`도 훌륭하다면, Git에서는 `iss91`만 그대로 버리고 `dumbidea`와 `iss91v2`만 **선택적으로 `master`에 병합**(`C10`, `C11`)할 수 있다. 무엇보다 이 모든 분기·실험·병합 과정이 **원격 서버와의 네트워크 통신 없이 100% 로컬 저장소 안에서만** 일어난다.

{{< figure src="workflows-and-remote-branches.svg" alt="Git 브랜칭 워크플로우(Long-Running, Topic)와 리모트 트래킹 브랜치(origin/master) 동기화 다이어그램" caption="안정성 단계별 Long-Running/Topic 브랜치 워크플로우와 git fetch/merge 시 리모트 트래킹 브랜치(origin/master)의 이동 원리 (Pro Git 3.4~3.5 재구성)" width="820" height="540" >}}

---

## 3. 리모트 브랜치와 트래킹(Tracking) 메커니즘 (Pro Git 3.5)

지금까지 살펴본 로컬 브랜치와 달리, 협업 시에는 서버에 존재하는 **리모트 참조**(Remote Reference)를 로컬에서 어떻게 추적하고 동기화하는지 정확히 이해해야 한다.

### 리모트 트래킹 브랜치(`origin/master`)의 정체: 수정할 수 없는 로컬 북마크

`git ls-remote <remote>`나 `git remote show <remote>`로 원격 저장소의 참조 목록을 직접 확인할 수도 있지만, 일상 작업에서는 **리모트 트래킹 브랜치**(Remote-tracking Branch)를 활용한다.

- 리모트 트래킹 브랜치는 **`<remote>/<branch>`** 형식(예: `origin/master`)의 이름을 갖는다.
- 핵심은 이 포인터가 **개발자가 직접 체크아웃해 커밋을 쌓을 수 없는 읽기 전용 로컬 북마크**라는 점이다. 오직 `git fetch`나 `git push`처럼 원격 저장소와 **네트워크 통신이 발생할 때만 Git이 자동으로 포인터를 갱신**해 "마지막으로 서버와 접속했을 때 원격 브랜치가 어디에 있었는지"를 기억한다.

> **`origin` 역시 특별하지 않다**: `git init`이 기본 로컬 브랜치 이름을 `master`로 지어 주듯, `git clone`은 원격 저장소 단축 이름을 기본값으로 `origin`이라 붙일 뿐이다. 만약 `git clone -o booyah <url>`처럼 `-o` 옵션을 주면 리모트 트래킹 브랜치 이름도 `booyah/master`가 된다.

위 다이어그램 하단처럼 `clone` 이후 내가 로컬 `master`에서 새 커밋 `C5`를 만들고, 동시에 동료가 원격 저장소에 커밋 `C4`를 푸시했다고 가정해 보자. 네트워크 통신 전까지 내 로컬 `.git` 안의 `origin/master` 포인터는 여전히 `C3`에 멈춰 있다. 이때 `git fetch origin`을 실행하면 원격에서 부족한 데이터(`C4`)만 내려받아 로컬 객체 DB에 저장하고 `origin/master` 포인터를 `C4`로 옮긴다. Working Directory의 파일이나 내 로컬 `master` 브랜치는 전혀 건드리지 않는다.

참고로 `git remote add teamone <url>`처럼 여러 원격을 등록하고 `git fetch teamone`을 실행하면, `teamone/master`처럼 각 서버별로 독립된 리모트 트래킹 브랜치가 생긴다.

### 브랜치 푸시(Refspec 확장)와 새 원격 브랜치 가져오기

로컬에 만든 `serverfix` 같은 토픽 브랜치는 원격으로 자동 동기화되지 않는다. 공유하고 싶은 브랜치만 명시적으로 Push해야 한다.

```bash
$ git push origin serverfix
...
To https://github.com/schacon/simplegit
 * [new branch]      serverfix -> serverfix
```

이 짧은 명령 뒤에서 Git은 브랜치 이름 `serverfix`를 `refs/heads/serverfix:refs/heads/serverfix`라는 **Refspec**으로 자동 확장해 "내 로컬의 `serverfix` 브랜치를 원격의 `serverfix` 브랜치로 보낸다"는 의미로 해석한다. 만약 로컬과 다른 이름의 원격 브랜치로 올리고 싶다면 `git push origin serverfix:awesomebranch`처럼 콜론(`:`)으로 대상을 명시하면 된다. (HTTPS 프로토콜 푸시 시 매번 비밀번호를 입력하지 않으려면 `git config --global credential.helper cache`로 메모리 캐시를 활용할 수 있다.)

반대로 다른 동료가 올린 브랜치를 내가 `git fetch origin`으로 가져오면 어떻게 될까?

```bash
$ git fetch origin
...
From https://github.com/schacon/simplegit
 * [new branch]      serverfix    -> origin/serverfix
```

중요한 점은 `fetch` 직후 내 로컬에는 편집 가능한 `serverfix` 브랜치가 바로 생기지 않고, **수정할 수 없는 `origin/serverfix` 포인터만 생성된다**는 것이다. 따라서 이를 내 현재 브랜치에 바로 합치려면 `git merge origin/serverfix`를 실행하거나, 직접 작업할 수 있는 로컬 브랜치를 새로 파고 싶다면 아래와 같이 `origin/serverfix`를 기점으로 새 브랜치를 체크아웃해야 한다.

```bash
$ git checkout -b serverfix origin/serverfix
Branch serverfix set up to track remote branch serverfix from origin.
Switched to a new branch 'serverfix'
```

### 트래킹 브랜치(Tracking Branch)와 Upstream 관리: `--track`, `-u`, `@{u}`, `-vv`

`origin/serverfix`처럼 리모트 트래킹 브랜치를 기반으로 로컬 브랜치를 체크아웃하면 **트래킹 브랜치**(Tracking Branch)가 만들어지며, 연결된 원격 쪽 브랜치를 **업스트림 브랜치**(Upstream Branch)라고 부른다. 트래킹 관계가 맺어지면 해당 브랜치에서 인자 없이 `git pull`이나 `git push`만 입력해도 어느 서버의 어느 브랜치와 동기화해야 할지 Git이 자동으로 찾아낸다.

트래킹 브랜치를 만들고 상태를 확인하는 실전 단축 명령어는 다음과 같다.

```bash
# 1. --track 옵션으로 로컬과 동일한 이름의 트래킹 브랜치 생성
$ git checkout --track origin/serverfix

# 2. DWIM 단축 문법: 로컬에 없는 브랜치명이 단 하나의 원격에만 존재하면 자동 트래킹 생성
# (Git 2.23+ git switch serverfix 도 동일하게 동작)
$ git checkout serverfix

# 3. 로컬 브랜치 이름을 다르게(예: sf) 지정해 트래킹 시작
$ git checkout -b sf origin/serverfix

# 4. 이미 존재하는 로컬 브랜치의 업스트림을 지금 연결하거나 변경 (-u / --set-upstream-to)
$ git branch -u origin/serverfix
```

업스트림이 연결된 브랜치에서는 **`@{upstream}`** 또는 **`@{u}`** 단축 표기를 쓸 수 있다. 예를 들어 `master`가 `origin/master`를 추적 중이라면 `git merge origin/master` 대신 `git merge @{u}`처럼 간결하게 입력 가능하다.

로컬 브랜치들이 각각 어떤 업스트림을 추적 중이고, 원격보다 몇 커밋 앞서(`ahead`) 또는 뒤처져(`behind`) 있는지 한눈에 점검하려면 `git branch -vv`를 사용한다.

```bash
$ git branch -vv
  iss53     7e424c3 [origin/iss53: ahead 2] Add forgotten brackets
  master    1ae2a45 [origin/master] Deploy index fix
* serverfix f8674d9 [teamone/server-fix-good: ahead 3, behind 1] This should do it
  testing   5ea463a Try something new
```

이 출력에서 중요한 사실이 하나 더 있다. **`git branch -vv`는 원격 서버와 실시간 통신하지 않고, 마지막 `fetch` 시점까지 로컬 `.git`에 캐시된 리모트 트래킹 브랜치와 비교할 뿐**이다. 따라서 가장 정확한 최신 `ahead`/`behind` 수치를 얻으려면 반드시 모든 원격에서 먼저 Fetch한 뒤 조회해야 한다.

```bash
$ git fetch --all; git branch -vv
```

### `git pull`의 내부 동작과 원격 브랜치 삭제(`--delete`)

이제 `git pull`이 어떻게 동작하는지 한 줄로 명확하게 설명할 수 있다. 트래킹 브랜치 위에서 `git pull`을 실행하면, Git은 현재 브랜치에 설정된 업스트림 리모트와 브랜치를 조회한 뒤 **그대로 `git fetch`를 실행하고 즉시 `git merge`(또는 설정에 따라 `rebase`)를 이어서 수행**한다. 원격의 상태를 먼저 안전하게 확인하고 싶다면 `git fetch` 후 `git branch -vv`나 `git log`로 검토한 다음 수동으로 `merge`하는 습관이 훨씬 안전하다.

마지막으로 토픽 브랜치의 머지가 끝나 서버에서 원격 브랜치를 삭제하려면 `git push`에 `--delete` 옵션을 준다.

```bash
$ git push origin --delete serverfix
To https://github.com/schacon/simplegit
 - [deleted]         serverfix
```

이 명령은 서버의 브랜치 포인터를 지울 뿐이며, 실제 커밋 객체 데이터는 서버의 가비지 컬렉션(GC)이 돌기 전까지 일정 기간 남아 있으므로 실수로 삭제했더라도 복구할 여지가 있다.

---

## 마치며

이번 편에서는 **Pro Git 3장 중반부**(3.3~3.5)를 통해 브랜치가 수십 개로 늘어나도 흔들리지 않고 제어할 수 있는 실전 관리법과 협업의 뼈대를 살펴보았다.

- `git branch --merged`와 `--no-merged`로 병합 완료 여부를 정확히 가려내 안전하게 브랜치를 정리할 수 있다.
- 안정성 단계에 따라 포인터를 진급시키는 **Long-Running 브랜치**와 격리된 실험·리뷰 후 즉시 합치거나 폐기하는 **Topic 브랜치**는 모두 100% 로컬에서 동작한다.
- **`origin/master`** 같은 리모트 트래킹 브랜치는 서버와 통신할 때만 갱신되는 로컬 북마크이며, 로컬 브랜치와 연결(`-u`)하면 `@{u}`나 `git branch -vv`로 업스트림과의 편차를 손쉽게 추적할 수 있다.

다음 마지막 편(`[Pro Git 톺아보기 #6]`)에서는 Git에서 가장 강력하면서도 오용 시 큰 사고로 이어지기 쉬운 **3.6 Rebasing**의 내부 동작(`--onto` 포함)과 공개 저장소에서의 Golden Rule, 그리고 Merge vs Rebase 철학까지 완전히 해부하며 3장을 마무리한다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.3 Git Branching - Branch Management](https://git-scm.com/book/en/v2/Git-Branching-Branch-Management)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.4 Git Branching - Branching Workflows](https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.5 Git Branching - Remote Branches](https://git-scm.com/book/en/v2/Git-Branching-Remote-Branches)
