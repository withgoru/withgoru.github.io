+++
date = '2026-12-01T09:00:00+09:00'
draft = false
title = 'Git Reset 원리 #12: 3가지 트리와 Rebase -i'
description = 'Pro Git 7장 2부(7.6~7.7)를 바탕으로 3가지 트리 관점의 reset(--soft/--mixed/--hard) 원리와 rebase -i, filter-repo를 정리한다.'
tags = ['Git', 'ProGit', 'GitReset', 'Rebase', 'GitTools', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#11)](/posts/pro-git-internals-ch7-part1/)에서는 `reflog`, `add -p`, `stash`, `log -S/-G/-L` 등 일상 생산성을 높이는 도구들을 살펴보았다. 이번 **7장 2부(`#12`, Pro Git 7.6~7.7)** 에서는 개발자들이 가장 두려워하면서도 가장 자주 쓰는 두 축인 **`git reset`의 내부 원리**와 **`git rebase -i` · `git filter-repo` 히스토리 재작성**을 완전 해부한다.

---

## 1. Git의 3가지 트리와 `git reset` 동작 원리 완전 해부 (7.7)

`git reset`을 명령어 옵션 암기가 아니라 원리로 이해하려면, Git이 파일 스냅샷을 관리하는 **3가지 트리(The Three Trees)** 개념만 정확히 잡으면 된다(여기서 트리란 폴더 구조와 파일 스냅샷 모음을 뜻한다).

1. **`HEAD` (마지막 커밋 스냅샷)**: 현재 체크아웃된 브랜치의 끝을 가리키는 포인터이자, 다음 커밋의 부모가 된다.
2. **`Index` (Staging Area, 다음 커밋 후보)**: `git add`로 올려 둔 다음 커밋의 초안 스냅샷이다.
3. **`Working Tree` (실제 작업 디렉토리)**: 우리가 에디터로 열어 직접 수정하는 디스크 위의 실제 파일들이다.

{{< figure src="reset-three-trees.svg" alt="Git의 3가지 트리(HEAD, Index, Working Tree)와 reset --soft, --mixed, --hard 도달 범위 및 --soft Squash 워크플로우 다이어그램" caption="Git 3가지 트리 관점의 git reset 3단계 도달 범위(상단)와 파일 지정 reset vs restore 차이 및 --soft 커밋 압축 흐름(하단) (Pro Git 7.7 재구성)" width="820" height="540" >}}

### `git reset`의 순차적 3단계: `--soft`, `--mixed`, `--hard`

`git reset <커밋>`을 실행하면 Git은 항상 **1단계부터 시작해 옵션이 지정한 단계까지만 진행하고 멈춘다.**

| 옵션 | 도달 단계 | 동작 내용 및 상태 변화 | 대표 활용 시나리오 |
| :--- | :--- | :--- | :--- |
| **`--soft`** | **Step 1 정지** | **`HEAD` 포인터(브랜치)만 지정 커밋으로 이동**한다. `Index`와 `Working Tree`는 전혀 건드리지 않으므로, 되돌린 커밋들의 변경분이 그대로 Staged 상태로 남는다. | 직전 커밋 메시지/구성을 고치거나 여러 커밋을 하나로 **Squash**할 때 |
| **`--mixed`** *(기본값)* | **Step 1 → 2 정지** | `HEAD`를 이동한 뒤(Step 1), **`Index`까지 이동한 `HEAD`의 스냅샷으로 되돌린다**(Step 2). 실제 파일 수정분은 `Working Tree`에 남는다. | 옵션을 생략했을 때의 기본 동작 · 스테이징을 해제하고 변경분을 다시 검토할 때 |
| **`--hard`** | **Step 1 → 2 → 3 진행** | `HEAD` 이동(Step 1), `Index` 초기화(Step 2), 그리고 **`Working Tree`의 실제 파일까지 강제로 덮어쓴다**(Step 3). | 작업 중인 변경을 완전히 폐기할 때 *(커밋 안 된 수정 내용은 복구 불가!)* |

핵심 주의점이 하나 있다. `--soft`나 `--mixed`로 되돌린 커밋은 여전히 `.git/objects`와 `git reflog`에 살아 있지만, **`--hard` 실행 시 아직 `git commit`을 한 번도 하지 않은 Working Tree 수정 사항은 Reflog에도 남지 않아 영구히 사라진다.**

### 경로(`<file>`)를 지정했을 때: `reset <file>` vs `restore` / `checkout`

`git reset HEAD file.txt`처럼 **커밋 뒤에 특정 파일 경로를 붙이면**, Git은 **Step 1(`HEAD` 포인터 이동)을 건너뛰고** 해당 파일의 `Index`만 지정 커밋 스냅샷으로 복원한다(Step 2). 브랜치 자체를 되돌리지 않고 특정 파일만 언스테이징하는 가장 안전한 방법이다.

오늘날 Git `2.23+`에서는 역할 혼동을 막기 위해 `checkout`을 **`switch`(브랜치 이동)** 와 **`restore`(파일 복원)** 로 분리했다.

```bash
# 1. Index만 되돌리기 (안전한 언스테이징 · 구 git reset HEAD file.txt와 동일)
$ git restore --staged file.txt

# 2. Working Tree의 수정 내역까지 버리고 복원 (위험! · 구 git checkout -- file.txt와 동일)
$ git restore file.txt
```

### 실전 팁: `git reset --soft`로 커밋 3개 초간단 Squash 하기

`rebase -i` 창을 열지 않고도 직전 커밋 N개를 1개로 압축할 때 `--soft`가 아주 요긴하다.

```bash
# C1 <- C2(WIP) <- C3(오타) <- C4(테스트) 상태에서 HEAD를 C1으로만 이동
$ git reset --soft HEAD~3

# C2~C4의 모든 변경분이 Index(Staged)에 고스란히 대기 중이므로 바로 1개 커밋으로 생성
$ git commit -m "feat(auth): implement JWT rotation"
```

---

## 2. 정교한 히스토리 재작성: `--amend`와 `git rebase -i` (7.6)

### 직전 커밋 1개 보완하기: `git commit --amend`

직전 커밋 메시지에 오타가 있거나 파일 하나를 빼먹고 커밋했다면 `--amend`로 바로 덮어쓸 수 있다. 메시지를 바꾸지 않고 빠진 파일만 추가하고 싶다면 `--no-edit`를 붙인다.

```bash
$ git add forgotten_file.go
$ git commit --amend --no-edit
```

### 여러 커밋을 자유자재로 조립하기: `git rebase -i`

과거 여러 개의 커밋을 한꺼번에 합치거나(`squash`/`fixup`), 순서를 바꾸거나, 중간 커밋 하나를 둘로 쪼개고 싶다면 대화형 리베이스(**Interactive Rebase**)를 연다.

```bash
$ git rebase -i HEAD~4
```

{{< figure src="interactive-rebase-and-filter-repo.svg" alt="git rebase -i 대화형 리베이스 편집 스크립트 6대 명령어와 구식 filter-branch 대비 현대 git filter-repo 비교 다이어그램" caption="git rebase -i의 6대 편집 명령(상단)과 filter-branch 폐기 및 현대 표준 git filter-repo 비교(하단) (Pro Git 7.6 재구성)" width="820" height="520" >}}

에디터가 열리면 **`git log`와 반대로 가장 오래된 커밋이 맨 위, 최신 커밋이 맨 아래**에 나열된다. 각 커밋 앞의 명령어를 바꿔 저장하면 Git이 맨 위 커밋부터 순서대로 재적용한다.

- **`pick` (`p`)**: 커밋을 변경 없이 그대로 유지한다.
- **`reword` (`r`)**: 코드 변경분은 유지한 채 커밋 메시지만 다시 작성한다.
- **`squash` (`s`) / `fixup` (`f`)**: 현재 커밋을 **바로 위 커밋**에 합친다. `s`는 두 커밋 메시지를 합쳐 편집할 기회를 주고, `f`는 아래 커밋의 메시지(`WIP`, `오타 수정`)를 깔끔히 버린다.
- **`edit` (`e`)**: 해당 커밋을 적용한 뒤 리베이스를 일시 정지한다. 여기서 `git reset HEAD^`로 커밋을 푼 뒤 `git add -p`를 이용해 하나의 큰 커밋을 **논리적으로 깔끔한 2개 이상의 원자적 커밋으로 쪼갤 수 있다.** 그 뒤 `git rebase --continue`로 재개한다.
- **`drop` (`d`) / 줄 이동**: 특정 커밋 줄을 지우거나 `d`로 바꾸면 커밋이 완전히 삭제되고, 텍스트 줄의 위아래 순서를 바꾸면 커밋 순서도 재배치된다.

물론 이 모든 작업은 커밋의 SHA-1 해시를 새로 굽는 행위이므로 [Rebase Golden Rule]({{< ref "/posts/pro-git-internals-ch3-part3" >}})에 따라 **원격 공유 브랜치에 Push하기 전 로컬 Topic 브랜치에서만 실행**해야 한다.

---

## 3. 전체 이력 영구 정화: `filter-branch` 폐기와 `git filter-repo` (7.6 현대화)

실수로 API 키나 비밀번호가 담긴 `.env` 파일, 혹은 수백 MB짜리 빌드 바이너리를 커밋해 수십 개의 커밋을 지나쳐 버렸다면, 최신 커밋에서 `git rm`을 해도 과거 히스토리 안에 그 파일이 영원히 남아 저장소 용량과 보안 사고 위험을 키운다.

과거 Pro Git 2판에서는 이 문제를 해결하기 위해 `git filter-branch`를 소개했다. 하지만 `filter-branch`는 모든 커밋마다 쉘 서브프로세스를 띄워 극도로 느리고, `refs/original/` 백업 참조가 남으며, 잘못 사용하면 저장소를 망가뜨리는 문제가 많아 **Git 공식 문서 자체에서 사용 중단(Deprecated)과 `git filter-repo`로의 교체를 권고**하고 있다.

```bash
# 1. 실수로 커밋한 대용량 파일/비밀 파일을 전체 히스토리에서 초고속 영구 삭제
$ git filter-repo --path .env.prod --invert-paths

# 2. 유출된 API 키 문자열을 전 커밋 이력에 걸쳐 일괄 마스킹 (expressions.txt: SECRET==>***)
$ git filter-repo --replace-text expressions.txt

# 3. 특정 서브폴더만 떼어내 별도 독립 저장소로 분리
$ git filter-repo --subdirectory-filter packages/auth
```

`git filter-repo`는 Git의 `fast-export`/`fast-import` 스트림을 Python으로 직접 처리하므로 기존 `filter-branch` 대비 **수십 배에서 수백 배 빠르게 동작**하며, 실행 후 안전하게 `origin` 원격 연결을 해제해 실수로 덮어쓰기 푸시하는 사고까지 막아 준다(정화 완료 후 팀 합의 하에 `git remote add origin` 및 `git push --force` 진행).

---

## 마치며

이번 **7장 2부(`#12`)** 를 통해 `git reset`이 마법의 되돌리기 버튼이 아니라 **`HEAD`, `Index`, `Working Tree` 중 어디까지를 갱신할지 결정하는 3단계 포인터 연산**이라는 점을 명확히 잡았다. 여기에 `git rebase -i`와 `git filter-repo`까지 손에 익히면, 실수로 엉킨 로컬 이력이나 보안 파일 유출 사고 앞에서도 저장소 내부 구조를 망가뜨리지 않고 깔끔하게 정돈할 수 있다.

이어지는 [7장 3부(#13)](/posts/pro-git-internals-ch7-part3/)에서는 **`merge.conflictstyle = zdiff3`** 을 활용한 정밀 충돌 분석, **머지 커밋을 `git revert -m 1`로 되돌릴 때 빠지기 쉬운 치명적 함정**, 그리고 버그가 처음 심어진 커밋을 $O(\log N)$ 만에 찾아내는 **`git bisect` 자동화 디버깅**을 다룬다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 7.6 Rewriting History](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 7.7 Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified)
- [newren/git-filter-repo Official Repository](https://github.com/newren/git-filter-repo)
