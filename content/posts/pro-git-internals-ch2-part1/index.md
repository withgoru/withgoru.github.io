+++
date = '2026-11-21T14:20:00+09:00'
draft = false
title = 'Git 내부 원리 #2: 파일 상태와 diff·reset·restore'
description = 'Pro Git 2장 전반부를 바탕으로 파일 상태 생명주기, git add·diff의 비교 기준, git log 필터링(-S), reset·restore 되돌리기 원리를 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글]({{< ref "/posts/pro-git-internals-ch1" >}})에서는 Git이 데이터를 변경분(Delta)의 목록이 아니라 스냅샷의 스트림으로 저장하며, **Working Tree**, **Staging Area**(Index), **Git Directory**(`.git`)라는 세 영역을 통해 동작한다는 점을 살펴보았다.

이번 글에서는 **Pro Git 2장**(Git Basics)의 전반부(2.1~2.4)를 다룬다. 현업에서 매일 습관처럼 입력하는 `init`, `clone`, `add`, `status`, `diff`, `commit`, `rm`, `mv`, `log`, `restore`가 내부의 세 영역 사이에서 파일 상태를 어떻게 전이시키는지 원리부터 차근차근 짚어본다.

---

## 1. Git 저장소 획득하기: `init`과 `clone`의 본질 (Pro Git 2.1)

로컬 머신에서 Git 저장소를 준비하는 방법은 두 가지뿐이다. 기존 디렉토리를 새 Git 저장소로 초기화하거나, 다른 곳에 이미 존재하는 Git 저장소를 복제해 오는 것이다.

### 기존 디렉토리에서 저장소 초기화: `git init`

버전 관리가 되지 않던 프로젝트 디렉토리로 이동해 `git init`을 실행하면, Git 저장소의 뼈대(Skeleton) 역할을 하는 필수 파일들이 담긴 `.git` 하위 디렉토리가 생성된다.

```bash
$ cd /Users/user/my_project
$ git init
```

이 시점에는 프로젝트 안의 어떤 파일도 아직 추적(Track)되지 않는다. 기존 파일들을 버전 관리에 포함하려면 `git add`로 추적할 대상을 지정한 뒤 초기 커밋을 생성해야 한다.

```bash
$ git add *.c
$ git add LICENSE
$ git commit -m 'Initial project version'
```

### 기존 저장소 복제: 왜 `checkout`이 아니라 `clone`인가

이미 존재하는 원격 저장소에 기여하거나 코드를 내려받을 때는 `git clone`을 사용한다. Subversion(SVN) 같은 중앙 집중식 VCS 경험자가 가장 먼저 짚고 넘어가야 할 차이는 명령어가 `checkout`이 아니라 **클론**(`clone`)이라는 점이다.

SVN의 `checkout`이 특정 시점의 작업 사본(Working copy)만 가져오는 것과 달리, `git clone`은 **서버가 보유한 거의 모든 데이터와 프로젝트 전체 파일의 모든 히스토리를 통째로 복제**한다. 서버 디스크가 손상되더라도 클라이언트의 클론 저장소 중 하나를 다시 서버로 올려 복구할 수 있는 이유가 여기에 있다(서버 측 Hook 등 일부 설정 제외).

```bash
# 기본 사용법: 현재 위치에 저장소 이름(libgit2)의 디렉토리 생성
$ git clone https://github.com/libgit2/libgit2

# 다른 이름의 디렉토리(mylibgit)로 복제하고 싶을 때
$ git clone https://github.com/libgit2/libgit2 mylibgit
```

Git은 `https://` 외에도 `git://` 프로토콜이나 SSH 기반의 `user@server:path/to/repo.git` 등 다양한 전송 프로토콜을 지원한다.

---

## 2. 변경 사항 기록하기: 파일 생명주기와 Index 조작 원리 (Pro Git 2.2)

저장소를 복제하거나 초기화했다면 이제 작업 디렉토리의 파일들이 어떤 상태를 거치며 `.git` 데이터베이스에 기록되는지 정확히 이해해야 한다.

### Tracked vs Untracked, 그리고 4가지 상태 생명주기

작업 디렉토리(Working Directory)의 모든 파일은 크게 **Tracked**(추적됨)와 **Untracked**(추적되지 않음) 두 그룹으로 나뉜다.

