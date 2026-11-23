+++
date = '2026-11-22T13:10:00+09:00'
draft = false
title = 'Git 내부 원리 #3: fetch vs pull과 태그 내부 구조'
description = 'Pro Git 2장 후반부를 바탕으로 원격 저장소 동기화, git fetch와 pull의 차이, Annotated·Lightweight 태그 객체 구조와 Git Alias를 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글]({{< ref "/posts/pro-git-internals-ch2-part1" >}})에서는 로컬 머신에서 Git 저장소를 만들고, Working Tree와 Staging Area(Index), `.git` 디렉토리 사이에서 파일 상태가 전이되는 원리를 살펴보았다.

이번 글에서는 **Pro Git 2장**(Git Basics)의 후반부(2.5~2.7)를 다룬다. 다른 개발자와 협업하기 위해 원격 저장소(Remote)를 등록하고 동기화하는 내부 메커니즘, 특정 시점의 커밋을 영구히 박제하는 두 가지 태그(Tag) 객체의 차이, 그리고 CLI 작업 효율을 극대화해 주는 **Git Alias**(특히 가장 유명한 `git lg` 설정)까지 하나씩 파헤쳐 본다.

---

## 1. 원격 저장소 다루기: 리모트 핸들과 동기화 원리 (Pro Git 2.5)

원격 저장소(Remote Repository)는 인터넷이나 네트워크 어딘가에 호스팅된 프로젝트의 또 다른 버전이다. 흥미로운 점은 "원격(Remote)"이라는 단어가 반드시 네트워크 너머의 서버를 뜻하지는 않는다는 것이다. **같은 로컬 머신의 다른 경로에 있는 저장소**라도 별도의 저장소라면 똑같이 `fetch`, `pull`, `push`가 동작하는 원격 저장소로 등록할 수 있다.

### 원격 저장소 조회 및 추가: `git remote`

저장소를 `git clone`으로 복제하면 Git은 자동으로 복제해 온 서버를 `origin`이라는 기본 이름(Shortname)으로 등록한다. `git remote -v` 옵션을 주면 각 리모트가 읽기(Fetch)와 쓰기(Push)에 사용하는 실제 URL까지 확인할 수 있다.

```bash
$ git clone https://github.com/schacon/ticgit
$ cd ticgit
$ git remote -v
origin	https://github.com/schacon/ticgit (fetch)
origin	https://github.com/schacon/ticgit (push)
```

오픈소스 협업 환경처럼 여러 기여자의 저장소를 동시에 연결해야 한다면 `git remote add <shortname> <url>`로 원하는 만큼 리모트를 추가할 수 있다. `grit` 프로젝트처럼 리모트마다 `https://`, `git://`, `git@github.com:...` 등 서로 다른 프로토콜이 섞여 있어도 무방하다.

```bash
$ git remote add pb https://github.com/paulboone/ticgit
$ git remote -v
origin	https://github.com/schacon/ticgit (fetch)
origin	https://github.com/schacon/ticgit (push)
pb	https://github.com/paulboone/ticgit (fetch)
pb	https://github.com/paulboone/ticgit (push)
```

이제 긴 URL 대신 `pb`라는 짧은 핸들로 Paul의 저장소에 접근할 수 있다. `git fetch pb`를 실행하면 내 로컬 저장소에 없던 데이터만 내려받고, Paul의 `master` 브랜치는 로컬에서 `pb/master`라는 **리모트 트래킹 브랜치**(Remote-tracking branch)로 접근 가능해진다.

```bash
$ git fetch pb
remote: Counting objects: 43, done.
...
From https://github.com/paulboone/ticgit
 * [new branch]      master     -> pb/master
 * [new branch]      ticgit     -> pb/ticgit
```

### `git fetch` vs `git pull`, 그리고 `git push`

현업에서 원격 데이터를 가져올 때 `fetch`와 `pull`을 혼동하는 경우가 많지만, 내부적으로 건드리는 영역에는 분명한 경계가 있다.

{{< figure src="remote-and-tags.svg" alt="Git 원격 저장소 동기화(fetch, pull, push)와 태그(Annotated, Lightweight) 내부 객체 구조 다이어그램" caption="원격 저장소와 로컬 .git/Working Tree 동기화 범위 및 Annotated/Lightweight 태그의 객체 구조 (Pro Git 2.5~2.6 재구성)" width="820" height="480" >}}

