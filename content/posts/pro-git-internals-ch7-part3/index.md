+++
date = '2026-12-02T09:00:00+09:00'
draft = false
title = 'Git 디버깅과 병합 #13: bisect·blame·revert -m'
description = 'Pro Git 7장 3부(7.8~7.10)를 바탕으로 zdiff3 충돌, revert -m 1 재병합 함정, git blame -C, bisect 자동 디버깅을 정리한다.'
tags = ['Git', 'ProGit', 'GitBisect', 'GitBlame', 'Merge', 'Debugging', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#12)](/posts/pro-git-internals-ch7-part2/)에서는 3가지 트리 관점에서의 `git reset` 원리와 `rebase -i`, `git filter-repo`를 살펴보았다. 이번 **7장 3부(`#13`, Pro Git 7.8~7.10)** 에서는 복잡한 충돌을 현명하게 풀어내는 **고급 병합 기법**, 머지 커밋을 되돌릴 때 발생하는 **치명적 재병합 함정**, 그리고 코드의 작성 맥락과 버그 유입 커밋을 추적하는 **`git blame` · `git bisect` 디버깅 기술**을 정리한다.

---

## 1. 고급 병합(Advanced Merging)과 `zdiff3` 충돌 해결 (7.8)

### 충돌 해결의 판도를 바꾸는 `merge.conflictstyle = zdiff3`

기본 Git 설정에서 병합 충돌이 나면 현재 브랜치(`HEAD`)와 상대 브랜치 두 블록만 보여 준다. 이 상태에서는 양쪽 중 어느 쪽 변경을 살려야 할지, 혹은 두 변경을 어떻게 합쳐야 할지 원문을 몰라 감으로 찍는 실수가 잦다.

반면 **`zdiff3`**(Zealous Diff3) 스타일을 켜면 분기 전 **공통 조상(Base)의 원래 코드까지 함께 출력**하고 앞뒤의 겹치는 줄은 깔끔히 축약해 준다.

```bash
# 한 번 설정해 두면 모든 프로젝트에서 Base 원문까지 포함된 3단 충돌 표시
$ git config --global merge.conflictstyle zdiff3
```

```text
<<<<<<< HEAD
timeout_ms = 5000
||||||| merged common ancestors
timeout_ms = 3000
=======
timeout_ms = 8000
>>>>>>> feature/retry
```

원본이 `3000`이었다는 사실을 눈으로 보는 순간, `HEAD`는 `+2000`을 더했고 `feature/retry`는 `+5000`을 늘렸다는 맥락이 한눈에 들어온다. Git 2.35+에서 도입된 `zdiff3`는 기존 `diff3`보다 공통 앞뒤 라인까지 깔끔하게 정리해 주므로 오늘날 사실상의 필수 글로벌 설정으로 꼽힌다.

### 충돌 자동 해법(`-Xours`/`-Xtheirs`)과 전격 포기(`ours`)

- **`git merge -Xours` / `-Xtheirs`**: 충돌이 발생한 지점에 한해 우리 쪽(`ours`) 또는 상대 쪽(`theirs`)을 기계적으로 채택하고, 충돌 나지 않는 나머지 변경은 정상 병합한다.
- **`git merge -s ours <브랜치>`**: 내용과 무관하게 **상대 브랜치의 변경 사항은 0줄도 반영하지 않고, 오직 "병합했다"는 히스토리 포인터 기록만 남길 때** 사용한다(예: 더 이상 유지하지 않을 레거시 브랜치를 이력에 합쳤다는 표시만 남길 때). `-Xours`와 `-s ours`는 완전히 다르므로 혼동하면 안 된다.

이 밖에도 병합 충돌 중에 `git log --merge -p`를 치면 **현재 충돌에 관여한 커밋들의 변경 내역만 필터링**해 볼 수 있으며, 반복되는 충돌 해결을 기억시키는 `git rerere`(Pro Git 7.9, [5장 2부(#9)]({{< ref "/posts/pro-git-internals-ch5-part2" >}}) 참고)를 켜 두면 수동 작업을 크게 줄일 수 있다.

---

## 2. 머지 커밋 되돌리기(`git revert -m 1`)와 재병합의 함정 (7.8)

{{< figure src="advanced-merge-and-bisect.svg" alt="zdiff3 3단 충돌 구조, git revert -m 1 재병합 함정, 그리고 git bisect O(log N) 이진 탐색 자동화 다이어그램" caption="zdiff3 충돌 맥락 복원과 revert -m 1 재병합 함정(상단) 및 git bisect / bisect run 이진 탐색 디버깅 흐름(하단) (Pro Git 7.8~7.10 재구성)" width="820" height="560" >}}

`main` 브랜치에 잘못된 `feature` 브랜치를 병합(`M`)한 뒤 이미 원격에 푸시까지 끝났다면 `reset` 대신 새로운 취소 커밋을 쌓는 `git revert`를 써야 한다. 그런데 일반 커밋과 달리 **Merge 커밋은 부모가 2개(`C1`과 `F1`)이므로 어느 쪽을 본류(Mainline)로 보고 되돌릴지 `-m 1` 번호를 명시**해야 한다.

```bash
# 1번 부모(병합 전 main 상태인 C1)를 기준으로 머지 커밋 M의 변경분을 전부 취소하는 ^M 생성
$ git revert -m 1 <Merge커밋해시>
```

**여기서 실무자가 가장 많이 빠지는 대형 함정이 시작된다.** 문제였던 `feature` 브랜치로 돌아가 버그만 살짝 고쳐(`F2`) 다시 `git merge feature`를 실행하면 어떻게 될까?

`feature`의 이전 커밋 `F1`은 이미 `M`의 조상 그래프에 포함되어 있으므로, Git은 `F1`의 변경 사항은 이미 `main`에 들어간 것으로 간주하고 **새로 고친 `F2`만 반영한 채 `F1`의 기능은 모조리 증발시켜 버린다!** 이 함정에서 빠져나오는 정석 순서는 두 가지 중 하나다.

1. **Revert의 Revert (`^^M`)**: `main` 위에서 아까 만든 취소 커밋 `^M`을 다시 `git revert`해 `F1` 코드를 먼저 복원한 다음 `feature`의 추가 수정분(`F2`)을 머지한다.
2. **Topic 재작성**: 혹은 `feature` 브랜치를 새로운 기반 커밋 위로 재커밋해 새로운 커밋 해시 집합으로 다시 통합한다.

---

## 3. 코드의 범인과 역사 추적: `git blame`과 `git bisect` (7.10)

### ① 줄 단위 작성자와 리팩토링 이동 추적: `git blame`

파일의 특정 줄을 누가 언제 왜 고쳤는지 확인할 때는 `git blame`을 사용한다. 특히 다른 파일이나 함수에서 코드를 복사·이동해 온 리팩토링 흔적까지 파헤치려면 **`-C`** 옵션을 붙여야 한다.

```bash
# 1. 파일 내 69~82번 줄의 마지막 수정 커밋과 작성자 조회
$ git blame -L 69,82 Makefile

# 2. 코드가 다른 파일로 잘라내어 옮겨지기(Copy/Move) 전 원작 커밋까지 추적
$ git blame -C -L 141,153 auth_service.go
```

> **최신 Git 참고 (`git last-modified` · `blame --diff-algorithm`)**: 개별 줄이 아니라 "특정 폴더 안의 각 파일이 마지막으로 언제 바뀌었는가(트리 단위 Blame)"만 빠르게 보고 싶다면 **Git 2.52(2025.11)** 에 도입된 **`git last-modified`** 가 기존 방식 대비 약 5배 빠르다. 또한 **Git 2.54(2026.04)** 부터는 `git blame --diff-algorithm=histogram` 처럼 diff 알고리즘을 직접 지정해 코드 이동 추적 정확도를 높일 수 있다.

### ② 1,000개 커밋도 10번 만에 잡는 이진 탐색 디버깅: `git bisect`

"지난주 `v1.0` 태그에서는 분명히 정상 작동했는데 오늘 `HEAD`에서는 테스트가 깨진다"는 장애를 만났을 때, 수백 개의 커밋을 하나씩 체크아웃해 보는 대신 **`git bisect`** 를 쓰면 `O(log N)` 횟수만으로 정확히 **버그를 처음 심은 단 1개의 커밋**을 찾아낸다. (`1,024`개 커밋도 `2^10`이므로 최대 `10`번 검사면 끝난다.)

```bash
# 1. 이진 탐색 시작 (현재 HEAD는 버그 있음=bad, v1.0 태그는 정상=good)
$ git bisect start HEAD v1.0
Bisecting: 512 revisions left to test after this step (roughly 9 steps)

# 2. Git이 자동으로 중간 지점 커밋으로 체크아웃해 주면 테스트 후 판정 입력
$ git bisect good   # 이 커밋은 정상이면 good, 깨졌으면 git bisect bad

# 3. 탐색 종료 및 원래 작업 브랜치 HEAD로 복귀 (필수!)
$ git bisect reset
```

### 따라 해 보는 `git bisect run` 완전 자동화 실습

매번 손으로 `good` / `bad`를 입력하는 대신 테스트 스크립트만 넘겨주면 커피를 마시고 돌아오는 사이에 Git이 알아서 범인 커밋을 찾아 놓는다.

```bash
# 테스트 저장소 생성 (1~20번 커밋 중 13번 커밋에서 버그가 유입된 상황 시뮬레이션)
$ mkdir bisect-lab && cd bisect-lab && git init -b main
$ for i in $(seq 1 20); do
    if [ "$i" -ge 13 ]; then echo "BUG" > status.txt; else echo "OK" > status.txt; fi
    git add status.txt && git commit -m "Commit #$i"
  done

# 검증 스크립트: OK면 exit 0 (good), BUG면 exit 1 (bad)
$ git bisect start HEAD HEAD~19
$ git bisect run sh -c 'grep -q "^OK$" status.txt'
...
<해시> is the first bad commit
commit <해시>
    Commit #13

# 탐색 종료 후 원래 위치로 복귀
$ git bisect reset
```

참고로 만약 탐색 도중 컴파일 자체가 안 되거나 테스트와 무관한 이유로 실행 불가능한 커밋을 만나면 스크립트가 **`exit 125`** 를 반환하게 작성하면 된다. 그러면 `git bisect run`은 해당 커밋을 판단 대상에서 영리하게 건너뛰고(`git bisect skip`) 인접한 커밋으로 탐색을 이어 간다.

---

## 마치며

이번 **7장 3부(`#13`)** 를 통해 충돌 맥락을 살리는 `zdiff3`, 머지 되돌리기의 재병합 함정(`revert -m 1` 후 `revert ^M`), 파일 이동까지 꿰뚫는 `git blame -C`, 그리고 장애 원인을 `O(log N)`으로 색출하는 `git bisect run`까지 중대형 프로젝트 유지보수에 꼭 필요한 무기들을 정리했다.

마지막 [7장 4부(#14)](/posts/pro-git-internals-ch7-part4/)에서는 많은 개발자가 실무에서 애를 먹는 **`git submodule`의 Gitlink 내부 원리와 함정**, 현대적 대안(**Subtree vs Monorepo**), 그리고 오프라인 배포 **`git bundle`** 과 이력 접합 **`git replace`** 로 Pro Git 7장을 완결짓는다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 7.8~7.10 Git Tools](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging)
- [Git Documentation - How to revert a faulty merge](https://git-scm.com/docs/git-revert)
- [Git Documentation - git-bisect](https://git-scm.com/docs/git-bisect)
