+++
date = '2026-11-27T09:00:00+09:00'
draft = false
title = '[Pro Git 톺아보기 #8] 중앙 집중형(CVCS) vs 분산형(DVCS) 아키텍처와 3대 분산 워크플로우, 그리고 프로젝트 기여의 정석'
description = 'Pro Git 5장 전반부(5.1~5.2)를 바탕으로 중앙 집중형 저장소와 분산형 저장소의 구조적 차이, 3대 분산 워크플로우(Centralized · Integration-Manager · Dictator and Lieutenants), 50/72 커밋 규율, 그리고 사내 팀과 공개 Fork 프로젝트의 브랜치 기여 전략을 깊게 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Workflow', 'OpenSource', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#7)]({{< ref "/posts/pro-git-internals-ch4" >}})에서는 서버에 놓이는 Bare 저장소와 4대 전송 프로토콜, 그리고 현대 GitHub이 Smart HTTPS를 표준으로 권장하는 이유까지 서버 측 인프라를 살펴보았다. 이제 시선을 **서버 위에서 살아 움직이는 사람들의 협업 구조**로 옮길 차례다.

**Pro Git 5장**(Distributed Git)은 분량은 짧아 보이지만, Git이 왜 단순한 백업 도구가 아니라 **'분산형(DVCS)' 협업 플랫폼**인지 증명하는 가장 핵심적인 장이다. 2부작으로 나누어 진행하는 이번 1부(5.1~5.2)에서는 먼저 중앙 집중형과 분산형 저장소의 본질적인 아키텍처 차이부터 짚고, Git이 열어젖힌 3대 분산 워크플로우와 전 세계 오픈소스·엔터프라이즈 팀에서 통용되는 프로젝트 기여의 정석을 파헤쳐 본다.

---

## 1. 중앙 집중형(CVCS) vs 분산형(DVCS): 똑같이 중앙 서버를 써도 본질이 다른 이유

실무에서 Git을 처음 접하는 개발자들이 가장 흔히 던지는 질문이 있다. *"어차피 우리 회사도 GitHub이나 GitLab에 있는 중앙 저장소 하나에 다 같이 `push`하고 `pull`하는데, 이러면 Subversion 같은 중앙 집중식 VCS(CVCS)랑 다를 게 없지 않나요?"*

겉모습은 비슷해 보이지만, 저장소 내부를 지배하는 규칙은 **세 가지 근본적인 지점**에서 완전히 다르다.

1. **'중앙 서버'는 기술적 마스터가 아니라 사회적 합의다**: CVCS(Subversion, Perforce)에서는 오직 중앙 DB 서버만이 전체 히스토리를 보유하고, 각 개발자 PC는 현재 시점의 작업 사본(Working Copy)만 빌려 온다. 반면 Git에서는 **모든 개발자의 로컬 클론**(`.git`)이 100% 전체 히스토리를 갖춘 **동등한 피어**(Peer)다. 우리가 부르는 '공식 중앙 저장소(Blessed Repository)'는 기술적으로 우월한 특수 서버가 아니라, 팀원들이 기준점으로 삼기로 **약속한 하나의 피어 노드**일 뿐이다. 서버가 사라져도 아무 개발자의 클론 하나를 다시 Bare 서버로 올리면 저장소는 즉시 부활한다.
2. **`Commit`과 `Push`의 분리 (로컬 독립성)**: CVCS에서는 커밋을 누르는 순간 곧바로 중앙 서버에 반영되어야 하므로 네트워크가 끊기면 커밋조차 할 수 없다. Git에서는 내 로컬 `.git`에 오프라인으로 수십 번 커밋하고 브랜치를 분기·실험·재작성(`rebase`, `--amend`)하며 작업을 정제한 뒤, 준비가 끝났을 때만 원격에 **Push**해 타인과 공유한다.
3. **병합(Merge) 책임의 위치가 다르다**: Pro Git 5.2가 특별히 강조하는 대목이다. 예를 들어 John과 Jessica가 같은 저장소를 클론한 뒤 **서로 다른 파일**을 수정했다고 가정해 보자. Subversion 같은 CVCS에서는 Jessica가 먼저 커밋하고 John이 뒤이어 커밋해도 서버가 commit 시점에 자동으로 두 변경을 합쳐 준다. 하지만 Git에서는 Jessica가 먼저 Push한 상태에서 John이 곧바로 `git push origin master`를 실행하면 **`! [rejected] master -> master (non-fast-forward)`** 에러로 즉시 거절당한다. John이 로컬에서 Jessica의 변경분(`origin/master`)을 `fetch`해 자기 작업과 직접 통합(Merge 또는 Rebase)하고 테스트까지 마친 뒤에야 Push를 허락한다.