- **Tracked**: 직전 스냅샷(커밋)에 포함되어 있었거나, 새로 스테이징 영역에 추가된 파일이다. Git이 존재를 알고 있는 파일이며, **Unmodified**(수정 안 됨), **Modified**(수정됨), **Staged**(스테이징됨) 중 하나의 상태에 놓인다.
- **Untracked**: 나머지 모든 파일이다. 직전 스냅샷에도 없었고 스테이징 영역에도 올라가지 않은 파일로, 사용자가 명시적으로 추적하라고 지시하기 전까지는 커밋 스냅샷에 포함되지 않는다. (빌드 산출물이나 바이너리 파일이 실수로 커밋되는 것을 막기 위한 안전장치다.)

{{< figure src="file-status-lifecycle.svg" alt="Git 파일 상태 생명주기와 git diff 비교 경계를 나타낸 다이어그램" caption="Git 파일의 4가지 상태 전이 생명주기와 `git diff` / `git diff --staged` 비교 범위 (Pro Git 2.2 도식 재구성)" width="820" height="480" >}}

### `git add`의 진짜 의미와 동시에 Staged/Unstaged가 되는 현상

현업에서 가장 자주 오해하는 부분이 `git add`의 역할이다. 새 파일(`README`)을 만들었을 때 `git status`로 확인하면 `Untracked files` 아래에 표시되고, `git add README`를 실행하면 `Changes to be committed` 아래에 `new file: README`로 올라간다. `git add`에 디렉토리 경로를 넘기면 하위 파일들을 재귀적으로 모두 추가한다.

그런데 이미 추적 중인 `CONTRIBUTING.md` 파일을 수정해 `git add CONTRIBUTING.md`로 스테이징한 뒤, 커밋하기 전에 파일을 한 번 더 수정하고 `git status`를 확인하면 기묘한 출력이 나타난다.

```bash
$ git add CONTRIBUTING.md
$ vim CONTRIBUTING.md
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)
	new file:   README
	modified:   CONTRIBUTING.md

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)
	modified:   CONTRIBUTING.md
```

같은 `CONTRIBUTING.md` 파일이 **Staged 영역과 Unstaged 영역 양쪽에 동시에** 잡힌 것이다. 이런 현상이 벌어지는 이유는 `git add`가 "*이 파일을 프로젝트 목록에 등록한다*"가 아니라, "**이 명령어를 실행한 바로 그 순간의 파일 내용을 다음 커밋(Index)에 담는다**"라는 의미로 동작하기 때문이다.

따라서 지금 상태에서 `git commit`을 실행하면, 작업 디렉토리의 최신 수정본이 아니라 **마지막으로 `git add`를 실행했던 시점의 스냅샷**이 커밋된다. 최신 수정 내용까지 포함하려면 `git add CONTRIBUTING.md`를 한 번 더 실행해 Index를 덮어써야 한다.

### `git status -s` (`--short`) 2열 코드 해석하기

`git status`의 기본 출력은 친절하지만 길다. `-s` 또는 `--short` 옵션을 쓰면 두 개의 컬럼으로 압축된 상태 정보를 볼 수 있다.

```bash
$ git status -s
 M README
MM Rakefile
A  lib/git.rb
M  lib/simplegit.rb
?? LICENSE.txt
```

핵심은 **왼쪽 컬럼이 Staging Area(Index)의 상태**이고, **오른쪽 컬럼이 Working Tree의 상태**라는 점이다.
- ` M README`: Index에는 변경이 없고, Working Tree에서만 수정됨(Unstaged).
- `MM Rakefile`: 수정 후 `git add`로 스테이징한 뒤(왼쪽 `M`), Working Tree에서 한 번 더 수정됨(오른쪽 `M`).
- `A  lib/git.rb`: 새로 추가되어 Index에 스테이징됨.
- `M  lib/simplegit.rb`: 수정되어 Index에 스테이징됨.
- `?? LICENSE.txt`: 아직 추적되지 않는 Untracked 파일.

### `.gitignore` 패턴 규칙과 중첩 적용

