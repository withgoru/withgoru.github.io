+++
date = '2026-11-24T09:00:00+09:00'
draft = false
title = '[Pro Git 톺아보기 #6] Rebase 내부 동작 원리(--onto, patch-id)와 Golden Rule, 그리고 Merge vs Rebase 철학'
description = 'Pro Git 3장 후반부(3.6)를 바탕으로 git rebase의 패치 추출·재적용 메커니즘, --onto를 활용한 다중 토픽 브랜치 분리, 공개 커밋 Rebase 참사와 patch-id 기반 복구(git pull --rebase), 그리고 Merge와 Rebase의 히스토리 철학을 깊게 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글]({{< ref "/posts/pro-git-internals-ch3-part2" >}})에서는 `--merged`/`--no-merged`를 이용한 브랜치 관리, Long-Running 및 Topic 브랜칭 워크플로우, 그리고 원격 저장소와의 동기화를 담당하는 리모트 트래킹 브랜치(`origin/master`)와 업스트림 추적 원리를 살펴보았다.

이번 글은 **Pro Git 3장(Git Branching)의 마지막 편**으로, **3.6 Rebasing**을 다룬다. Git에서 한 브랜치의 변경 사항을 다른 브랜치에 통합하는 방법은 크게 **Merge**와 **Rebase** 두 가지다. Rebase가 내부적으로 커밋을 어떻게 분해해 다시 쌓는지, `--onto` 옵션으로 복잡한 분기 그래프를 어떻게 도려내어 붙이는지, 그리고 공개 저장소에서 Rebase를 오용했을 때 왜 대참사가 벌어지며 `patch-id`로 어떻게 복구하는지 끝까지 파헤쳐 본다.

---

## 1. Rebase의 내부 동작 원리: 패치 추출과 히스토리 재작성 (Pro Git 3.6)

### 기본 Rebase의 4단계 메커니즘 (`C4` → `C4'`)

공통 조상 `C2`에서 분기해 `master`는 `C3`으로, `experiment`는 `C4`로 나아간 상황을 떠올려 보자. 3-Way Merge를 사용하면 `C3`과 `C4`를 합친 새 병합 커밋 `C5`가 생긴다. 반면 `git rebase`를 사용하면 `C4`에서 만든 변경 패치를 떼어내어 그대로 `C3` 위에 다시 쌓을 수 있다.

```bash
$ git checkout experiment
$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: added staged command
```

명령이 실행되는 동안 Git 내부에서는 다음 네 단계가 순차적으로 일어난다.

1. 현재 체크아웃된 브랜치(`experiment`)와 대상 브랜치(`master`)의 **공통 조상**(`C2`)으로 거슬러 올라간다.
2. 현재 브랜치(`experiment`)의 각 커밋(`C4`)이 이전 커밋 대비 만들어낸 변경분(Diff)을 추출해 **임시 파일(패치)로 저장**한다.
3. 현재 브랜치 포인터(`experiment`)를 재적용할 대상 브랜치(`master`가 가리키는 `C3`)로 **재설정**(Reset)한다.
4. 앞서 저장해 둔 임시 패치들을 순서대로 다시 적용해 **새로운 커밋 객체**(`C4'`)을 생성한다. (주의: 내용은 같아도 부모 커밋과 타임스탬프가 달라지므로 SHA-1 해시는 `C4`와 전혀 다른 완전히 새로운 커밋이다.)

이제 `master`로 돌아가 `experiment`를 병합하면 직계 조상 관계가 성립하므로 **Fast-forward Merge**가 일어나 두 브랜치가 나란히 `C4'`를 가리키게 된다.

```bash
$ git checkout master
$ git merge experiment
```

여기서 핵심 통찰이 하나 있다. **`C4'`가 가리키는 최종 Tree 스냅샷은 3-Way Merge로 만든 `C5`의 스냅샷과 완벽히 동일하다.** 결과물(코드 상태)은 같지만, Rebase를 거친 로그는 병렬 작업이었음에도 처음부터 한 줄로 개발된 것처럼 깔끔한 선형 히스토리를 남긴다. 오픈소스 기여 시 메인테이너에게 패치를 보내기 직전 `origin/master` 위로 `rebase`해 두면, 메인테이너가 복잡한 병합 없이 깔끔하게 Fast-forward로 반영할 수 있는 이유가 바로 여기에 있다.

### 고급 Rebase: `--onto` 옵션과 체크아웃 생략 단축 문법

Rebase는 단순히 현재 브랜치 전체를 다른 브랜치 끝으로 옮기는 것 외에도, **`--onto`** 옵션을 통해 특정 분기 지점 이후의 커밋만 정밀하게 오려 내어 전혀 다른 줄기에 붙일 수 있다.

