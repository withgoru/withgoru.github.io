+++
date = '2026-11-25T09:00:00+09:00'
draft = false
title = '[Pro Git 실습편] 설치부터 브랜치·리베이스·원격 동기화까지: 터미널과 Vim으로 직접 확인하는 Git 내부 원리 (Ch 1~3 총정리)'
description = 'Pro Git 1~3장 시리즈(#1~#6)에서 다룬 스냅샷, 3가지 영역, 파일 생명주기, 41바이트 브랜치 포인터, Fast-forward 및 3-Way Merge 충돌 해결, Rebase, 리모트 트래킹 브랜치와 태그까지 터미널과 Vim 실습 코드로 직접 따라 하며 눈으로 확인한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'HandsOn', 'Vim', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

지금까지 **Pro Git 1~3장**에 걸쳐 Git의 내부 철학과 동작 원리를 여섯 편의 글로 나누어 정리했다. ([1장]({{< ref "/posts/pro-git-internals-ch1" >}}), [2장 전반]({{< ref "/posts/pro-git-internals-ch2-part1" >}}), [2장 후반]({{< ref "/posts/pro-git-internals-ch2-part2" >}}), [3장 1부]({{< ref "/posts/pro-git-internals-ch3-part1" >}}), [3장 2부]({{< ref "/posts/pro-git-internals-ch3-part2" >}}), [3장 3부]({{< ref "/posts/pro-git-internals-ch3-part3" >}}))

개념을 읽고 고개를 끄덕이는 것과, 실제 터미널을 열어 파일 상태가 전이되고 `.git` 디렉토리 안의 41바이트 포인터가 움직이는 장면을 직접 눈으로 보는 것 사이에는 큰 차이가 있다. 이번 글에서는 **Git 설치 → 쉘·Vim 생존 연습 → 환경설정 → 저장소 생성 → 파일 편집 및 상태 전이 → 브랜치·머지·충돌 해결·리베이스 → 원격 저장소 동기화 및 태그 릴리즈**까지 1장부터 3장까지의 모든 핵심 원리를 처음부터 끝까지 하나의 흐름으로 직접 실습해 본다.

> **실습 진행 팁**: 본문의 모든 명령어는 macOS 터미널과 Windows의 **Git Bash**에서 순서대로 그대로 따라 치면 끝까지 막힘없이 완료되도록 설계했다. 원격 저장소 실습 역시 별도의 GitHub 계정 설정 없이 로컬 머신에서 100% 재현할 수 있도록 로컬 Bare 저장소(`--bare`)를 함께 활용한다.

---

## 1. 실습 환경 구축: Git 설치 (Windows & macOS)

먼저 자신의 운영체제에 맞는 방법으로 Git을 설치하고 CLI 실행 환경을 준비한다.

### Windows: Git for Windows 및 Git Bash

Windows에는 기본적으로 POSIX 호환 유닉스 쉘이 없으므로, 공식 패키지인 **Git for Windows**를 설치하는 것이 표준이다.