로그 파일이나 빌드 결과물처럼 Git이 추적하지도, Untracked 목록에 띄우지도 않길 원하는 파일 패턴은 `.gitignore`에 정의한다.

```gitignore
# 모든 .a 확장자 파일 무시
*.a

# 위에서 *.a를 무시했더라도 lib.a는 예외적으로 추적
!lib.a

# 하위 디렉토리(subdir/TODO)가 아닌 현재 디렉토리의 TODO 파일만 무시
/TODO

# 이름이 build인 모든 디렉토리 내 파일 무시
build/

# doc/notes.txt는 무시하되 doc/server/arch.txt는 무시하지 않음
doc/*.txt

# doc/ 및 모든 하위 디렉토리의 .pdf 파일 무시
doc/**/*.pdf
```

- 빈 줄과 `#`로 시작하는 주석은 무시된다.
- 표준 Glob 패턴(`*` 0개 이상 문자, `?` 1개 문자, `[abc]` 또는 `[0-9]` 문자 범위, `**` 중첩 디렉토리)이 작업 트리 전체에 재귀적으로 적용된다.
- 저장소 루트뿐만 아니라 하위 디렉토리에도 별도의 `.gitignore`를 둘 수 있으며, 중첩된 `.gitignore`는 해당 디렉토리 하위에만 적용된다(참고로 리눅스 커널 소스 저장소에는 무려 206개의 `.gitignore` 파일이 존재한다).

### `git diff` vs `git diff --staged` (`--cached`)

`git status`가 어떤 파일이 바뀌었는지 알려준다면, `git diff`는 정확히 어떤 줄이 추가되고 삭제되었는지 패치(Patch) 단위로 보여준다.

- **`git diff`** (인자 없음): **Working Tree와 Staging Area(Index)를 비교**한다. 즉, 수정했지만 아직 스테이징하지 않은(Unstaged) 변경 사항만 보여준다. 만약 모든 변경 파일을 `git add`로 스테이징했다면, 두 영역의 내용이 같아지므로 `git diff`는 아무것도 출력하지 않는다.
- **`git diff --staged`** (동의어: **`git diff --cached`**): **Staging Area(Index)와 직전 커밋(`HEAD`)을 비교**한다. 즉, 다음 `git commit`을 실행했을 때 기록될 내용을 미리 검토할 때 사용한다.

```bash
$ git add CONTRIBUTING.md
$ echo '# test line' >> CONTRIBUTING.md

# 아직 스테이징하지 않은 '# test line'만 출력
$ git diff
diff --git a/CONTRIBUTING.md b/CONTRIBUTING.md
index 643e24f..87f08c8 100644
--- a/CONTRIBUTING.md
+++ b/CONTRIBUTING.md
@@ -119,3 +119,4 @@ at the
 ## Starter Projects
 See our [projects list](https://github.com/libgit2/libgit2/blob/development/PROJECTS.md).
+# test line

# 이미 스테이징되어 다음 커밋에 들어갈 변경분 출력
$ git diff --cached
diff --git a/CONTRIBUTING.md b/CONTRIBUTING.md
index 8ebb991..643e24f 100644
--- a/CONTRIBUTING.md
+++ b/CONTRIBUTING.md
@@ -65,7 +65,8 @@ branch directly, things can get messy.
...
```

외부 GUI 도구로 비교하고 싶다면 `git difftool`을 사용할 수 있으며, `git difftool --tool-help`로 시스템에 설치된 도구를 확인할 수 있다.

### `git commit`과 `-a` 단축 옵션의 동작

인자 없이 `git commit`을 실행하면 `core.editor`에 설정된 편집기(`.git/COMMIT_EDITMSG`)가 열리며 `git status` 요약이 주석(`#`)으로 제공된다. `-v` 플래그를 함께 넘기면 편집기 하단에 변경된 diff 전체가 함께 표시되어 실수를 줄일 수 있다. 인라인으로 커밋 메시지를 남기려면 `-m` 옵션을 쓴다.

```bash
$ git commit -m "Story 182: fix benchmarks for speed"
[master 463dc4f] Story 182: fix benchmarks for speed
 2 files changed, 2 insertions(+)
 create mode 100644 README
```