`master`에서 서버 작업용 `server` 브랜치를 따서 `C3`, `C4`를 커밋한 뒤, 그 위에서 클라이언트 작업용 `client` 브랜치를 다시 따서 `C8`, `C9`를 만들고, 이후 `server`로 돌아와 `C10`을 추가한 3단 중첩 구조를 가정해 보자. 이때 아직 미완성인 `server` 코드는 보류하고 **`client`의 변경 사항(`C8`, `C9`)만 먼저 `master`에 릴리즈**하고 싶다면 다음과 같이 실행한다.

```bash
$ git rebase --onto master server client
```

이 명령은 Git에게 "**`client` 브랜치를 대상으로 삼되, `server` 브랜치에서 갈라져 나온 이후의 패치(`C8`, `C9`)만 추출해서 마치 처음부터 `master`(`C6`)에서 분기했던 것처럼 그 위에 다시 적용하라**"고 지시한다.

```bash
# 1. master를 client(C8', C9')로 Fast-forward
$ git checkout master
$ git merge client

# 2. 이후 server 테스트가 끝나면 별도 체크아웃 없이 바로 master 위로 rebase
# 문법: git rebase <basebranch> <topicbranch>
$ git rebase master server

# 3. master를 server 위로 Fast-forward 후 불필요한 토픽 브랜치 정리
$ git checkout master
$ git merge server
$ git branch -d client
$ git branch -d server
```

특히 `git rebase master server`처럼 베이스 브랜치와 토픽 브랜치를 나란히 넘기면 `git checkout server`를 생략하고 한 번에 리베이스할 수 있다.

{{< figure src="rebase-internals.svg" alt="Git Rebase 내부 동작(C4에서 C4' 재적용, --onto)과 공개 커밋 강제 푸시 참사 및 patch-id 기반 복구 다이어그램" caption="기본 Rebase와 --onto 옵션의 커밋 재적용 구조, 그리고 공개 커밋 강제 푸시 시 일반 Merge 참사와 patch-id 기반 Rebase 복구 비교 (Pro Git 3.6 재구성)" width="820" height="560" >}}

---

## 2. Rebase의 위험성(The Perils of Rebasing)과 복구 메커니즘

Rebase는 히스토리를 아름답게 다듬는 강력한 도구이지만, 잘못 사용하면 협업 저장소 전체를 혼란에 빠뜨린다. 이를 막기 위한 절대 원칙이 존재한다.

### Rebase의 절대 원칙 (The Golden Rule)

> **"이미 자신의 저장소 바깥으로 나가 다른 사람들이 기반 작업(Base)을 쌓았을 가능성이 있는 커밋은 절대로 Rebase하지 마라."**

앞서 보았듯 Rebase는 기존 커밋을 수정하는 것이 아니라 **기존 커밋을 버리고(`Abandon`) 해시가 다른 비슷한 새 커밋을 생성**한다. 만약 내가 이미 서버에 푸시한 커밋을 동료가 Pull해 그 위에서 새 커밋을 만들었는데, 뒤늦게 내가 로컬에서 그 커밋들을 Rebase한 뒤 강제 푸시(`git push --force`)해 버리면 히스토리가 꼬이기 시작한다.

### 강제 푸시(`git push --force`)가 부르는 중복 커밋 참사

위 다이어그램 하단 왼쪽(A)을 따라가며 실제로 어떤 재앙이 일어나는지 확인해 보자.

1. 내가 중앙 서버(`teamone`)에서 `C1`을 Clone해 작업을 시작했다. 그사이 동료가 커밋과 병합이 포함된 작업(`C4`, `C6`)을 서버에 푸시했다.
2. 나는 동료의 변경분을 Fetch해 내 작업(`C2`, `C3`)과 합친 Merge 커밋 `C7`을 만들었다. 이때 내 히스토리는 동료가 올린 `C4`와 `C6`을 부모로 물고 있다.
3. 그런데 동료가 방금 자신이 푸시한 이력이 마음에 들지 않아 **로컬에서 `C4`, `C6`을 `C4'` 하나로 Rebase한 뒤 `git push --force`로 서버 히스토리를 덮어써 버렸다.**
4. 이 상태에서 내가 서버의 변경을 가져오기 위해 `git pull`(일반 Merge)을 실행하면 대참사가 일어난다. Git 입장에서는 내 `C7` 아래에 여전히 살아 있는 `C4`, `C6`과 서버에서 새로 받은 `C4'`가 서로 다른 SHA-1 해시를 가진 별개의 커밋이므로, **양쪽을 모두 합치는 새로운 Merge 커밋 `C8`을 만들어 버린다.**

그 결과 `git log`를 조회하면 작성자와 날짜, 커밋 메시지, 심지어 수정 내용까지 동일한 `C4`와 `C4'` 커밋이 히스토리에 두 번 나란히 기록된다. 설상가상으로 내가 이 `C8`을 다시 서버에 Push하면 동료가 애써 정리해 지웠던 `C4`, `C6` 커밋이 중앙 서버에 다시 부활해 팀 전체를 혼란에 빠뜨린다.

### Rebase를 Rebase로 덮기: `patch-id`와 `git pull --rebase`

