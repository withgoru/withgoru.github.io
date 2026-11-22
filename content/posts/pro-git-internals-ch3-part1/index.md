+++
date = '2026-11-22T16:35:00+09:00'
draft = false
title = '[Pro Git 톺아보기 #4] 브랜치의 내부 실체(41바이트 포인터)와 Fast-forward vs 3-Way Merge 메커니즘'
description = 'Pro Git 3장 전반부(3.1~3.2)를 바탕으로 커밋·트리·블롭 객체 그래프에서 브랜치와 HEAD 포인터가 동작하는 원리, 41바이트 파일로서의 브랜치, 그리고 Fast-forward 병합과 3-Way Merge 및 충돌 해결 과정을 깊게 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글]({{< ref "/posts/pro-git-internals-ch2-part2" >}})에서는 원격 저장소와 로컬 저장소가 `fetch`, `pull`, `push`를 통해 동기화되는 구조, Annotated 태그와 Lightweight 태그의 객체 차이, 그리고 `git lg` Alias 설정까지 살펴보며 Pro Git 2장을 마쳤다.

이번 글부터는 Git이 여타 버전 관리 도구(VCS)와 완전히 궤를 달리하는 핵심 무기, **Pro Git 3장**(Git Branching)을 3부작으로 나누어 파헤친다. 그 첫 번째 편인 이번 글(3.1~3.2)에서는 Git의 "**킬러 피처**"라 불리는 브랜치 모델이 `.git` 내부에서 실제로 어떻게 표현되는지, 그리고 브랜치를 통합할 때 **Fast-forward**와 **3-Way Merge**가 어떤 기준으로 갈리고 **병합 충돌**을 어떻게 해결하는지 해부한다.

---

## 1. 브랜치의 본질: 객체 그래프와 41바이트 포인터 (Pro Git 3.1)

Subversion이나 CVS 같은 전통적인 중앙 집중식 VCS에서 브랜치(Branch)를 만든다는 것은 프로젝트 전체 소스 디렉토리를 통째로 복사하는 값비싼 작업이었다. 프로젝트 크기가 조금만 커져도 브랜치 하나를 따는 데 수 초에서 수 분이 걸렸다. 반면 Git은 하루에도 수십 번씩 브랜치를 만들고 병합하는 워크플로우를 적극 권장한다. 그 비밀은 1장에서 보았던 **스냅샷 기반 데이터 저장 방식**에 있다.

### 커밋 시점에 생성되는 5개의 객체: Commit, Tree, Blob

작업 디렉토리에 `README`, `test.rb`, `LICENSE`라는 세 개의 파일이 있고, 이를 모두 스테이징한 뒤 최초 커밋을 만든다고 가정해 보자.

```bash
$ git add README test.rb LICENSE
$ git commit -m 'Initial commit'
```

이 두 줄의 명령이 실행되는 순간 `.git/objects` 내부에는 정확히 **5개의 객체**가 생성된다.

1. `git add` 시점: 세 파일 각각의 내용에 대한 SHA-1 체크섬을 계산하고, 파일 내용을 **Blob 객체** 3개(`README`, `test.rb`, `LICENSE`)로 저장소에 저장한 뒤 해당 체크섬을 Staging Area(Index)에 등록한다.
2. `git commit` 시점: 루트 디렉토리 구조를 체크섬으로 계산하여 파일 이름과 Blob 해시를 매핑한 **Tree 객체** 1개를 저장한다.
3. 이어서 **Commit 객체** 1개를 생성한다. 이 커밋 객체에는 작성자(Author) 이름과 이메일, 커밋 메시지, 루트 **Tree 객체 포인터**, 그리고 직전 커밋을 가리키는 **부모(Parent) 커밋 포인터**가 담긴다(최초 커밋은 부모가 0개, 일반 커밋은 1개, 병합 커밋은 2개 이상이다).

이후 작업을 이어가며 커밋을 추가로 남기면, 새 커밋 객체가 직전 커밋을 부모로 가리키며 방향성 비순환 그래프(DAG) 체인(`98ca9 ← 34ac2 ← f30ab`)이 자란다.

### 브랜치와 HEAD의 실체: 41바이트 파일

그렇다면 Git에서 **브랜치**의 정체는 무엇일까? 놀랍게도 브랜치는 소스 코드 복사본이 아니라, **커밋 하나를 가리키는 가볍고 이동 가능한 포인터**일 뿐이다. 실제로 `.git/refs/heads/` 디렉토리에 저장되는 각 브랜치는 **40자리의 SHA-1 커밋 해시 문자열 + 1바이트 개행 문자**, 즉 단 **41바이트**짜리 일반 텍스트 파일이다.