- **`git fetch <remote>`**: 원격 저장소에 새로 푸시된 객체 데이터와 브랜치 참조(예: `origin/master`)만 **로컬 `.git` 데이터베이스로 내려받을 뿐, 현재 내가 작업 중인 로컬 브랜치나 Working Tree의 코드는 전혀 건드리지 않는다.** 변경 사항을 먼저 검토한 뒤 원하는 시점에 직접 병합(Merge)하고자 할 때 안전하다.
- **`git pull`**: 현재 로컬 브랜치가 원격 브랜치를 추적(Track)하도록 설정되어 있을 때, 원격 데이터를 **Fetch한 직후 현재 브랜치에 자동으로 Merge(또는 Rebase)까지 수행**하여 Working Tree를 갱신한다. (`git clone`은 기본적으로 로컬 `master` 또는 `main` 브랜치가 `origin`의 기본 브랜치를 추적하도록 설정한다.)

> **Git 2.27+ `pull.rebase` 경고**: Git 2.27 이후 버전부터는 `pull.rebase` 설정이 비어 있으면 `git pull` 실행 시 기본 병합 전략을 명시하라는 안내 문구가 출력된다. 기존처럼 Fast-forward가 가능할 때 이동하고 아니면 Merge 커밋을 만들려면 `git config --global pull.rebase "false"`, Pull 시 Rebase를 기본 동작으로 삼으려면 `"true"`로 지정하면 된다.

내 작업을 원격에 공유할 때는 `git push <remote> <branch>`(예: `git push origin master`)를 사용한다. 이 명령은 해당 원격 저장소에 **쓰기 권한이 있고, 내가 Clone하거나 마지막으로 Fetch한 이후 다른 사람이 먼저 Push한 커밋이 없을 때만** 성공한다. 만약 다른 동료가 먼저 푸시했다면 내 Push는 거절(Reject)되며, 반드시 먼저 상대의 변경분을 Fetch해 내 작업과 통합(Merge 또는 Rebase)한 뒤에야 다시 Push할 수 있다.

### 리모트 상세 점검, 이름 변경, 삭제

`git remote show <remote>`를 실행하면 원격 저장소의 Fetch/Push URL, `HEAD` 브랜치, 로컬에 아직 가져오지 않은 새 원격 브랜치(`new`), 원격에서는 이미 지워졌지만 로컬에 잔존하는 브랜치(`stale`, `git remote prune`으로 정리 대상), 그리고 `git pull`과 `git push` 실행 시 어느 브랜치와 매핑되는지까지 한눈에 파악할 수 있다.

```bash
$ git remote show origin
* remote origin
  Fetch URL: https://github.com/my-org/complex-project
  Push  URL: https://github.com/my-org/complex-project
  HEAD branch: master
  Remote branches:
    master                           tracked
    dev-branch                       tracked
    issue-43                         new (next fetch will store in remotes/origin)
    refs/remotes/origin/issue-11     stale (use 'git remote prune' to remove)
  Local branches configured for 'git pull':
    dev-branch merges with remote dev-branch
    master     merges with remote master
  Local refs configured for 'git push':
    dev-branch pushes to dev-branch (up to date)
    master     pushes to master     (up to date)
```

리모트 이름을 바꾸고 싶다면 `git remote rename pb paul`을 실행한다. 이때 리모트 이름뿐만 아니라 연결된 리모트 트래킹 브랜치 이름도 `pb/master`에서 `paul/master`로 함께 바뀐다. 더 이상 쓰지 않는 리모트를 제거하려면 `git remote remove paul`(또는 `git remote rm paul`)을 실행하며, 이때 해당 리모트에 속한 트래킹 브랜치와 설정도 모두 삭제된다.

---

## 2. 태그(Tagging): Annotated vs Lightweight 내부 구조 (Pro Git 2.6)

Git은 히스토리 상에서 릴리즈 시점(`v1.0`, `v2.0`)처럼 중요한 지점을 표시하기 위해 **태그**(Tag) 기능을 제공한다.

### 태그 목록 조회와 와일드카드 필터링

아무 옵션 없이 `git tag`만 입력하면 태그 목록이 알파벳 순으로 출력된다. Git 자체 소스 저장소처럼 태그가 500개 이상 쌓여 특정 패턴만 조회하고 싶다면 **`-l`** 또는 **`--list`** 옵션을 반드시 함께 써야 한다.

```bash
# 전체 목록 조회 시에는 -l 생략 가능
$ git tag

# 와일드카드 패턴으로 필터링할 때는 -l (--list) 필수
$ git tag -l "v1.8.5*"
v1.8.5
v1.8.5-rc0
...
```

### Annotated 태그 vs Lightweight 태그

Git 태그는 내부 저장 방식에 따라 두 종류로 나뉜다.