만약 동료의 강제 푸시로 이런 난처한 상황에 처했다면, 일반 `git pull`(Merge) 대신 **`git rebase teamone/master`**(또는 **`git pull --rebase`**)를 실행해 복구할 수 있다.

Git이 이미 버려진 커밋 `C4`와 새로 쓰인 `C4'`가 본질적으로 같은 작업이라는 사실을 어떻게 알아낼 수 있을까? Git은 커밋 전체 객체의 SHA-1 해시 외에도 **해당 커밋이 도입한 순수 패치(Diff) 내용만을 기준으로 계산하는 체크섬**인 `patch-id`를 함께 계산하기 때문이다.

동료의 강제 푸시 이후 `git rebase teamone/master`를 실행하면 Git은 내부적으로 다음 네 단계를 수행한다.

1. 내 로컬 브랜치(`C7`)에만 고유하게 존재하는 커밋 집합(`C2`, `C3`, `C4`, `C6`, `C7`)을 추린다.
2. 그중 Merge 커밋(`C7`)을 제외한다(`C2`, `C3`, `C4`).
3. 각 커밋의 `patch-id`를 대상 브랜치(`teamone/master`의 `C4'`)와 대조해, 이미 상대 브랜치에 재작성되어 반영된 커밋(`C4`)은 건너뛰고 진짜 내 작업만 남긴다(`C2`, `C3`).
4. 남은 내 커밋들만 `teamone/master`(`C4'`) 위로 깔끔하게 다시 적용해 `C2' ← C3'`를 만든다.

단, 이 자동 중복 제거는 `C4`와 `C4'`의 패치가 사실상 동일할 때만 동작한다. 동료가 Rebase 도중 내용을 크게 고쳐 `patch-id`가 달라졌다면 중복을 감지하지 못해 충돌이 날 수 있다. 매번 `git pull` 실행 시 Merge 대신 Rebase를 기본값으로 쓰고 싶다면 아래와 같이 설정할 수 있다.

```bash
# 특정 Pull 시에만 Rebase 사용
$ git pull --rebase

# 전역 기본 동작을 pull --rebase로 고정
$ git config --global pull.rebase true
```

---

## 3. Rebase vs Merge: 히스토리를 바라보는 두 가지 철학

그렇다면 결국 **Merge**와 **Rebase** 중 무엇을 써야 할까? Pro Git은 이 질문이 단순한 우열 문제가 아니라 "**커밋 히스토리를 무엇으로 정의할 것인가**"라는 두 가지 관점의 차이라고 설명한다.

| 관점 | 커밋 히스토리의 의미 | 선호 전략 |
| :--- | :--- | :--- |
| **역사적 기록(Record)** | 저장소 히스토리는 **실제로 일어난 일의 사실적 기록**이다. 투박한 병합 커밋과 시행착오의 흔적 역시 후대를 위해 고스란히 보존되어야 하며, 역사를 수정하는 것은 거짓을 쓰는 것과 같다. | **Merge** |
| **정제된 이야기(Story)** | 커밋 히스토리는 **프로젝트가 어떻게 만들어졌는지에 대한 서사**다. 책의 초고를 그대로 출간하지 않듯, 개발 중의 자잘한 실수와 지저분한 과정은 다듬어 미래의 동료가 읽기 좋은 직관적인 흐름을 남겨야 한다. | **Rebase** |

두 철학의 장점을 모두 취하는 가장 현명한 실무 전략은 명확하다. **아직 내 로컬 머신에만 있고 외부에 공유되지 않은 커밋은 Push하기 전에 `rebase`로 깔끔하게 다듬고, 한 번이라도 공유 저장소에 Push되어 다른 사람이 참조할 수 있는 커밋은 절대 `rebase`하지 않는 것**이다.

---

## 마치며

이로써 총 3부작에 걸쳐 **Pro Git 3장**(Git Branching) 전체를 완주했다.

- **1부 (`#4`)**: `.git/refs/heads/` 안의 **41바이트 파일**인 브랜치 포인터와 `HEAD`, 그리고 직계 조상 여부에 따라 갈리는 **Fast-forward** vs **3-Way Merge**의 동작 원리
- **2부 (`#5`)**: `--merged`/`--no-merged` 관리 기법, **Long-Running / Topic 브랜칭 워크플로우**, 그리고 서버의 북마크인 **리모트 트래킹 브랜치**(`origin/master`)와 업스트림 추적
- **3부 (`#6`)**: 패치 추출·재적용으로 선형 이력을 만드는 **`git rebase`**(`--onto`), 공개 커밋 재작성을 금지하는 **Golden Rule**, 그리고 `patch-id`를 활용한 **`git pull --rebase`** 복구 메커니즘

Git 브랜치와 병합·리베이스의 내부 동작을 머릿속 객체 그래프로 그릴 수 있게 되었다면, 이제 어떤 복잡한 협업 시나리오나 충돌 앞에서도 당황하지 않고 원하는 상태로 저장소를 제어할 수 있다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.6 Git Branching - Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