매번 `git add`를 거치는 과정이 번거롭다면 **`git commit -a`** 옵션을 사용할 수 있다. 이는 커밋 직전에 **이미 추적 중인(Tracked) 모든 수정 파일을 자동으로 스테이징**해 주는 단축 옵션이다(새로 만든 Untracked 파일은 포함되지 않는다). 편리하지만 의도치 않은 디버깅 코드까지 함께 커밋될 수 있으므로 주의가 필요하다.

### `git rm`과 `git mv`의 내부 동작

#### 파일 삭제: `git rm`과 `--cached`
셸에서 단순히 `rm PROJECTS.md`만 실행하면 작업 디렉토리에서만 삭제되고 Index에는 여전히 파일이 남아 `Changes not staged for commit`에 `deleted`로 뜬다. 이를 커밋에 반영하려면 `git rm PROJECTS.md`를 실행해 **Index에서도 삭제를 스테이징**해야 한다.

- 이미 수정했거나 Index에 스테이징한 파일을 삭제하려면 안전장치 때문에 `-f` 옵션을 강제로 줘야 한다(아직 스냅샷으로 저장되지 않은 데이터의 유실을 막기 위함).
- 실수로 로그 파일이나 `.a` 컴파일 파일을 추적해 버렸을 때, **로컬 디스크의 파일은 그대로 유지하면서 Git의 추적(Index)에서만 제거**하려면 `--cached` 옵션을 사용한다.
- `git rm log/\*.log`처럼 글로브 패턴을 쓸 때 백슬래시(`\`)를 붙이는 이유는, 셸의 파일명 확장과 별개로 Git이 자체적인 파일명 확장 규칙을 수행하도록 넘기기 위해서다.

```bash
# 로컬 파일은 유지하고 Git 추적(Staging Area)에서만 제거
$ git rm --cached README
```

#### 파일 이동: Git은 Rename 메타데이터를 저장하지 않는다
많은 VCS와 달리 **Git은 파일 이름 변경(Rename)이나 이동에 대한 별도의 메타데이터를 저장하지 않는다.** 그럼에도 `git mv` 명령어가 존재하고 `git status`에서 `renamed: README.md -> README`로 표시되는 이유는 무엇일까?

```bash
$ git mv README.md README
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)
	renamed:    README.md -> README