---

## 2. Git의 3대 분산 워크플로우 토폴로지 (Pro Git 5.1)

CVCS에서는 모든 개발자가 중앙 허브의 소비자(Consumer)로만 존재하지만, Git에서는 **모든 개발자가 동시에 노드**(Node)이자 **허브**(Hub)가 될 수 있다. 즉, 누군가에게 코드를 기여하면서도 동시에 나만의 공개 저장소를 유지해 다른 사람이 내 작업을 기반으로 개발하게 할 수 있다. 이 유연성 위에서 세 가지 대표적인 협업 토폴로지가 탄생했다.

{{< figure src="cvcs-dvcs-and-workflows.svg" alt="중앙 집중형(CVCS)과 분산형(DVCS) 아키텍처 비교 및 Git 3대 분산 워크플로우 토폴로지 다이어그램" caption="중앙 DB 종속 구조(CVCS)와 대등한 피어 구조(DVCS)의 차이 및 Centralized · Integration-Manager · Dictator and Lieutenants 3대 워크플로우 비교 (Pro Git 5.1 재구성)" width="820" height="540" >}}

### ① Centralized Workflow (중앙 집중식 워크플로우)

전통적인 CVCS와 가장 닮은 구조로, 사내 소규모 팀이나 비공개 프로젝트에서 가장 흔히 채택한다. 단 하나의 공용 저장소(Hub)에 모든 팀원이 Push 권한을 가지며, 앞서 본 것처럼 앞서 Push한 동료의 이력을 먼저 `fetch` & `merge`하지 않으면 Push를 막아 실수로 남의 코드를 덮어쓰는 참사를 차단한다. 브랜치 모델 덕분에 수백 명이 수십 개의 브랜치를 띄워 작업하는 환경으로도 거뜬히 확장된다.

### ② Integration-Manager Workflow (통합 관리자 워크플로우)

오늘날 **GitHub과 GitLab의 Fork & Pull Request 모델**이 그대로 계승한 아키텍처다.

- 공식 저장소(**`Blessed Repository`**)에는 통합 관리자(Maintainer)만 직접 Push 권한을 갖는다.
- 각 기여자는 자신의 **개인 공개 저장소**(Fork, 쓰기 가능)를 따로 복제해 두고, 작업이 끝나면 그곳에 Push한 뒤 메인테이너에게 "*내 변경분을 당겨 가 달라*(Pull Request)"고 요청한다.
- 메인테이너는 기여자의 저장소를 원격으로 추가해 로컬에서 테스트·검토한 뒤 공식 저장소에 병합해 Push한다.

기여자 입장에서는 메인테이너의 승인을 기다리는 동안에도 내 저장소에서 다음 작업을 멈추지 않고 독립적으로 이어갈 수 있다는 것이 가장 큰 강점이다.

### ③ Dictator and Lieutenants Workflow (독재자와 부관 워크플로우)

**리눅스 커널**처럼 수백~수천 명의 기여자가 매일 수천 개의 패치를 쏟아내는 초대규모 프로젝트에서 쓰는 다계층 통합 모델이다.