> **`master` 브랜치는 특별하지 않다**: 많은 개발자가 `master`(또는 `main`) 브랜치에 특별한 권한이나 메타데이터가 있다고 오해하지만, Git 내부에서 `master`는 다른 브랜치와 완벽히 동일한 41바이트 포인터다. 단지 `git init`이 기본으로 만들어 주는 이름일 뿐이다.

이 상태에서 `testing`이라는 새 브랜치를 만들어 보자.

```bash
$ git branch testing
```

이 명령은 현재 커밋(`f30ab`)을 똑같이 가리키는 `testing`이라는 41바이트 포인터를 하나 더 생성할 뿐이다. 그렇다면 Git은 수많은 브랜치 포인터 중 내가 **지금 어느 브랜치 위에서 작업 중인지** 어떻게 알 수 있을까? 바로 `HEAD`라는 특수 포인터가 현재 로컬 브랜치를 가리키고 있기 때문이다. (`git branch testing`은 포인터만 만들 뿐 `HEAD`를 옮기지는 않으므로 여전히 `HEAD -> master` 상태다.)

`git log --oneline --decorate`로 확인하면 각 브랜치 포인터와 `HEAD`가 가리키는 위치를 정확히 볼 수 있다.

```bash
$ git log --oneline --decorate
f30ab (HEAD -> master, testing) Add feature #32 - ability to add new formats to the central interface
34ac2 Fix bug #1328 - stack overflow under certain conditions
98ca9 Initial commit
```

### 브랜치 전환(`checkout` / `switch`)과 히스토리 분기

이제 `testing` 브랜치로 전환하고 커밋을 하나 만들어 보자.

```bash
$ git checkout testing
$ vim test.rb
$ git commit -a -m 'Make a change'
```

`git checkout testing`을 실행하면 `HEAD`가 `testing` 브랜치를 가리키게 된다. 이 상태에서 커밋(`87ab2`)을 하면 `HEAD`가 가리키는 `testing` 포인터만 앞으로 전진하고, `master` 포인터는 여전히 `f30ab`에 머무른다. 이제 다시 `master`로 돌아가 다른 변경을 커밋해 보자.

```bash
$ git checkout master
$ vim test.rb
$ git commit -a -m 'Make other changes'
```

여기서 반드시 짚고 넘어가야 할 두 가지 중요한 동작이 있다.

1. **`git checkout master`는 두 가지 일을 동시에 수행한다**: 첫째로 `HEAD` 포인터를 `master`로 되돌리고, 둘째로 **Working Directory의 파일들을 `master`가 가리키는 스냅샷(`f30ab`) 상태로 복원**한다. 만약 Working Directory나 Staging Area에 아직 커밋하지 않은 변경 사항이 전환 대상 브랜치의 파일과 충돌한다면, Git은 데이터 유실을 막기 위해 브랜치 전환 자체를 거부한다.
2. **기본 `git log`는 모든 브랜치를 보여 주지 않는다**: `master`로 돌아온 상태에서 단순히 `git log`를 실행하면 방금 `testing`에서 만든 `87ab2` 커밋이 보이지 않는다. 기본값으로 현재 체크아웃된 `HEAD` 아래쪽 조상 히스토리만 추적하기 때문이다. 다른 브랜치까지 함께 보려면 `--all` 옵션과 `--graph` 옵션을 주어야 한다.

```bash
$ git log --oneline --decorate --graph --all
* c2b9e (HEAD -> master) Make other changes
| * 87ab2 (testing) Make a change
|/
* f30ab Add feature #32 - ability to add new formats to the central interface
* 34ac2 Fix bug #1328 - stack overflow under certain conditions
* 98ca9 Initial commit of my project
```

> **Git 2.23+ `git switch` 도입**: 기존 `git checkout`은 브랜치 전환과 파일 복원 역할을 모두 맡아 혼란스럽다는 지적이 많았다. Git 2.23부터는 브랜치 전환 전용 명령어인 `git switch`가 추가되었다.
> - 기존 브랜치로 전환: `git switch testing`
> - 새 브랜치 생성과 동시에 전환(`git checkout -b` 대체): `git switch -c new-branch` (`--create`)
> - 직전 브랜치로 즉시 복귀: `git switch -`

지금까지 살펴본 커밋 객체 그래프(`Commit -> Tree -> Blob`), 41바이트 브랜치 포인터와 `HEAD`의 분기, 그리고 이어질 두 가지 병합 메커니즘을 한 장의 다이어그램으로 정리하면 아래와 같다.

{{< figure src="branch-and-merge.svg" alt="Git 객체 그래프(Commit, Tree, Blob)와 브랜치·HEAD 포인터 및 Fast-forward vs 3-Way Merge 비교 다이어그램" caption="커밋·트리·블롭 객체 참조 구조와 브랜치 포인터의 이동, 그리고 Fast-forward와 3-Way Merge의 동작 차이 (Pro Git 3.1~3.2 재구성)" width="820" height="540" >}}

