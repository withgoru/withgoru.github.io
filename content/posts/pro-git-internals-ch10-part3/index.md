+++
date = '2026-12-08T09:00:00+09:00'
draft = false
title = 'Git 데이터 복구 #19: GC·fsck와 환경 변수 총정리'
description = 'Pro Git 10장 3부(10.7~10.8)를 바탕으로 git gc·maintenance, git fsck Dangling 커밋 복구, 대용량 객체 추적과 Git 환경 변수를 정리한다.'
tags = ['Git', 'ProGit', 'GitInternals', 'GitGC', 'GitFsck', 'DataRecovery', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#18)](/posts/pro-git-internals-ch10-part2/)에서는 Packfile의 역방향 델타 압축, Refspec 규칙, 그리고 Smart Protocol의 와이어 협상 원리를 살펴보았다. 이번 **10장 3부(`#19`, Pro Git 10.7~10.8)** 는 **Pro Git 톺아보기 시리즈의 대단원을 마무리하는 최종화**다.

이번 편에서는 저장소를 건강하고 가볍게 유지하는 `git gc` 와 `git maintenance`, **`git reflog` 마저 날아갔을 때 바닥 객체 DB에서 커밋을 건져 올리는 `git fsck` 복구 실습**, 실수로 커밋한 대용량 파일 색출법, 그리고 실무·CI 파이프라인 디버깅에 강력한 위력을 발휘하는 **Git 핵심 환경 변수**를 총정리한다.

---

## 1. 저장소 유지보수와 Garbage Collection (10.7 전반)

Git은 커밋, 리베이스, 체리픽, 스테이징 취소 같은 작업이 일어날 때마다 기존 객체를 지우지 않고 새 객체를 계속 쌓아 둔다. 이 때문에 시간이 지나면 어떤 브랜치나 태그에서도 가리키지 않는 **고립 객체(Unreachable Object)** 와 자잘한 Loose Object가 늘어나는데, 이를 청소하고 압축하는 명령이 **`git gc`** 다.

`git gc`를 실행하면 내부적으로 다음 네 가지 작업이 한 번에 일어난다.

1. **Loose Object 팩킹**: 흩어진 개별 객체 파일들을 `.pack` + `.idx` 파일로 압축 통합한다(`git repack`).
2. **참조 압축 (`pack-refs`)**: 흩어진 브랜치·태그 Ref 파일을 `.git/packed-refs` 로 모아 조회 속도를 높인다.
3. **만료된 Reflog 정리**: 기본 90일이 지난 `unreachable` Reflog 엔트리(도달 가능한 이력은 90일, 도달 불가능한 고립 이력은 30일)를 제거한다.
4. **고립 객체 영구 삭제 (`prune`)**: 어떤 Ref나 Reflog에서도 참조하지 않는 Unreachable 객체 중 유예 기간(`gc.pruneExpire`, **기본값 2주**)이 지난 객체를 디스크에서 완전히 지운다. *(현대 Git에서는 고립 객체를 수많은 Loose 파일로 방치하지 않고 수정 시각(`.mtimes`)을 함께 기록한 **Cruft Pack**으로 묶어 관리하며, **Git 2.50(2025.06)** 에서는 다중 Cruft Pack 관리 효율과 도구가 크게 개선되었다.)*

### `git gc --auto` vs 현대적 백그라운드 유지보수 `git maintenance`

평소 Git은 `git commit`이나 `git merge` 직후 내부적으로 `git gc --auto`를 조용히 체크한다. Loose Object가 **6,700개(`gc.auto`)** 를 넘거나 Packfile이 **50개(`gc.autoPackLimit`)** 이상 쌓이면 자동으로 GC를 돌린다.

하지만 수 기가바이트 규모의 대규모 저장소에서는 작업 중간에 갑자기 `git gc`가 터지면 개발자가 몇 분씩 터미널 앞에서 멈춰 기다려야 하는 불편이 컸다. 이를 해결하기 위해 현대 Git(`2.30+`)은 **`git maintenance start`** 를 제공한다.

```bash
# 현재 저장소에 대해 백그라운드 자동 유지보수 스케줄러 등록
# (macOS는 launchd, Linux는 systemd/cron, Windows는 작업 스케줄러에 자동 등록)
$ git maintenance start
```

`git maintenance`는 개발자가 커밋을 치는 순간을 방해하지 않고 백그라운드에서 **매시간** `prefetch`(원격 미리 가져오기)·`loose-objects`·`incremental-repack`을, **매일** `commit-graph` 갱신·`pack-refs`를 나누어 실행해 대규모 모노레포에서도 쾌적한 반응 속도를 유지해 준다. 특히 **Git 2.54(2026.04)** 부터는 `git maintenance run`의 기본 리패킹 전략이 크기 구간별로 Pack을 계층 통합하는 **Geometric Repacking**으로 전환되었고, 유지보수 실행 필요 여부를 사전 점검하는 **`git maintenance is-needed`** 서브커맨드도 추가되었다.

---

## 2. [터미널 실습] Reflog마저 없을 때 `git fsck`로 커밋 복구하기 (10.7 중반)

[7장 1부(#11)](/posts/pro-git-internals-ch7-part1/)에서 실수로 `git reset --hard`를 실행했을 때의 1차 구명줄로 **`git reflog`** 를 소개했다. 그렇다면 만약 누군가 `git reflog expire`를 돌렸거나, 90일 시효가 만료되었거나, 혹은 `Detached HEAD` 상태에서 실수로 브랜치를 잃어버려 **Reflog에서조차 커밋이 보이지 않을 때**는 정말 끝일까?

**아니다. 앞서 본 `gc.pruneExpire`(기본 2주) 유예 기간이 지나 `prune`되기 전까지 실제 Commit·Tree·Blob 객체는 여전히 `.git/objects/` 바닥에 온전히 살아 있다.** 이때 최후의 구명줄 역할을 하는 명령이 바로 파일시스템 정합성 검사기 **`git fsck`** 다.

{{< figure src="gc-fsck-recovery-and-env.svg" alt="Git GC, fsck 기반 Dangling 커밋 복구 경로 및 핵심 환경 변수 3개 영역 다이어그램" caption="Reachable vs Dangling 객체 그래프와 git fsck --full 복구 수명 주기(상단) 및 실무·CI 디버깅용 Git 환경 변수 3대 영역(하단) (Pro Git 10.7~10.8 재구성)" width="820" height="560" >}}

직접 터미널에서 `git reset --hard` 뒤 Reflog까지 일부러 싹 비운 극한 상황을 만들고, `git fsck`로 잃어버린 커밋을 살려 보자.

```bash
$ mkdir fsck-lab && cd fsck-lab && git init -b main
$ echo "init" > app.txt && git add app.txt && git commit -m "init app"
$ echo "lost feature" > feature.txt && git add feature.txt && git commit -m "feat: lost work"

# 1. 실수로 직전 커밋(feat: lost work)을 하드 리셋으로 날려 버림
$ git reset --hard HEAD~1
HEAD is now at 97fce17 init app

# 2. 설상가상으로 Reflog 기록까지 전부 만료시켜 1차 복구 수단 봉쇄
$ git reflog expire --expire=now --all

# 3. git fsck --full --no-reflogs 로 고립된(dangling) 객체 전수 탐색
$ git fsck --full --no-reflogs
Checking ref database: 100% (1/1), done.
Checking object directories: 100% (256/256), done.
dangling commit b705156cdaf28526d5c50df92e13374b1dfa32bb

# 4. 찾아낸 dangling commit의 변경 내용 검증
$ git show --stat b705156cdaf28526d5c50df92e13374b1dfa32bb
commit b705156cdaf28526d5c50df92e13374b1dfa32bb
Author: withgoru <fromgoru@gmail.com>
Date:   Tue Nov 24 13:52:53 2026 +0900

    feat: lost work

 feature.txt | 1 +
 1 file changed, 1 insertion(+)

# 5. 새 브랜치(recover)로 포인터만 다시 매달아 주면 100% 복구 완료!
$ git branch recover b705156cdaf28526d5c50df92e13374b1dfa32bb
$ git log --oneline --all
97fce17 (HEAD -> main) init app
b705156 (recover) feat: lost work
```

`--no-reflogs` 플래그를 주어야 Reflog가 가리키던 객체까지 "진짜 아무 Ref도 연결되지 않은 Dangling 상태"로 드러난다. 반대로 커밋조차 하기 전 `git add`만 했다가 잃어버린 파일이라면 `dangling blob`으로 뜨므로 `git cat-file -p <해시>` 로 내용을 확인해 되살릴 수 있다.

---

## 3. 저장소를 비대하게 만든 대용량 객체 추적과 제거 (10.7 후반)

누군가 과거에 500MB짜리 동영상이나 데이터베이스 덤프를 실수로 커밋했다가 다음 커밋에서 `git rm`으로 지웠더라도, Git의 모든 커밋은 스냅샷이므로 `.git/objects/` 안에는 여전히 그 500MB짜리 Blob이 영원히 남는다.

원인 Blob을 빠르게 색출하고 용량을 회수하는 절차는 2단계로 이루어진다.

```bash
# 1단계: verify-pack으로 Packfile 내 원본 크기(3번째 컬럼)가 가장 큰 상위 5개 객체 추출
$ git verify-pack -v .git/objects/pack/pack-*.idx \
  | sort -k 3 -n \
  | tail -5

# 2단계: 찾아낸 거대 Blob 해시가 어느 커밋의 어떤 파일 경로인지 역추적
$ git rev-list --objects --all | grep <찾아낸_Blob_해시>
```

범인 파일 경로를 특정했다면 이제 히스토리에서 지워야 한다. Pro Git 2판 원문(10.7)이 출간되던 시절에는 `git filter-branch`를 썼지만, 오늘날 Git 공식 문서는 속도·안전성 문제로 이를 **Deprecated** 처리하고 **`git-filter-repo`**([7장 2부(#12)](/posts/pro-git-internals-ch7-part2/) 참조)를 표준으로 권장한다.

```bash
# git-filter-repo로 해당 대용량 경로를 전체 히스토리에서 영구 제거
$ git filter-repo --path assets/huge-dump.sql --invert-paths

# 필터링 후 남은 Reflog와 백업 Ref를 즉시 만료시키고 디스크 용량 강제 회수
$ git reflog expire --expire=now --all
$ git gc --prune=now --aggressive
```

---

## 4. 실무 · CI 디버깅용 Git 환경 변수 총정리 (10.8)

Git은 수많은 동작을 명령줄 옵션이나 `.gitconfig` 외에 **환경 변수(Environment Variables)** 로 세밀하게 제어할 수 있다. 특히 CI/CD 러너(Runner), 자동화 스크립트, 네트워크 장애 트러블슈팅에서 환경 변수는 설정 파일을 오염시키지 않고 즉석에서 Git의 동작을 바꾸는 가장 깔끔한 도구다.

| 영역 | 주요 환경 변수 | 실무 · 디버깅 활용 사례 |
| :--- | :--- | :--- |
| **저장소 경로 제어** | `GIT_DIR`, `GIT_WORK_TREE` | `.git` 폴더와 작업 트리를 분리(예: Bare 저장소로 `$HOME`의 dotfiles 관리) |
| **인덱스 · 객체 분리** | `GIT_INDEX_FILE`, `GIT_OBJECT_DIRECTORY` | 실제 Staging Area(`.git/index`)를 건드리지 않고 임시 Index로 빌드·커밋 스크립트 실행 |
| **메타데이터 주입** | `GIT_AUTHOR_NAME/EMAIL/DATE`, `GIT_COMMITTER_*` | CI 봇 커밋 생성, 혹은 타임스탬프를 고정해 결정론적 해시 재현(이번 `#17` 실습에서도 활용!) |
| **무인 환경 인증** | `GIT_SSH_COMMAND`, `GIT_TERMINAL_PROMPT=0` | CI에서 특정 배포 키 강제 지정(`ssh -i /key`), 비밀번호 입력 대기로 행(Hang) 걸림 방지 |
| **트레이싱 · 프로파일링** | `GIT_TRACE=1`, `GIT_TRACE_PACKET=1`, `GIT_TRACE2_PERF=1` | 내부 실행 명령, **Smart Protocol `want`/`have` 패킷**, 구간별 실행 시간(ms) 실시간 추적 |

예를 들어 10장 2부(`#18`)에서 배운 Smart Protocol의 `want`/`have` 패킷 교환이나 SSH/HTTP 인증 실패 원인을 눈으로 확인하고 싶을 때는 아래 한 줄이면 충분하다.

```bash
# 네트워크 와이어 패킷과 내부 서브프로세스 실행 로그를 동시에 확인하기
$ GIT_TRACE=1 GIT_TRACE_PACKET=1 git ls-remote origin
```

---

## 5. Pro Git 톺아보기 시리즈를 마치며 (`#1` ~ `#19` 완결 회고)

이로써 **Pro Git 2판을 바닥부터 끝까지 해부한 19편의 여정(번외 2편 포함 총 21편)** 을 모두 마친다.

처음 1장에서 "Git은 파일의 차이(Delta)가 아니라 스냅샷을 저장한다"는 문장으로 시작했던 개념이, 10장 마지막에 이르러 **Blob·Tree·Commit의 바이트 포맷과 Packfile의 역방향 델타 압축, Smart Protocol의 `want`/`have` 협상, 그리고 `git fsck` 복구**라는 구체적인 엔지니어링 실체로 하나로 맞물렸다.

| 구간 | 핵심 주제 | 시리즈 글 목록 |
| :--- | :--- | :--- |
| **1장 · 기초** | 스냅샷 모델과 3가지 상태·영역 | [`#1` 스냅샷 모델과 3가지 영역](/posts/pro-git-internals-ch1/) |
| **2~3장 · 일상과 브랜치** | diff·reset, fetch vs pull, 41B 포인터, Rebase, 전략과 실습 | [`#2`](/posts/pro-git-internals-ch2-part1/) · [`#3`](/posts/pro-git-internals-ch2-part2/) · [`#4`](/posts/pro-git-internals-ch3-part1/) · [`#5`](/posts/pro-git-internals-ch3-part2/) · [`#6`](/posts/pro-git-internals-ch3-part3/) · [브랜칭 전략](/posts/pro-git-ch3-branching-strategies/) · [터미널 실습](/posts/pro-git-hands-on-ch1-to-ch3/) |
| **4~6장 · 협업과 서버** | Bare 저장소·HTTPS vs SSH, 워크플로우, `A...B`·cherry-pick, 호스팅 플랫폼 | [`#7`](/posts/pro-git-internals-ch4/) · [`#8`](/posts/pro-git-internals-ch5-part1/) · [`#9`](/posts/pro-git-internals-ch5-part2/) · [`#10`](/posts/pro-git-internals-ch6/) |
| **7~8장 · 실전 도구와 커스터마이징** | Reflog·Stash, 3-Tree Reset, `bisect`·`blame`, Submodule, `.gitattributes`, Git Hooks | [`#11`](/posts/pro-git-internals-ch7-part1/) · [`#12`](/posts/pro-git-internals-ch7-part2/) · [`#13`](/posts/pro-git-internals-ch7-part3/) · [`#14`](/posts/pro-git-internals-ch7-part4/) · [`#15`](/posts/pro-git-internals-ch8-part1/) · [`#16`](/posts/pro-git-internals-ch8-part2/) |
| **10장 · Git 내부 구조** | Blob·Tree·Commit·Refs, Packfile·Refspec·프로토콜, GC·`fsck`·환경 변수 | [`#17`](/posts/pro-git-internals-ch10-part1/) · [`#18`](/posts/pro-git-internals-ch10-part2/) · **`#19` (현재 글)** |

Git 명령어가 어딘가 마법처럼 느껴지거나 충돌·유실 앞에서 막막할 때마다, `.git/` 아래의 **단순하고 우아한 4가지 불변 객체와 가벼운 포인터(Ref)** 를 떠올려 보자. 원리를 알고 다루는 도구는 더 이상 두려움의 대상이 아니라 가장 든든한 엔지니어링 무기가 된다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.7 Maintenance and Data Recovery](https://git-scm.com/book/en/v2/Git-Internals-Maintenance-and-Data-Recovery)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.8 Environment Variables](https://git-scm.com/book/en/v2/Git-Internals-Environment-Variables)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.9 Summary](https://git-scm.com/book/en/v2/Git-Internals-Summary)
- [Git Official Documentation - git-gc(1), git-maintenance(1) & git-fsck(1)](https://git-scm.com/docs/git-maintenance)
- [Git Official Documentation - git(1) Environment Variables](https://git-scm.com/docs/git#_environment_variables)