1. 일반 개발자들은 공식 기준 저장소(`Dictator`가 Push하는 Reference Repo)의 `master`를 기점으로 토픽 브랜치 작업을 진행해 제출한다.
2. 파일시스템·네트워크 등 각 서브시스템을 책임지는 **부관**(Lieutenant)들이 해당 분야 개발자들의 브랜치를 1차 리뷰해 자신의 저장소에 병합한다.
3. 최종 통합자인 **자비로운 독재자**(Benevolent Dictator, 예: Linus Torvalds)가 부관들의 저장소를 모아 자신의 `master`에 병합하고, 이를 다시 공식 기준 저장소로 Push해 모든 개발자가 그 위에서 다음 작업을 시작하게 한다.

*(더 다양한 실무 브랜치 전략 패턴이 궁금하다면 Pro Git이 추천하는 Martin Fowler의 가이드 [Patterns for Managing Source Code Branches](https://martinfowler.com/articles/branching-patterns.html)도 함께 참고할 만하다.)*

---

## 3. 좋은 기여(Contribution)를 만드는 엔지니어링 규율 (Pro Git 5.2 전반)

어떤 워크플로우를 쓰든, 코드를 다른 개발자나 메인테이너에게 넘기기 전에 반드시 지켜야 할 기본 규율이 있다. 핵심은 "**리뷰어가 내 변경 사항을 가장 적은 인지 부하로 검토하고 안전하게 되돌릴 수 있게 만드는 것**"이다.

{{< figure src="commit-guidelines-and-fork-workflow.svg" alt="Atomic Commit과 Tim Pope 50/72 커밋 메시지 규율 및 공개 Fork 프로젝트 이중 리모트 Topic 브랜치 격리 다이어그램" caption="기여자 커밋 체크리스트(공백 점검·원자적 분리·50/72 템플릿)와 공개 프로젝트 Fork(origin, myfork)에서의 Topic별 Rebase·Squash 전략 (Pro Git 5.2 재구성)" width="820" height="540" >}}

### ① 공백 오류 검증(`git diff --check`)과 원자적 커밋(Atomic Commit)

1. **공백 오류 사전 차단**: 줄 끝의 불필요한 공백(Trailing Whitespace) 등 사소한 포맷 노이즈가 diff를 더럽히지 않도록 커밋 직전에 `git diff --check`를 실행해 이상이 없는지 확인한다.
2. **논리적 단위(Atomic Commit) 분리**: 주말 내내 다섯 가지 이슈를 한꺼번에 고친 뒤 월요일 아침에 거대한 커밋 1개로 몰아넣어서는 안 된다. 설령 한 파일(`calc.py` 등) 안에서 두 가지 다른 이슈를 동시에 고쳤더라도 `git add --patch`(`-p`)로 변경 조각(Hunk)을 골라 스테이징해 **이슈당 최소 하나의 논리적으로 완결된 커밋**으로 나누어 남겨야 한다. 최종 브랜치 끝의 스냅샷 결과는 동일하지만, 코드 리뷰와 사후 `git revert` · `cherry-pick` 난이도는 하늘과 땅 차이다.

### ② 전 세계 표준이 된 `50/72` 커밋 메시지 규약

Pro Git은 Tim Pope의 템플릿을 인용하며 오늘날 Git 생태계 표준으로 자리 잡은 커밋 메시지 규칙을 강조한다.

```text
Capitalized, short (50 chars or less) summary

More detailed explanatory text, if necessary. Wrap it to about 72
characters or so. In some contexts, the first line is treated as the
subject of an email and the rest of the text as the body. The blank
line separating the summary from the body is critical (unless you omit
the body entirely); tools like rebase will confuse you if you run the
two together.

Write your commit message in the imperative: "Fix bug" and not "Fixed bug"
or "Fixes bug." This convention matches up with commit messages generated
by commands like git merge and git revert.

- Bullet points are okay, too
- Use a hanging indent
```

- **첫 줄 (50자 이내 명령형 요약)**: `"Fixed bug"`(과거형)나 `"Fixes bug"`(3인칭 단수)가 아니라 `"Fix bug"`처럼 **명령형**(Imperative)으로 적는다. `git merge`나 `git revert`가 자동 생성하는 메시지와 문체를 맞추기 위함이다.
- **두 번째 줄 (필수 빈 줄)**: 본문이 있다면 첫 줄과 본문 사이에 **반드시 빈 줄 1개**를 넣어야 `git log --oneline`이나 `git rebase` 같은 도구가 제목과 본문을 정확히 분리한다.
- **본문 (72자 줄바꿈)**: *무엇을 고쳤는가*보다 **왜 이 변경이 필요한가(Motivation)와 기존 동작 대비 무엇이 달라졌는가**를 적는다.

---

## 4. 팀 규모와 권한에 따른 3가지 실전 기여 시나리오 (Pro Git 5.2 후반)

권한 구조와 팀 규모에 따라 기여자가 Git 명령어를 엮는 실전 패턴은 크게 세 가지로 발전한다.

### ① 비공개 소규모 팀 (Private Small Team)

모든 팀원이 `origin` 저장소의 `master`에 Push 권한을 가진 환경이다. 앞서 1절에서 살펴보았듯 John이 `git push origin master`에서 `non-fast-forward` 거절을 당하면 다음 순서로 로컬에서 통합한 뒤 재시도한다.

```bash
# 1. 원격의 최신 변경을 로컬 origin/master 북마크로 내려받기
$ git fetch origin

# 2. (예시) topic 브랜치(issue54) 기준으로 origin/master에 새로 들어온 커밋만 미리 검토
$ git log --no-merges issue54..origin/master

# 3. 내 master에 topic과 origin/master를 모두 통합한 뒤 원격 푸시
$ git checkout master
$ git merge issue54
$ git merge origin/master
$ git push origin master
72bbc59..8059c15  master -> master
```

참고로 푸시 결과 마지막 줄의 `<oldref>..<newref> fromref -> toref` 출력(`72bbc59..8059c15 master -> master`)은 원격 참조가 기존 어느 커밋에서 새 커밋으로 옮겨 갔는지를 보여 준다.

### ② 비공개 관리형 팀 (Private Managed Team)

프로젝트가 조금만 커져도 일반 개발자에게 `master` 직접 푸시 권한을 닫고 통합 담당자(Integrator)만 `master`를 갱신하게 만든다. 대신 개발자들은 `featureA`, `featureB` 같은 기능 브랜치를 공유하며 협업한다.

만약 내가 `origin/master`에서 `featureB` 브랜치를 따서 작업했는데, 동료 Josie가 이미 원격에 `featureBee`라는 이름으로 같은 기능 브랜치를 올려 두었다면 어떻게 합칠까?

```bash
# 1. 동료가 올린 원격 브랜치(origin/featureBee)를 가져와 내 로컬 featureB와 병합
$ git fetch origin
$ git merge origin/featureBee

# 2. Refspec 문법(<로컬>:<원격>)을 사용해 내 featureB를 원격의 featureBee로 푸시 + 업스트림(-u) 연결
$ git push -u origin featureB:featureBee
fba9af8..cd685d1  featureB -> featureBee
```

### ③ 공개 프로젝트 Fork & Pull Request (Forked Public Project)

공식 저장소에 쓰기 권한이 아예 없는 오픈소스 기여(또는 사내 InnerSource PR 문화)에서 가장 중요한 원칙은 "**절대로 내 `master` 브랜치 위에서 직접 작업하지 않는다**"는 것이다.

1. 공식 저장소(`origin`)를 클론해 로컬 `master`는 항상 `origin/master`만 순수하게 추적하도록 비워 둔다.
2. 작업 주제마다 `origin/master`를 기점으로 독립 **Topic 브랜치**(`featureA`, `featureB`)를 따로 만들고, 내 쓰기 가능한 공개 저장소(**`myfork`**)에만 Push한다.
3. 만약 메인테이너가 다른 PR들을 먼저 머지해 내 `featureA`가 더 이상 깔끔하게 병합되지 않는다면, 해당 브랜치만 골라 `origin/master` 위로 리베이스한 뒤 내 Fork에 강제 푸시(`-f`)한다. (공식 저장소가 아니라 내 개인 Fork의 토픽 브랜치이므로 안전하다.)

```bash
# Case A: 기존 featureA를 최신 origin/master 위로 재정렬 후 Fork 갱신
$ git fetch origin
$ git checkout featureA
$ git rebase origin/master
$ git push -f myfork featureA

# Case B: 리뷰 피드백을 반영해 여러 커밋을 1개로 깔끔히 압축(Squash)한 새 브랜치 제출
$ git checkout -b featureBv2 origin/master
$ git merge --squash featureB
# ... 리뷰 피드백 수정 작업 ...
$ git commit
$ git push myfork featureBv2
```

여기서 `git merge --squash featureB`는 `featureB`의 모든 변경 사항을 한꺼번에 워킹 트리와 Index로 가져오되 **병합 커밋(부모 2개)을 만들지 않고**, 이후 `git commit` 시 깔끔한 단일 커밋(부모 1개)으로 새로 기록할 수 있게 해 주는 유용한 테크닉이다.

> **레거시 워크플로우 한 줌: `git request-pull`과 이메일 패치 (`format-patch`, `send-email`)**
>
> 오늘날에는 GitHub 웹 UI에서 "Create pull request" 버튼만 누르면 되지만, Pro Git 5.2는 웹 허브가 없던 시절(그리고 지금도 리눅스 커널·Git 코어 개발에서 현역으로 쓰이는) 전통 방식도 함께 기록해 두고 있다.
> - **`git request-pull origin/master myfork`**: 기준 커밋과 변경 커밋 요약, Diffstat, 그리고 당겨 갈 저장소 URL/브랜치를 텍스트 요약서로 뽑아 메일로 보내는 명령이다(오늘날 PR 본문 요약의 원조다).
> - **`git format-patch` & `git send-email`**: Fork 저장소조차 없이, 각 커밋을 `0001-...patch` 같은 유닉스 mbox 이메일 포맷으로 굽고(`format-patch`) SMTP(`send-email`)나 IMAP(`imap-send`)으로 개발자 메일링 리스트에 직접 쏘는 방식이다. 웹 기반 Git 호스팅을 쓰는 현대 실무에서는 거의 마주칠 일이 없지만, Git의 커밋 본문 `50/72` 규율이나 `---` 구분선 문화가 왜 이메일 구조에서 유래했는지 이해하는 좋은 열쇠가 된다.

---

## 마치며

이번 편에서는 **Pro Git 5장 전반부**(5.1~5.2)를 통해 분산형 버전 관리 시스템이 만들어 내는 협업 토폴로지와 기여자의 규율을 정리했다.

- Git의 중앙 서버는 기술적 마스터가 아닌 **합의된 피어**(Blessed Repo)이며, `Commit`과 `Push`의 분리와 Non-Fast-Forward Push 거절이 로컬 선(先)병합을 강제한다.
- 단일 공유 저장소 모델(**Centralized**), Fork & PR의 원형인 **Integration-Manager**, 초대규모 계층형 **Dictator & Lieutenants**까지 프로젝트 규모에 맞춰 구조를 유연하게 선택할 수 있다.
- 기여자는 **`git diff --check`**, **`git add -p` 원자적 분리**, **50/72 명령형 커밋 메시지**, 그리고 **`master`를 더럽히지 않는 Topic 브랜치 격리**를 지킬 때 가장 리뷰받기 좋은 코드를 만들 수 있다.

다음 2부(`[Pro Git 톺아보기 #9]`)에서는 시선을 반대편인 **메인테이너(Maintainer)와 시니어 리뷰어**로 돌려, 기여받은 코드를 정교하게 검증하고 통합하는 기술인 **이중 점(`A..B`) vs 삼중 점(`A...B`)**, **`git cherry-pick`**, **`git rerere`**, 그리고 **`git describe`** 릴리즈 빌드 식별자까지 파헤쳐 본다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 5.1 Distributed Git - Distributed Workflows](https://git-scm.com/book/en/v2/Distributed-Git-Distributed-Workflows)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 5.2 Distributed Git - Contributing to a Project](https://git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project)
- [Martin Fowler - Patterns for Managing Source Code Branches](https://martinfowler.com/articles/branching-patterns.html)