1. **Annotated 태그 (`-a` 옵션)**
   - `.git` 데이터베이스에 **완전한 독립 객체**(Full Object)로 저장된다.
   - 자체 SHA-1 체크섬을 가지며, 태그를 생성한 사람(Tagger)의 이름과 이메일, 생성 날짜, 태그 메시지, 그리고 필요 시 GPG(GNU Privacy Guard) 전자서명까지 함께 포함한다. 릴리즈 태그 등 공식 이력에는 Annotated 태그가 권장된다.
2. **Lightweight 태그 (옵션 없음)**
   - 별도의 태그 객체를 만들지 않고, **특정 커밋의 체크섬 해시만 파일에 담아 가리키는 고정 포인터**다. 임시 표시나 로컬 작업용으로 가볍게 쓸 때 적합하다.

`git show`로 두 태그를 조회해 보면 내부 객체의 차이가 명확하게 드러난다.

```bash
# 1. Annotated 태그 생성 및 조회: Tagger 메타데이터 + 태그 메시지 + 커밋 정보 출력
$ git tag -a v1.4 -m "my version 1.4"
$ git show v1.4
tag v1.4
Tagger: Ben Straub <ben@straub.cc>
Date:   Sat May 3 20:19:12 2014 -0700

my version 1.4

commit ca82a6dff817ec66f44342007202690a93763949
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Mon Mar 17 21:52:11 2008 -0700

    Change version number

# 2. Lightweight 태그 생성 및 조회: 태그 메타데이터 없이 커밋 정보만 출력
$ git tag v1.4-lw
$ git show v1.4-lw
commit ca82a6dff817ec66f44342007202690a93763949
Author: Scott Chacon <schacon@gee-mail.com>
Date:   Mon Mar 17 21:52:11 2008 -0700

    Change version number
```

이미 지나간 과거 커밋에 나중에 태그를 달아야 한다면 `git log --pretty=oneline`으로 커밋 해시(또는 앞부분 축약 해시)를 확인한 뒤 명령어 끝에 지정하면 된다.

```bash
$ git tag -a v1.2 9fceb02
```

### 태그 공유와 삭제, 그리고 Detached HEAD 주의점

- **태그 공유 (`git push`)**: 일반적인 `git push`는 태그를 원격 서버로 전송하지 않는다. 따라서 브랜치처럼 명시적으로 태그를 푸시해야 한다.
  - 특정 태그 1개 전송: `git push origin v1.5`
  - 모든 로컬 태그(Annotated + Lightweight 모두) 전송: `git push origin --tags`
  - **Annotated 태그만 선별해** 커밋과 함께 전송: `git push origin --follow-tags` (임시 Lightweight 태그까지 원격에 올라가는 것을 방지할 때 유용하다.)
- **태그 삭제**:
  - 로컬 삭제: `git tag -d v1.4-lw` (원격 서버의 태그는 지워지지 않는다.)
  - 원격 삭제: 빈 값(null)을 원격 태그 참조에 푸시하는 `git push origin :refs/tags/v1.4-lw` 또는 직관적인 `git push origin --delete v1.4-lw` 명령을 사용한다.
- **태그 체크아웃과 Detached HEAD**:
  - `git checkout v2.0.0`처럼 태그를 직접 체크아웃하면 브랜치가 아니라 특정 커밋을 직접 가리키는 **Detached HEAD** 상태가 된다. 이 상태에서 새 커밋을 만들더라도 태그는 움직이지 않으며, 새 커밋은 어떤 브랜치에도 속하지 않아 다른 브랜치로 이동하는 순간 도달할 수 없는 고아 커밋이 된다. 따라서 과거 태그 시점에서 버그 수정이나 새 작업을 이어가려면 반드시 새 브랜치를 생성해 체크아웃해야 한다.

```bash
$ git checkout -b version2 v2.0.0
Switched to a new branch 'version2'
```

---

## 3. Git Aliases와 실전 `git lg` 완전 해부 (Pro Git 2.7)

Git은 명령어 일부만 입력했을 때 자동으로 명령을 추론해 주지 않는다. 대신 `git config`의 `alias` 설정을 통해 자주 쓰는 명령어를 단축하거나, 아예 Git에 없는 나만의 하위 명령어를 만들어 쓸 수 있다.

### 기본 단축 Alias와 외부 명령어 실행 (`!`)

Pro Git에서 소개하는 가장 기본적인 단축 Alias와 유틸리티 Alias는 다음과 같다.