```

실제로 아래 세 줄의 명령어는 `git mv README.md README`와 내부적으로 **100% 동일**하다.

```bash
$ mv README.md README
$ git rm README.md
$ git add README
```

`git mv`는 세 단계를 한 번에 처리해 주는 편의 함수(Convenience function)일 뿐이다. Git은 커밋 스냅샷 간의 파일 내용과 해시를 사후에 비교해 이름 변경을 암묵적(Implicitly)으로 추론해 낸다. 따라서 IDE나 탐색기에서 파일 이름을 직접 바꾼 뒤 나중에 `rm`/`add`로 정리해도 결과는 완전히 같다.

---

## 3. 커밋 히스토리 조회하기: `git log`와 Pickaxe 검색 (Pro Git 2.3)

커밋이 쌓였거나 기존 저장소를 클론했다면 `git log`로 히스토리를 추적한다. 아무 옵션 없이 실행하면 최근 커밋부터 역순(Reverse chronological order)으로 커밋의 SHA-1 체크섬, Author, Date, 커밋 메시지를 출력한다.

### 출력 포맷 제어와 Author vs Committer

실무에서 가장 유용한 출력 옵션들은 다음과 같다.

- **`-p` (`--patch`)**: 각 커밋에 도입된 실제 코드 변경분(Diff)을 함께 출력한다. `-2` 같은 숫자 옵션을 결합하면 최근 N개의 커밋만 추려낸다(`git log -p -2`).
- **`--stat`**: 수정된 파일 목록과 파일별 추가/삭제 라인 수 통계를 요약해 준다.
- **`--pretty`**: 출력 형식을 바꾼다. `oneline`, `short`, `full`, `fuller` 외에 `format`을 쓰면 머신 파싱에 적합한 커스텀 포맷을 지정할 수 있다.
- **`--graph`**: ASCII 문자로 브랜치 분기와 병합 히스토리를 시각화한다(`git log --pretty=format:"%h %s" --graph`).
- **`--oneline`**: `--pretty=oneline --abbrev-commit`을 합친 단축 옵션이다.

`--pretty=format:"%h - %an, %ar : %s"`처럼 커스텀 포맷을 작성할 때 자주 쓰는 주요 지정자(Specifier)는 다음과 같다.

| 지정자 | 출력 내용 |
| :--- | :--- |
| `%H` / `%h` | 전체 커밋 해시 / 축약 커밋 해시 |
| `%T` / `%t` | 전체 트리(Tree) 해시 / 축약 트리 해시 |
| `%P` / `%p` | 전체 부모(Parent) 해시 / 축약 부모 해시 |
| `%an` / `%ae` | **Author** 이름 / 이메일 |
| `%ad` / `%ar` | Author 작성일 / 상대 시간 (예: `6 years ago`) |
| `%cn` / `%ce` | **Committer** 이름 / 이메일 |
| `%cd` / `%cr` | Committer 적용일 / 상대 시간 |
| `%s` | 커밋 제목 (Subject) |

여기서 **Author**와 **Committer**의 차이를 구분해야 한다. **Author**는 코드를 처음 작성한 원작자이고, **Committer**는 그 패치를 저장소에 최종 적용한 사람이다. 오픈소스 프로젝트에 패치를 보내 메인테이너가 이를 병합하면, 원작자는 Author로, 메인테이너는 Committer로 각각 기록된다.

### 조건부 필터링과 Pickaxe(`-S`) 검색

방대한 히스토리에서 원하는 커밋만 정밀하게 추려내는 필터링 옵션도 강력하다.

- **기간 제한**: `--since` (`--after`), `--until` (`--before`)에 `2.weeks`, `"2008-01-15"`, `"2 years 1 day 3 minutes ago"` 같은 절대·상대 날짜를 넘긴다.
- **작성자 및 메시지 검색**: `--author`, `--committer`, `--grep`을 사용한다. `--author`나 `--grep`을 여러 번 넘기면 기본적으로 OR 조건(하나라도 일치)으로 동작하며, 모든 조건을 동시에 만족(AND)시키려면 `--all-match`를 추가한다.
- **머지 커밋 제외**: `--no-merges`를 붙이면 정보량이 적은 머지 커밋을 걸러내고 실제 변경 커밋만 볼 수 있다.
- **경로 지정**: 옵션 맨 마지막에 `--`로 구분한 뒤 파일이나 디렉토리 경로를 넘기면(`git log -- path/to/file`) 해당 경로를 건드린 커밋만 조회한다.
- **Pickaxe 옵션 (`-S`)**: **특정 문자열(함수명, 변수명 등)의 등장 횟수가 변한 커밋**—즉, 해당 코드를 처음 추가했거나 삭제한 커밋만 정확히 찾아낸다.

```bash
# 특정 함수가 추가되거나 제거된 커밋 찾기 (Pickaxe)
$ git log -S function_name

# Git 소스 코드 약 4만 개 커밋 중 Junio C Hamano가 2008년 10월에 t/ 디렉토리에 남긴 논-머지 커밋 조회
$ git log --pretty="%h - %s" --author='Junio C Hamano' --since="2008-10-01" \
  --before="2008-11-01" --no-merges -- t/