---

## 2. 실전 브랜칭과 병합: Fast-forward vs 3-Way Merge (Pro Git 3.2)

브랜치가 41바이트 포인터에 불과하다는 사실이 실제 협업 시나리오에서 어떤 위력을 발휘하는지 Pro Git의 고전적인 시나리오로 따라가 보자.

### 이슈 작업 중 긴급 핫픽스 대응과 Fast-forward Merge

현재 운영 코드가 담긴 `master` 브랜치 위에서 이슈 `#53`을 해결하기 위해 `iss53` 브랜치를 생성해 전환(`git checkout -b iss53`)하고 작업 커밋(`C3`)을 남겼다고 가정하자.

```bash
$ git checkout -b iss53
Switched to a new branch "iss53"
$ vim index.html
$ git commit -a -m 'Create new footer [issue 53]'
```

그때 갑자기 운영 웹사이트에서 치명적인 버그가 발생했다는 연락이 온다. Git에서는 진행 중인 `iss53` 변경을 되돌리거나 함께 배포할 필요가 없다. 커밋이 완료된 깨끗한 상태라면 그대로 `master`로 돌아가 `hotfix` 브랜치를 따서 버그만 고치면 된다.

```bash
$ git checkout master
Switched to branch 'master'
$ git checkout -b hotfix
Switched to a new branch 'hotfix'
$ vim index.html
$ git commit -a -m 'Fix broken email address'
[hotfix 1fb7853] Fix broken email address
 1 file changed, 2 insertions(+)
```

핫픽스 검증이 끝났다면 `master`로 돌아가 `hotfix`를 병합한다.

```bash
$ git checkout master
$ git merge hotfix
Updating f42c576..3a0874c
Fast-forward
 index.html | 2 ++
 1 file changed, 2 insertions(+)
```

출력 메시지 속 `"Fast-forward"`가 핵심이다. 현재 `master`가 가리키던 커밋(`C2`)은 `hotfix`가 가리키는 커밋(`C4`)의 **직계 조상**(Direct Ancestor)이다. 즉, `master`에서 분기한 이후 양쪽으로 갈라져 나간 이력이 전혀 없기 때문에, Git은 복잡한 비교 연산이나 새로운 병합 커밋을 생성하지 않고 **단순히 `master` 포인터를 `C4` 앞으로 쑥 당겨(Fast-forward) 이동**시킨다.

이제 `hotfix`는 `master`와 완전히 같은 커밋을 가리키므로 더 이상 필요 없다. `-d` 옵션으로 삭제한 뒤 중단했던 `iss53` 브랜치로 돌아가 작업을 마저 끝낸다(`C5`).

```bash
$ git branch -d hotfix
Deleted branch hotfix (3a0874c).
$ git checkout iss53
Switched to branch "iss53"
$ vim index.html
$ git commit -a -m 'Finish the new footer [issue 53]'
[iss53 ad82d7a] Finish the new footer [issue 53]
 1 file changed, 1 insertion(+)
```

주의할 점은 `iss53` 브랜치의 작업 공간에는 아직 `hotfix`의 수정 사항이 반영되어 있지 않다는 것이다. 필요하다면 이 시점에 `git merge master`로 핫픽스를 먼저 당겨오거나, 아니면 `#53` 작업이 다 끝난 뒤 `master` 쪽으로 병합할 때 한꺼번에 통합할 수 있다.

### 분기된 히스토리의 통합: 3-Way Merge와 Merge Commit

이슈 `#53` 개발이 끝나고 `master`로 돌아가 `iss53` 브랜치를 병합해 보자.

```bash
$ git checkout master
Switched to branch 'master'
$ git merge iss53
Merge made by the 'recursive' strategy.
index.html |    1 +
1 file changed, 1 insertion(+)
```

> **참고 (`recursive` vs `ort`)**: Pro Git 집필 당시 Git의 기본 병합 전략은 `recursive`였으며, Git 2.34부터는 속도와 정확도를 개선한 **`ort`**("Ostensibly Recursive's Twin") 전략이 기본값으로 대체되었다. 3개의 스냅샷을 대조하는 3-Way Merge의 본질은 동일하다.

이번에는 아까의 핫픽스 병합과 출력이 다르다. `master`(`C4`)와 `iss53`(`C5`)이 공통 조상(`C2`)으로부터 **서로 다른 방향으로 분기**(Diverge)했기 때문에 포인터만 앞으로 밀어서는 통합할 수 없다. 이때 Git은 **3-Way Merge**를 수행한다.