```bash
# 자주 쓰는 기본 명령어 단축
$ git config --global alias.co checkout
$ git config --global alias.br branch
$ git config --global alias.ci commit
$ git config --global alias.st status

# 'git unstage fileA' -> 'git reset HEAD -- fileA'로 치환하는 커스텀 명령
$ git config --global alias.unstage 'reset HEAD --'

# 마지막 커밋 1개만 빠르게 확인하는 'git last'
$ git config --global alias.last 'log -1 HEAD'
```

Git 하위 명령어가 아니라 **외부 셸 명령어나 GUI 도구를 실행하고 싶다면 명령어 맨 앞에 `!` 기호**를 붙인다. 예를 들어 `$ git config --global alias.visual '!gitk'`를 등록하면 `git visual`로 `gitk`를 실행할 수 있다.

### 전설적인 한 줄 로그 Alias 해부: `johanmeiring/3002458`

Git 커뮤니티에서 오랫동안 가장 유명하고 사랑받아 온 Alias 설정은 [johanmeiring/3002458 Gist](https://gist.github.com/johanmeiring/3002458)로 널리 공유된 `git lg`다. 기본 `git log`가 화면을 너무 많이 차지해 브랜치 흐름을 한눈에 보기 어렵다는 단점을 완벽하게 해결한다.

```bash
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --"
```

이 긴 한 줄이 내부적으로 어떤 포맷 지정자를 조합해 동작하는지 분해해 보면 다음과 같다.

| 구성 요소 | 역할 및 출력 의미 |
| :--- | :--- |
| `--color` | 파이프나 환경에 관계없이 ANSI 컬러 출력을 활성화한다. |
| `--graph` | 좌측에 브랜치 분기와 머지 흐름을 ASCII 그래프로 그린다. |
| `%Cred%h%Creset` | **축약 커밋 해시**(`%h`)를 빨간색(`%Cred`)으로 표시하고 색상을 초기화(`%Creset`)한다. |
| `%C(yellow)%d%Creset` | 해당 커밋을 가리키는 **브랜치·태그 포인터**(`HEAD -> main`, `origin/main`, `tag: v1.4` 등)를 노란색으로 표시한다. |
| `%s` | 커밋 제목(Subject)을 출력한다. |
| `%Cgreen(%cr)%Creset` | **커미터 기준 상대 시간**(`2 hours ago` 등)을 초록색으로 표시한다. |
| `%C(bold blue)<%an>%Creset` | **Author 이름**을 굵은 파란색으로 표시한다. |
| `--abbrev-commit` | 40자 전체 SHA-1 대신 식별 가능한 짧은 해시(기본 7자 내외)만 출력한다. |
| `--` | 마지막의 이중 대시는 Alias 뒤에 파일 경로나 추가 인자를 넘길 때 옵션과 경로를 안전하게 분리해 준다. |

실제 저장소에서 `git lg`를 실행하면 아래와 같이 브랜치 분기, 태그, 최신 커밋, 작성자, 작성 시점이 한 줄씩 압축되어 한눈에 들어온다.

```text
* 4c3c1ae - (HEAD -> main, origin/main) fix(post): remove #center fragment (20 hours ago) <Wooyoung Choi>
* 576d72e - Merge pull request #6 from withgoru/docs/post-pro-git-ch1 (4 days ago) <Wooyoung Choi>
|\
| * 2e44c76 - (tag: v1.0) docs(post): add Pro Git Chapter 1 internals deep-dive post (4 days ago) <Wooyoung Choi>
|/
* 41860fc - Merge pull request #5 from withgoru/docs/post-no-ai-slop-manifesto (4 days ago) <Wooyoung Choi>
```

---

## 마치며

이로써 **Pro Git 2장**(Git Basics)의 전체 여정을 마무리했다. 원격 저장소와의 동기화는 단순히 코드를 주고받는 것이 아니라 로컬 `.git` 안의 **리모트 트래킹 브랜치(`origin/master`)와 로컬 작업 브랜치(`master`)를 분리해 관리**하는 구조에서 출발하며, 태그 역시 **독립 객체**(Annotated)와 **단순 포인터**(Lightweight)라는 Git의 객체 철학을 그대로 따른다.

다음 편에서는 Git을 여타 버전 관리 도구와 완전히 갈라놓는 핵심 무기이자 Pro Git에서 가장 빛나는 챕터인 **3장**(Git Branching)을 통해 브랜치와 HEAD 포인터가 실제로 `.git` 내부에서 어떻게 움직이는지 파헤쳐 본다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.5 Git Basics - Working with Remotes](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.6 Git Basics - Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.7 Git Basics - Git Aliases](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases)
- [Johan Meiring - Pretty git log alias (`git lg`) Gist](https://gist.github.com/johanmeiring/3002458)