```

---

## 4. 되돌리기(Undoing Things): `amend`, `reset`, `restore` (Pro Git 2.4)

작업 중 실수를 되돌리는 기능은 Git에서 드물게 **잘못 쓰면 작업 내용을 영구적으로 잃을 수 있는 영역**이므로 내부 동작을 명확히 알고 사용해야 한다.

### 직전 커밋 수정의 실체: `git commit --amend`

커밋을 너무 일찍 해서 파일을 하나 빠뜨렸거나 커밋 메시지에 오타를 냈다면, 빠진 파일을 `git add`로 스테이징한 뒤 `--amend` 옵션으로 다시 커밋한다.

```bash
$ git commit -m 'Initial commit'
$ git add forgotten_file
$ git commit --amend
```

만약 아무런 파일 변경 없이 바로 `git commit --amend`만 실행하면 직전 스냅샷은 그대로 둔 채 커밋 메시지만 편집한다.

중요한 사실은 `--amend`가 기존 커밋을 제자리에서 고치는 것이 아니라, **스테이징 영역을 바탕으로 완전히 새로운 커밋 객체를 만들어 기존 커밋을 대체**(Replace)한다는 점이다. 따라서 기존 커밋 해시는 히스토리에서 밀려나 사라진다. 이 때문에 **이미 원격 저장소에 푸시(Push)한 커밋에는 절대 `--amend`를 쓰지 않는 것**이 원칙이다(강제 푸시 시 협업자들의 히스토리가 꼬이게 된다).

### Staged 파일 언스테이징과 Modified 파일 변경 취소

실수로 `git add *`를 입력해 두 파일을 모두 스테이징했거나, 작업 디렉토리에서 수정한 내용을 직전 커밋 상태로 폐기하고 싶을 때 전통적으로 Git은 `git reset`과 `git checkout`을 안내했다.

- **전통적인 방식 (Git 2.23 미만)**
  - Unstaging (Index → Modified): `git reset HEAD CONTRIBUTING.md` (워킹 디렉토리의 파일 내용은 건드리지 않고 Index 상태만 직전 커밋 상태로 되돌린다. `--hard` 플래그를 주지 않는 한 비교적 안전하다.)
  - Unmodifying (Modified → 직전 커밋 상태로 폐기): `git checkout -- CONTRIBUTING.md`

하지만 `checkout`이 브랜치 전환과 파일 복원을 동시에 맡고, `reset` 역시 여러 용도로 쓰여 혼란이 컸기 때문에 **Git 2.23.0부터는 역할을 명확히 분리한 `git restore` 명령어가 도입**되었고, **Git 2.51(2025.08)** 에 이르러 `git switch`와 함께 공식적으로 실험적(experimental) 꼬리표를 떼고 정식 안정 명령어로 승격되었다. 최신 Git에서 `git status` 역시 `git restore` 사용을 권장한다.

```bash
# 1. 실수로 스테이징한 파일들 확인
$ git add *
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   CONTRIBUTING.md
	renamed:    README.md -> README

# 2. Index에서만 내리기 (Unstage): 워킹 디렉토리의 수정 사항은 보존됨
$ git restore --staged CONTRIBUTING.md

# 3. 워킹 디렉토리의 수정 사항까지 완전히 폐기하고 직전 커밋/스테이징 상태로 되돌리기
$ git restore CONTRIBUTING.md
```

### 데이터 복구의 대원칙

Pro Git 2.4가 마지막으로 남기는 경고는 실무에서 반드시 기억해야 할 대원칙이다.

- **커밋된 데이터**: 한 번이라도 커밋한 스냅샷은 설령 브랜치를 삭제했거나 `--amend`로 덮어썼더라도 Git 내부 데이터베이스에서 거의 항상 복구할 수 있다.
- **커밋되지 않은 데이터**: 반면 한 번도 커밋하지 않은 로컬 변경 사항을 `git checkout -- <file>`이나 `git restore <file>`로 덮어쓰면, Git이 워킹 디렉토리의 파일을 직전 스테이징/커밋 버전으로 교체해 버리므로 **다시 복구할 수 없다.** 반드시 해당 로컬 변경을 버려도 된다는 확신이 있을 때만 실행해야 하며, 잠시 치워두고 싶다면 브랜치나 `stash`를 사용해야 한다.

---

## 마치며

결국 2장의 앞부분을 관통하는 핵심은 1장에서 정리한 **세 영역(Working Tree, Index, `.git`)의 상호작용**이다. `git add`는 Index에 현재 시점의 스냅샷을 올리는 행위이고, `git diff`와 `git diff --staged`는 각 영역 사이의 경계를 비교하며, `git restore --staged`와 `git restore`는 각각 Index와 Working Tree를 이전 상태로 되돌리는 명령이다.

다음 편에서는 2장의 후반부(2.5~2.7)를 통해 원격 저장소(`git remote`, `fetch`, `pull`, `push`)와 태그(Tagging), 그리고 실무 생산성을 높여주는 Git Alias를 살펴본다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.1 Git Basics - Getting a Git Repository](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.2 Git Basics - Recording Changes to the Repository](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.3 Git Basics - Viewing the Commit History](https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 2.4 Git Basics - Undoing Things](https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things)