1. 두 브랜치가 갈라지기 직전의 **공통 조상(Merge Base) 스냅샷**(`C2`)을 부모 포인터를 거슬러 올라가 자동으로 찾는다.
2. 현재 `master` 끝 스냅샷(`C4`)과 `iss53` 끝 스냅샷(`C5`), 그리고 공통 조상(`C2`) **세 스냅샷을 대조**하여 통합된 새 스냅샷을 만든다.
3. 이 결과를 가리키는 **Merge Commit**(`C6`)을 새로 생성하고 `master` 포인터를 `C6`으로 옮긴다. 이 커밋은 **부모를 2개(`C4`와 `C5`)** 가진다는 점에서 일반 커밋과 구별된다.

병합이 끝났다면 역할을 다한 `iss53` 브랜치를 삭제해 깔끔하게 정리한다.

```bash
$ git branch -d iss53
```

### 병합 충돌(Merge Conflict)의 내부 동작과 해결 절차

만약 `hotfix`와 `iss53` 브랜치가 **같은 파일(`index.html`)의 같은 부분을 서로 다르게 수정**했다면, Git은 어느 쪽을 택해야 할지 판단하지 못하고 병합을 일시 중단(Pause)한다.

```bash
$ git merge iss53
Auto-merging index.html
CONFLICT (content): Merge conflict in index.html
Automatic merge failed; fix conflicts and then commit the result.
```

이 시점에서 Git은 Merge Commit을 생성하지 않고 `.git/MERGE_HEAD` 파일에 상대 브랜치의 커밋 해시를 임시 기록해 둔 채 개발자의 개입을 기다린다. `git status`를 쳐 보면 해결되지 않은 파일이 `Unmerged paths` 아래 `both modified`로 표시된다.

```bash
$ git status
On branch master
You have unmerged paths.
  (fix conflicts and run "git commit")
Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:      index.html
```

충돌이 난 `index.html`을 에디터로 열어 보면 Git이 삽입한 표준 충돌 마커가 보인다.

```html
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
```

- `<<<<<<< HEAD`부터 `=======` 위쪽까지는 **현재 체크아웃된 브랜치(`master`)의 버전**이다.
- `=======` 아래부터 `>>>>>>> iss53`까지는 **병합하려는 브랜치(`iss53`)의 버전**이다.

개발자는 양쪽 중 하나를 선택하거나 두 내용을 적절히 조합해 수정한 뒤, **`<<<<<<<`, `=======`, `>>>>>>>` 마커 줄을 완전히 제거**해야 한다. 예를 들어 두 수정을 조합하면 다음과 같다.

```html
<div id="footer">
please contact us at email.support@github.com
</div>
```

CLI 텍스트 편집 대신 GUI 머지 툴(macOS의 `opendiff`, `meld`, `vimdiff` 등)을 쓰고 싶다면 `git mergetool` 명령으로 시각적 3-way 비교 도구를 띄울 수도 있다. 수정이 끝났다면 **반드시 `git add index.html`을 실행**해 Index에 스테이징해야 Git이 해당 파일의 충돌이 해결되었음을 인식한다.

```bash
$ git add index.html
$ git status
On branch master
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes to be committed:
	modified:   index.html
```

모든 충돌 파일이 스테이징되었다면 `git commit`을 실행해 최종 Merge Commit을 확정한다. 이때 Git은 어떤 브랜치를 병합했고 어떤 파일에서 충돌이 났는지(주석 영역)가 담긴 기본 커밋 메시지를 자동으로 채워 준다.

```text
Merge branch 'iss53'

# Conflicts:
#	index.html
#
# It looks like you may be committing a merge.
...
```

---

## 마치며

이번 편에서는 **Pro Git 3장 전반부**(3.1~3.2)를 통해 Git 브랜치가 왜 그토록 가볍고 빠른지, 그리고 두 브랜치가 통합될 때 내부적으로 어떤 판단이 내려지는지 정리했다.

- 브랜치는 디렉토리 복사가 아니라 커밋 SHA-1 해시를 가리키는 **41바이트 포인터**이며, `HEAD`는 현재 활성화된 브랜치를 가리키는 포인터다.
- 현재 브랜치가 병합 대상의 직계 조상이면 포인터만 이동하는 **Fast-forward**가 일어나고, 이력이 분기했다면 공통 조상(Merge Base)까지 세 스냅샷을 비교해 **부모가 2개인 Merge Commit**을 만드는 **3-Way Merge**가 동작한다.

다음 편(`[Pro Git 톺아보기 #5]`)에서는 Pro Git 3.3~3.5를 바탕으로 실무에서 브랜치 목록을 깔끔하게 관리하는 법(`--merged`, `--no-merged`), Long-Running 및 Topic 브랜치 워크플로우, 그리고 로컬과 서버를 잇는 리모트 트래킹 브랜치의 메커니즘을 다룬다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.1 Git Branching - Branches in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.2 Git Branching - Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