1. [Git for Windows 공식 사이트(gitforwindows.org)](https://gitforwindows.org/)에서 설치 파일을 내려받아 실행한다.
2. 설치 옵션 중 **Git Bash**가 기본으로 포함되며, 기본 에디터 선택 단계에서 **Vim**을 선택할 수 있다(나중에 `git config`로도 변경 가능).
3. 설치가 끝나면 시작 메뉴에서 **Git Bash**를 실행한다. 이후 모든 실습은 Windows 기본 `cmd`가 아니라 **Git Bash** 창에서 진행한다.

### macOS: Xcode Command Line Tools 또는 Homebrew

macOS에서는 두 가지 대표적인 방법으로 Git을 설치할 수 있다.

```bash
# 방법 1: Apple 공식 Xcode Command Line Tools 설치 (git 포함)
$ xcode-select --install

# 방법 2: Homebrew가 설치되어 있다면 최신 버전의 git 직접 설치 (권장)
$ brew install git
```

설치가 완료되었다면 버전 명령어로 정상 동작을 확인한다. (본 실습의 `git switch` 및 `git restore` 명령어는 **Git 2.23 이상**을 기준으로 한다.)

```bash
$ git --version
git version 2.47.0
```

---

## 2. Git 실습을 위한 필수 쉘(Shell) & Vim 생존 커맨드

GUI 도구를 쓰더라도, Git의 핵심 기능(대화형 리베이스, 커밋 메시지 편집, 머지 충돌 해결, `.git` 내부 파일 확인)을 제대로 다루려면 기본적인 **터미널 쉘 명령어**와 **Vim 편집기 조작법**을 반드시 손에 익혀야 한다.

### 실습에 쓰이는 최소 쉘(Shell) 커맨드 연습

먼저 홈 디렉토리 아래에 이번 실습 전용 공간(`~/git-lab`)을 만들고 기본적인 파일시스템 탐색 명령어를 연습한다.

```bash
# 1. 현재 작업 디렉토리의 절대 경로 확인 (Print Working Directory)
$ pwd

# 2. 실습용 부모 디렉토리 생성 및 이동
$ mkdir -p ~/git-lab
$ cd ~/git-lab

# 3. 숨김 파일(.으로 시작)을 포함한 디렉토리 목록 상세 조회
$ ls -la

# 4. 파일에 텍스트 한 줄 쓰고 내용 확인하기
$ echo "shell test" > hello.txt
$ cat hello.txt
shell test

# 5. 연습용 파일 삭제
$ rm hello.txt
```

### Vim 3분 생존 연습: 3가지 모드만 기억하자

Vim을 처음 켜고 당황해서 터미널 창을 강제로 닫는 가장 큰 이유는 Vim이 **모드(Mode) 기반 편집기**라는 사실을 모르기 때문이다. 딱 세 가지 모드 전환만 익히면 Git 작업에는 전혀 무리가 없다.

{{< figure src="vim-and-git-config.svg" alt="Vim 3가지 모드 상태 전이와 git config 3계층 우선순위 다이어그램" caption="Git 커밋·충돌·리베이스 시 사용하는 Vim의 3가지 모드 전환과 git config 3계층 우선순위 (Pro Git 1.6, 2.7 재구성)" width="820" height="440" >}}

1. **Normal Mode (기본 모드)**: `vim 파일명`으로 열었을 때의 시작 상태다. 여기서는 글자가 써지지 않고 `h`(좌), `j`(하), `k`(상), `l`(우) 이동, **`dd`**(현재 줄 통째로 삭제), **`u`**(방금 작업 되돌리기) 같은 명령이 동작한다.
2. **Insert Mode (입력 모드)**: Normal 모드에서 **`i`**(현재 커서 앞), **`a`**(현재 커서 뒤), **`o`**(아래 새 줄 열기) 키를 누르면 화면 하단에 `-- INSERT --`가 뜨며 일반 메모장처럼 자유롭게 타이핑할 수 있다. 입력이 끝나면 반드시 **`ESC`** 키를 눌러 Normal 모드로 돌아온다.
3. **Command-Line Mode (명령 모드)**: Normal 모드에서 **콜론**(`:`)을 누르고 명령을 입력한 뒤 `Enter`를 친다.
   - `:w` → 저장(Write)
   - `:q` → 종료(Quit)
   - `:wq` → **저장 후 종료** (커밋 메시지 작성 뒤 가장 많이 쓴다)
   - `:q!` → **저장하지 않고 강제 종료** (수정을 잘못했을 때 탈출용)
   - `/검색어` + `Enter` → 문서 내 텍스트 검색 (`n`으로 다음 결과 이동)

지금 바로 `~/git-lab`에서 연습 파일을 열어 입력·저장·종료까지 한 번 손으로 익혀 보자.

```bash
$ vim vim-practice.txt
```

1. `i`를 눌러 입력 모드로 전환한 뒤 아무 문장이나 두 줄 입력한다.
2. `ESC`를 눌러 Normal 모드로 돌아온 뒤, 두 번째 줄에 커서를 두고 `dd`를 눌러 한 줄을 지워본다.
3. `u`를 눌러 방금 지운 줄을 되살린다.
4. `:wq`를 입력하고 `Enter`를 쳐서 저장 후 나온다.

저장 결과를 확인하고 연습용 파일을 정리한다.

```bash
$ cat vim-practice.txt
$ rm vim-practice.txt
```

---

## 3. 최초 환경설정: `git config` 3계층과 전설의 `git lg` Alias

Git 설정 파일은 적용 범위에 따라 **System(`/etc/gitconfig`) → Global(`~/.gitconfig`) → Local(`.git/config`)** 세 계층으로 나뉘며, 좁은 범위의 설정이 넓은 범위를 덮어쓴다. 실습에 앞서 전역(`--global`) 설정으로 사용자 서명, 기본 편집기, 기본 브랜치 이름, 그리고 히스토리 확인용 `git lg` 별칭을 등록한다.

```bash
# 1. 커밋 Author/Committer에 기록될 이름과 이메일 (필수)
$ git config --global user.name "Goru Dev"
$ git config --global user.email "goru@example.com"

# 2. 커밋 메시지·리베이스·머지 편집 시 Vim을 기본 편집기로 고정
$ git config --global core.editor "vim"

# 3. git init 실행 시 생성되는 기본 브랜치 이름을 main으로 지정
$ git config --global init.defaultBranch main

# 4. Pro Git 2.7에서 소개한 실전 한 줄 로그 Alias (git lg)
$ git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --"

# 5. 현재 적용된 모든 설정과 각 설정이 어느 파일에서 왔는지 출처 확인
$ git config --list --show-origin
file:/Users/<username>/.gitconfig	user.name=Goru Dev
file:/Users/<username>/.gitconfig	user.email=goru@example.com
file:/Users/<username>/.gitconfig	core.editor=vim
file:/Users/<username>/.gitconfig	init.defaultBranch=main
file:/Users/<username>/.gitconfig	alias.lg=log --color --graph ...
```

---

## 4. 로컬 저장소 생성과 `.git` 디렉토리 해부

이제 실제 실습을 진행할 프로젝트 디렉토리 `git-practice`를 만들고, `git init` 실행 직후 내부에 무엇이 생기는지 직접 열어 본다.

```bash
$ mkdir -p ~/git-lab/git-practice
$ cd ~/git-lab/git-practice
$ git init
Initialized empty Git repository in /Users/<username>/git-lab/git-practice/.git/
```

아직 파일을 하나도 만들지 않았지만, `ls -la .git`으로 숨겨진 `.git` 디렉토리를 들여다보면 Git의 뼈대가 이미 완성되어 있다.

```bash
$ ls -la .git
HEAD
config
description
hooks/
info/
objects/
refs/

$ cat .git/HEAD
ref: refs/heads/main
```

여기서 1장에서 배운 개념이 즉시 현실과 연결된다. 현재 `git-practice` 디렉토리 자체가 **Working Tree**이고, 곧 파일이 스테이징되면 생길 `.git/index`가 **Staging Area**이며, 지금 눈앞에 보이는 `.git` 디렉토리가 모든 스냅샷과 포인터를 보관하는 **Git Directory**다. `.git/HEAD` 파일 안에는 지금 내가 서 있는 브랜치를 뜻하는 `ref: refs/heads/main` 한 줄이 적혀 있다(아직 첫 커밋이 없으므로 `refs/heads/main` 파일 자체는 아직 존재하지 않는다).

---

## 5. Vim으로 파일 작업과 3가지 영역 상태 전이 실습

이제 Working Tree, Staging Area(Index), `.git` 디렉토리 사이에서 파일이 어떤 상태를 오가는지 순서대로 직접 실험해 본다.

{{< figure src="three-areas-and-file-states.svg" alt="Git 3대 영역과 파일 상태 전이 생명주기 및 git status -s 코드 다이어그램" caption="Working Tree · Staging Area(Index) · .git Directory 간 명령어별 데이터 이동과 MM(Partially Staged) 상태 전이 (Pro Git 1.3, 2.2~2.4 재구성)" width="820" height="540" >}}

### ① Untracked에서 Staged, 그리고 첫 Commit까지

먼저 Vim으로 `README.md`와 `calc.py` 두 파일을 생성한다. 터미널에서 `vim README.md`를 입력해 아래 내용을 적고 `ESC` → `:wq`로 저장한다.

```bash
$ vim README.md
```

```markdown
# Git Practice Project
Learning Git internals hands-on.
```

이어서 `vim calc.py`를 입력해 첫 번째 함수를 작성하고 `ESC` → `:wq`로 저장한다.

```bash
$ vim calc.py
```

```python
def add(a, b):
    return a + b
```

이제 `git status -s`로 상태를 확인한 뒤 `git add`로 Staging Area에 올리고, **인자 없이** `git commit`을 실행해 Vim으로 첫 커밋을 남겨 보자.

```bash
$ git status -s
?? README.md
?? calc.py

$ git add README.md calc.py
$ ls -la .git/index
$ git status -s
A  README.md
A  calc.py

# 인자 없이 git commit을 실행하면 Vim(.git/COMMIT_EDITMSG)이 열린다!
$ git commit
```

Vim이 열리면 첫 줄에 `i`를 눌러 `feat: add initial README and calc.py`라고 입력하고 `ESC` → `:wq`를 눌러 커밋을 확정한다. (원한다면 `git commit -m "..."` 대신 이렇게 Vim에서 커밋 메시지를 작성하는 흐름에 익숙해지자.)

```bash
$ git lg
* 8a1b2c3 - (HEAD -> main) feat: add initial README and calc.py (5 seconds ago) <Goru Dev>

$ find .git/objects -type f
```

`find .git/objects -type f` 출력에는 Blob 2개(`README.md`, `calc.py`), Tree 1개, Commit 1개, 총 **4개의 객체 파일**이 `.git/objects` 안에 해시 디렉토리로 저장되어 있다.

### ② `MM` 상태 만들기와 `git diff` vs `git diff --staged`

이제 두 파일이 **Unmodified**(`Work = Index = HEAD`) 상태가 되었다. 여기서 `calc.py`를 수정해 한 번 스테이징(`v1`)한 뒤, 커밋하기 전에 한 번 더 수정(`v2`)하여 전설적인 **`MM` 상태**를 직접 만들어 보자.

```bash
# 1. vim calc.py로 sub 함수를 추가하고 저장(:wq)한 뒤 git add 한다
$ vim calc.py
```

```python
def add(a, b):
    return a + b

def sub(a, b):
    return a - b
```

```bash
$ git add calc.py
$ git status -s
M  calc.py
```

왼쪽 첫 번째 열에 `M`이 찍혔다(`Staged Modified`). 이제 커밋을 누르기 전에 `vim calc.py`를 **한 번 더 열어** `mul` 함수를 추가하고 저장해 보자.

```bash
$ vim calc.py
```

```python
def add(a, b):
    return a + b

def sub(a, b):
    return a - b

def mul(a, b):
    return a * b
```

```bash
$ git status -s
MM calc.py
```

정확히 `MM calc.py`가 출력되었다! 이 상태에서 바로 `git commit`을 하면 `Working Tree`의 `mul` 함수는 빠지고 `Index`에 들어 있는 `sub` 함수까지만 커밋된다. 아래 두 `diff` 명령어로 각 경계의 차이를 직접 눈으로 확인하자.

```bash
# 1. Working Tree vs Index: 아직 스테이징하지 않은 mul 함수만 출력된다
$ git diff
+def mul(a, b):
+    return a * b

# 2. Index vs HEAD: 직전 커밋 이후 스테이징된 sub 함수만 출력된다
$ git diff --staged
+def sub(a, b):
+    return a - b
```

이제 최신 수정분(`mul`)까지 마저 스테이징하고, `git diff`가 비어 있음을 확인한 뒤 두 번째 커밋을 생성한다.

```bash
$ git add calc.py
$ git diff
$ git commit -m "feat: add sub and mul functions"
```

### ③ 되돌리기(`--amend`, `restore`)와 추적 제외(`.gitignore`, `rm --cached`)

방금 만든 두 번째 커밋에 나눗셈 함수(`div`)를 깜빡하고 빠뜨렸다고 가정하고, **직전 커밋을 수정**(`--amend`)해 보자.

```bash
$ vim calc.py
```

```python
def add(a, b):
    return a + b

def sub(a, b):
    return a - b

def mul(a, b):
    return a * b

def div(a, b):
    return a / b
```

```bash
$ git add calc.py
$ git commit --amend
# Vim이 열리면 제목을 "feat: add sub, mul, and div functions"로 수정한 뒤 :wq

$ git lg
* 4f9d0e1 - (HEAD -> main) feat: add sub, mul, and div functions (3 seconds ago) <Goru Dev>
* 8a1b2c3 - feat: add initial README and calc.py (2 minutes ago) <Goru Dev>
```

직전 커밋의 해시가 완전히 새로운 SHA-1 값으로 교체되었다. `--amend`는 제자리 수정이 아니라 새 커밋 객체로 직전 커밋을 대체한다.

다음으로 **스테이징 취소**(`--staged`)와 **작업 디렉토리 변경 폐기**를 실습한다.

```bash
# 실수로 README.md를 수정하고 스테이징까지 했다고 가정하자
$ echo "Mistake line" >> README.md
$ git add README.md
$ git status -s
M  README.md

# 1. Index에서만 내리기 (Working Tree의 "Mistake line"은 살아 있다)
$ git restore --staged README.md
$ git status -s
 M README.md

# 2. Working Tree의 수정 사항까지 완전히 버리고 직전 커밋 상태로 되돌리기
$ git restore README.md
$ git status -s
# 출력 없음 (깨끗한 상태)
```

이제 실수로 추적한 로그 파일을 `.gitignore`와 `git rm --cached`로 분리해 보자.

```bash
$ echo "DEBUG token=1234" > debug.log
$ git add debug.log
$ git status -s
A  debug.log

# 1. Index(추적 대상)에서만 제거하고 로컬 디스크 파일은 보존한다
$ git rm --cached debug.log
$ ls -la debug.log

# 2. Vim으로 .gitignore 파일을 만들어 무시 패턴을 등록한다
$ vim .gitignore
```

```gitignore
*.log
secret.env
```

```bash
$ git status -s
?? .gitignore
# debug.log는 .gitignore에 의해 아예 목록에서 사라졌다!

$ git add .gitignore
$ git commit -m "chore: ignore log and env files"
```

---

## 6. 브랜치(41바이트 포인터), 병합(Merge), 충돌 해결, 리베이스(Rebase) 실습

이제 Pro Git 3장의 핵심인 **브랜치 포인터의 물리적 실체**와 **Fast-forward / 3-Way Merge / Merge Conflict / Rebase**를 차례대로 재현해 본다.

{{< figure src="branch-merge-rebase-remote-lab.svg" alt="41바이트 브랜치 포인터, 머지, 충돌 해결, Rebase 및 원격 동기화·태그 실습 흐름 다이어그램" caption="로컬 .git 내부 포인터(HEAD, refs/heads) 확인과 Fast-forward · 3-Way Merge · Rebase, 그리고 원격 Bare 저장소 동기화 및 태그 배포 흐름 (Pro Git 3.1~3.6 실습 통합)" width="820" height="560" >}}

### ① 브랜치가 41바이트 파일임을 눈으로 직접 확인하기

먼저 `feature/login` 브랜치를 생성만(전환하지 않고) 한 뒤 `.git/refs/heads/` 안을 직접 출력해 보자.

```bash
$ git branch feature/login
$ cat .git/refs/heads/main
(main의 40자 SHA-1 해시 출력)

$ cat .git/refs/heads/feature/login
(main과 완전히 동일한 40자 SHA-1 해시 출력)

$ wc -c .git/refs/heads/main
      41 .git/refs/heads/main

$ cat .git/HEAD
ref: refs/heads/main
```

정말로 Git의 브랜치는 프로젝트 폴더를 복사한 것이 아니라 **40자리 SHA-1 커밋 해시 + 개행 문자 1바이트 = 정확히 41바이트짜리 텍스트 파일**이며, `HEAD`는 그중 지금 활성화된 브랜치 경로 하나를 가리킬 뿐이다!

### ② 긴급 `hotfix` 생성과 Fast-forward Merge

이제 운영 버그를 수정하기 위해 `hotfix` 브랜치를 생성해 전환하고 커밋한 뒤, `main`으로 돌아와 병합한다.

```bash
# 새 브랜치 생성과 동시에 전환 (Git 2.23+ 권장 문법)
$ git switch -c hotfix
$ cat .git/HEAD
ref: refs/heads/hotfix

$ echo "Hotfix: fixed typo." >> README.md
$ git commit -am "fix: update README via hotfix"

# main으로 복귀 -> Working Tree의 README.md가 자동으로 hotfix 이전 상태로 되돌아간다!
$ git switch main
$ cat README.md

# hotfix 병합: main이 hotfix의 직계 조상이므로 Fast-forward 발생!
$ git merge hotfix
Updating ...
Fast-forward
 README.md | 1 +
 1 file changed, 1 insertion(+)

# 병합 완료된 브랜치 목록 확인 후 삭제
$ git branch --merged
  hotfix
* main
$ git branch -d hotfix
```

### ③ 분기된 이력에서 Merge Conflict 유도 및 Vim 해결 (3-Way Merge)

이제 `feature/login`과 `main` 양쪽에서 **`calc.py`의 똑같은 마지막 줄 아래를 서로 다르게 수정**해 병합 충돌(Merge Conflict)을 일부러 일으키고 해결해 보자.

```bash
# 1. feature/login 브랜치로 이동해 calc.py 끝에 login 함수 추가 후 커밋
$ git switch feature/login
$ vim calc.py
```

파일 맨 아래(`return a / b` 다음 줄)에 아래 함수를 추가하고 `:wq`로 저장한 뒤 커밋한다.

```python
def login(user):
    return f"Welcome, {user}"
```

```bash
$ git commit -am "feat: add user login greeting"

# 2. main 브랜치로 복귀해 같은 위치에 서로 다른 auth 함수 추가 후 커밋
$ git switch main
$ vim calc.py
```

마찬가지로 `return a / b` 다음 줄에 아래 함수를 추가하고 `:wq`로 저장한 뒤 커밋한다.

```python
def auth(user):
    return f"Authenticated: {user}"
```

```bash
$ git commit -am "feat: add auth check helper"

# 3. main에서 feature/login 병합 시도 -> 충돌 발생!
$ git merge feature/login
Auto-merging calc.py
CONFLICT (content): Merge conflict in calc.py
Automatic merge failed; fix conflicts and then commit the result.
```

병합이 일시 정지된 지금, `.git` 내부와 `git status`를 확인해 보자.

```bash
$ cat .git/MERGE_HEAD
# 상대 브랜치(feature/login) 끝 커밋 해시가 임시 기록되어 있다

$ git status
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")
Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:      calc.py
```

이제 `vim calc.py`로 파일을 열어 Git이 삽입한 충돌 마커를 직접 확인하고 해결한다.

```bash
$ vim calc.py
```

Vim 화면 하단부에 아래와 같은 표준 충돌 마커가 보인다.

```python
<<<<<<< HEAD
def auth(user):
    return f"Authenticated: {user}"
=======
def login(user):
    return f"Welcome, {user}"
>>>>>>> feature/login
```

두 함수 모두 유지하기로 결정했다면, Normal 모드에서 `<<<<<<< HEAD`, `=======`, `>>>>>>> feature/login` 마커 세 줄 위로 커서를 옮겨 **`dd`로 각각 삭제**해 아래와 같은 최종 모습으로 만든 뒤 `:wq`로 저장한다.

```python
def auth(user):
    return f"Authenticated: {user}"
def login(user):
    return f"Welcome, {user}"
```

이제 `git add`로 충돌 해결 완료를 알리고 병합 커밋을 확정한다.

```bash
$ git add calc.py
$ git commit
# Vim(.git/MERGE_MSG)이 열리면 기본 메시지(Merge branch 'feature/login') 확인 후 :wq

# 부모가 2개인 Merge Commit이 생성되었는지 확인!
$ git cat-file -p HEAD
tree ...
parent ... (main 직전 커밋)
parent ... (feature/login 커밋)
author Goru Dev ...

$ git lg
$ git branch -d feature/login
```

`git cat-file -p HEAD` 출력에서 `parent` 줄이 정확히 **2개** 찍힌다면, 3-Way Merge의 결과를 완벽히 눈으로 확인한 셈이다!

### ④ `git rebase`로 과거 분기 커밋을 최신 `main` 위로 재적용하기

이번에는 과거 커밋(`HEAD~2`, 즉 `hotfix` 반영 직후 시점)에서 `feature/ui` 브랜치를 분기해 작업한 뒤, Merge가 아닌 **Rebase**로 `main` 끝으로 옮겨 보자.

```bash
# 2단계 전 커밋을 기점으로 feature/ui 브랜치 생성
$ git switch -c feature/ui HEAD~2
$ echo "UI Banner v1" > ui.txt
$ git add ui.txt
$ git commit -m "feat: add UI banner"

# 현재 브랜치 분기 그래프와 feature/ui 커밋 해시를 눈으로 기억해 두자
$ git lg --all

# main의 최신 커밋 위로 feature/ui의 패치를 재적용(Rebase)
$ git rebase main

# 다시 그래프 확인: feature/ui 커밋의 SHA-1 해시가 새로 바뀌며 main 바로 위 일직선에 붙었다!
$ git lg --all

# 이제 main으로 돌아와 Fast-forward로 깔끔하게 통합
$ git switch main
$ git merge feature/ui
$ git branch -d feature/ui
```

---

## 7. 원격 저장소 동기화(`push`, `fetch`, `pull`)와 태그(Tag) 릴리즈 실습

마지막으로 내 로컬 저장소를 원격 서버와 연결하고, 다른 동료와 동시에 작업할 때 `origin/main` 리모트 트래킹 브랜치가 어떻게 움직이는지 재현한다.

### ① 원격 저장소 준비 (`--bare`) 및 `git push -u origin main`

GitHub에 빈 저장소를 만들어 URL을 연결해도 되지만, 여기서는 네트워크 없이도 즉시 동작하도록 내 PC 안에 작업 디렉토리 없는 **원격 서버용 Bare 저장소**(`remote-repo.git`)를 하나 만들어 사용한다. (GitHub를 쓴다면 URL 자리에 `https://github.com/<user>/<repo>.git`을 넣으면 이후 과정은 100% 같다.)

```bash
# 1. 작업 트리 없는 원격 Bare 저장소 생성
$ git init --bare ~/git-lab/remote-repo.git

# 2. 내 로컬 저장소(git-practice)에 origin 원격으로 등록
$ cd ~/git-lab/git-practice
$ git remote add origin ~/git-lab/remote-repo.git
$ git remote -v

# 3. main 브랜치를 origin에 푸시하면서 업스트림(-u) 추적 관계 설정
$ git push -u origin main

# 4. 생성된 리모트 트래킹 브랜치 포인터와 추적 상태 확인
$ cat .git/refs/remotes/origin/main
$ git branch -vv
* main 5a6efaf [origin/main] feat: add UI banner
```

### ② 동료의 푸시 재현과 `git fetch` vs `git merge @{u}`

이제 `~/git-lab/colleague-repo` 경로에 원격을 새로 `clone`하여 동료 역할을 해 보자.

```bash
# 1. 동료 PC를 시뮬레이션해 remote-repo.git 클론 후 새 커밋 푸시
$ cd ~/git-lab
$ git clone ~/git-lab/remote-repo.git colleague-repo
$ cd colleague-repo
$ echo "Peer update" >> README.md
$ git commit -am "docs: add peer note"
$ git push origin main
```

다시 내 원래 저장소(`~/git-lab/git-practice`)로 돌아가 `git branch -vv`를 쳐 보면, **네트워크 통신(`fetch`) 전까지는 여전히 `[origin/main]` 최신 상태라고 착각**하고 있음을 확인할 수 있다.

```bash
$ cd ~/git-lab/git-practice
$ git branch -vv
* main 5a6efaf [origin/main] feat: add UI banner

# git fetch origin 실행: Working Tree는 건드리지 않고 origin/main 포인터만 갱신!
$ git fetch origin
$ git branch -vv
* main 5a6efaf [origin/main: behind 1] feat: add UI banner

# 그래픽 로그로 내 main과 한 칸 앞선 origin/main 확인
$ git lg --all

# 업스트림 단축 표기(@{u})를 이용해 안전하게 병합 (Fast-forward)
$ git merge @{u}
$ cat README.md
```

만약 내 로컬 `main`에도 푸시하지 않은 새 커밋이 있고 원격 `origin/main`에도 동료 커밋이 쌓인 상태라면, `git pull --rebase`를 써서 내 로컬 커밋을 동료 커밋 위로 깔끔하게 재정렬할 수도 있다.

### ③ 릴리즈 태깅: Annotated vs Lightweight & `--follow-tags`

모든 실습 이력이 `main`에 깔끔하게 모였으니, 첫 릴리즈 `v1.0.0` 태그를 남기고 원격에 배포하며 마무리한다.

```bash
# 1. 임시 표시용 Lightweight 태그 vs 공식 메타데이터가 담긴 Annotated 태그(-a) 생성
$ git tag v1.0.0-lw
$ git tag -a v1.0.0 -m "Release version 1.0.0: Ch 1-3 Git internals lab"

# 2. 두 태그의 내부 객체 차이 비교 (Annotated 태그만 Tagger, 날짜, 메시지 객체가 존재)
$ git show v1.0.0-lw
$ git show v1.0.0

# 3. --follow-tags 옵션으로 임시 Lightweight 태그는 빼고 Annotated 태그(v1.0.0)만 원격 푸시!
$ git push origin --follow-tags

# 4. 원격에 v1.0.0(태그 객체 + 역참조 커밋 ^{})만 전송되고 v1.0.0-lw는 제외됐는지 최종 확인
$ git ls-remote --tags origin
9ecfa89...	refs/tags/v1.0.0
e6f1810...	refs/tags/v1.0.0^{}
```

---

## 마치며

이 한 편의 실습을 처음부터 끝까지 직접 타이핑하며 통과했다면, 이제 Git 명령어 하나를 입력할 때마다 머릿속에 세 가지 장면이 자연스럽게 그려질 것이다.

1. **`add` · `diff` · `commit` · `restore`** 는 모두 **Working Tree, Staging Area(`.git/index`), `.git` 디렉토리** 세 공간 사이에서 스냅샷을 복사하고 경계를 비교하는 동작이다.
2. **브랜치와 리모트 트래킹 브랜치(`origin/main`)** 는 무거운 디렉토리 복사가 아니라 `.git/refs/` 안에 놓인 **41바이트 커밋 해시 포인터**이며, `HEAD`는 내가 현재 바라보는 포인터다.
3. **Fast-forward, 3-Way Merge(Merge Commit), Rebase(`C8 → C8'`)** 는 이 커밋 그래프 위의 포인터와 패치를 어떻게 이어 붙일지 결정하는 서로 다른 전략일 뿐이다.

이론편([#1]({{< ref "/posts/pro-git-internals-ch1" >}})~[#6]({{< ref "/posts/pro-git-internals-ch3-part3" >}}))과 이번 실습편이 Git을 단순히 암기해서 쓰는 도구에서 **원리를 알고 자유롭게 다루는 강력한 무기**로 바꾸는 데 든든한 발판이 되기를 바란다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - Chapter 1. Getting Started](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - Chapter 2. Git Basics](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - Chapter 3. Git Branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
- [Git for Windows 공식 사이트](https://gitforwindows.org/)
