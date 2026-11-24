+++
date = '2026-12-07T09:00:00+09:00'
draft = false
title = 'Git Packfile #18: 델타 압축과 Refspec·프로토콜'
description = 'Pro Git 10장 2부(10.4~10.6)를 바탕으로 Packfile 역방향 델타 압축, Refspec 매핑 규칙, Dumb vs Smart 전송 프로토콜의 패킷 협상을 정리한다.'
tags = ['Git', 'ProGit', 'GitInternals', 'Packfile', 'Refspec', 'Protocol', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#17)](/posts/pro-git-internals-ch10-part1/)에서는 Git의 모든 데이터가 Blob·Tree·Commit·Tag 객체로 분해되어 `.git/objects/` 에 저장되고, `.git/refs/` 와 `HEAD` 가 이를 가리키는 내부 구조를 살펴보았다.

여기서 자연스럽게 하나의 의문이 생긴다. "1장부터 Git은 변경분(Delta)이 아니라 매 커밋마다 파일 전체 스냅샷을 새 Blob으로 저장한다고 했는데, 수만 번 커밋해도 저장소 용량이 생각보다 엄청나게 커지지 않는 이유는 무엇일까?"

이번 **10장 2부(`#18`, Pro Git 10.4~10.6)** 에서는 그 비밀을 풀어 주는 **Packfile의 역방향 델타 압축**, 원격과 로컬 참조 이름을 자유자재로 연결하는 **Refspec**, 그리고 네트워크 위에서 실제로 오가는 **전송 프로토콜 패킷 협상 원리**를 파헤친다.

---

## 1. Packfile의 반전: 스냅샷 Git이 디스크를 절약하는 비밀 (10.4)

초기 커밋 단계에서 `.git/objects/` 에 하나씩 따로 저장되는 개별 객체 파일을 **Loose Object**라고 부른다. Loose Object는 생성이 단순하고 빠르지만, 만약 10MB짜리 데이터 파일에서 딱 1줄만 고치고 10번 커밋하면 디스크에 10MB짜리 Blob 파일 10개(총 100MB)가 통째로 쌓인다는 문제가 생긴다.

이 문제를 해결하기 위해 Git은 주기적으로(또는 `git gc` 실행 시) 여러 Loose Object를 하나의 바이너리 아카이브인 **Packfile(`.pack`)** 과 이진 탐색용 오프셋 색인인 **Index 파일(`.idx`)** 로 묶는다.

{{< figure src="packfiles-refspec-and-wire-protocol.svg" alt="Git Packfile 역방향 델타 압축, Refspec 매핑 규칙 및 Smart Protocol want와 have 패킷 협상 다이어그램" caption="최신 버전을 온전히 두고 과거 버전을 역방향 델타로 압축하는 Packfile 구조(상단), Refspec 매핑 규칙(중단), Smart Protocol 패킷 협상 흐름(하단) (Pro Git 10.4~10.6 재구성)" width="820" height="580" >}}

### 1.1 왜 "역방향 델타 압축(Reverse Delta)"인가?

흔히 Subversion 같은 델타 기반 VCS는 최초 버전을 원본으로 두고 이후 수정본을 순방향 델타(`v1 → +delta → v2 → +delta → v3`)로 쌓는다. 하지만 Git이 Packfile을 굽는 방식은 완전히 반대다.

**Git은 가장 최신 버전을 온전한 원본(Base)으로 남기고, 오히려 과거 버전을 최신 버전 기준의 역방향 델타(Reverse Delta)로 압축한다.**

이 설계의 이유는 명쾌하다. 개발자가 수행하는 `git checkout`, `git diff`, `git merge` 작업의 99%는 **가장 최근 커밋**을 대상으로 일어난다. 과거 원본에 델타를 덧대는 방식은 최신 파일을 읽을 때마다 수백 번의 델타 재계산이 필요하지만, **최신 본체를 그대로 두는 Git의 역방향 델타 방식**은 현재 작업 버전을 읽을 때 단 한 번의 압축 해제만으로 즉시 반환할 수 있다.

### 1.2 [터미널 실습] `git gc`와 `git verify-pack -v`로 역방향 델타 직접 목격하기

약 25KB(500줄)짜리 파일을 커밋(`v1`)한 뒤 맨 아래 1줄만 추가해 다시 커밋(`v2`)하고, `git gc`를 돌려 Packfile 내부를 들여다보자.

```bash
$ mkdir pack-lab && cd pack-lab && git init -b main
$ python3 -c "open('big.txt','w').write('\n'.join([f'line {i} sample data for packfile compression test' for i in range(500)]) + '\n')"
$ git add big.txt && git commit -m "v1"
$ echo "appended line 500" >> big.txt
$ git add big.txt && git commit -m "v2"

# gc 이전에는 Loose Object 파일 6개 (Blob 2 + Tree 2 + Commit 2)
$ find .git/objects -type f | wc -l
6

# 가비지 컬렉션 실행 후 객체 파일 구조 확인
$ git gc
$ find .git/objects -type f
.git/objects/pack/pack-9c78ce2cd6c75b9aae80d22b65245c80d64b2cc2.idx
.git/objects/pack/pack-9c78ce2cd6c75b9aae80d22b65245c80d64b2cc2.rev
.git/objects/pack/pack-9c78ce2cd6c75b9aae80d22b65245c80d64b2cc2.pack
.git/objects/info/commit-graph
.git/objects/info/packs
```

이제 `git verify-pack -v`로 `.idx` 파일을 검증해 보면, **정말로 최신 `v2` Blob이 원본이고 과거 `v1` Blob이 델타로 바뀌었는지** 한눈에 확인할 수 있다.

```bash
$ git verify-pack -v .git/objects/pack/pack-*.idx
ddaf89c65f40079ce04a557b336e65af7e4f9277 commit 209 143 12
14766864d53bccefafed0738bdba51c21f2aaf52 commit 161 114 155
01384aa846342d257add628328a1bfef1ae577eb blob   25408 1340 269
be0586f42c259ed24a37e51d5abaa0f73edfffc8 tree   35 46 1609
83a306d08645c648c9c299cb794c4e90613e763c tree   35 46 1655
a854552adfb8f5ff1bbeabe1e22944f4562f6bd4 blob   9 20 1701 1 01384aa846342d257add628328a1bfef1ae577eb
non delta: 5 objects
chain length = 1: 1 object
.git/objects/pack/pack-9c78ce2cd6c75b9aae80d22b65245c80d64b2cc2.pack: ok
```

출력 열(Column)을 해석하면 다음과 같다: `[SHA-1] [타입] [원본 크기] [Pack 내 압축 크기] [오프셋] ([델타 깊이] [베이스 SHA-1])`.

- 세 번째 줄 `01384aa...`가 **최신 `v2` Blob**이다. 원본 `25,408`바이트가 Packfile 안에서 `1,340`바이트로 zlib 압축되어 온전히 저장되었고, 델타 깊이가 없다(`non-delta`).
- 마지막 줄 `a854552...`가 **과거 `v1` Blob**이다. 뒤에 **`1 01384aa8...`** 이 붙어 있는데, 이는 **"최신 Blob `01384aa...`를 베이스로 삼는 깊이 1짜리 역방향 델타"** 라는 뜻이며, 무려 **20바이트** 크기로 줄어들었다.

*(참고: 현대 Git에서는 Packfile이 여러 개로 늘어날 때 탐색 속도를 유지하기 위해 여러 `.idx`를 하나로 통합 색인하는 **Multi-Pack-Index(`git multi-pack-index write`)** 와 커밋 부모 그래프 탐색을 가속하는 **Commit-Graph(`.git/objects/info/commit-graph`)** 를 자동으로 생성·활용한다.)*

---

## 2. The Refspec 완전 정복 (10.5)

원격 저장소를 추가(`git remote add origin ...`)하고 나면 `.git/config` 파일 안에 다음과 같은 낯설지만 중요한 설정 블록이 자동으로 들어간다.

```ini
[remote "origin"]
	url = https://github.com/user/repo.git
	fetch = +refs/heads/*:refs/remotes/origin/*
```

여기서 `fetch = ` 뒤에 적힌 **`+refs/heads/*:refs/remotes/origin/*`** 문자열이 바로 **Refspec(Reference Specification)** 이다. Refspec은 **"원격의 어떤 참조를 로컬의 어떤 참조 이름에 매핑할 것인가"** 를 정의하는 규칙이며 일반 문법은 **`[+]<src>:<dst>`** 다.

- **`+` (선택 접두사)**: Fast-Forward로 이어지지 않는 갱신(예: 누군가 원격 브랜치를 Force Push해 히스토리가 바뀐 경우)도 강제로 덮어쓰도록 허용한다.
- **`<src>` (원본 참조 패턴)**: Fetch 시에는 원격 저장소의 경로, Push 시에는 로컬 저장소의 경로를 뜻한다.
- **`<dst>` (대상 참조 패턴)**: Fetch 시에는 로컬 저장소의 경로, Push 시에는 원격 저장소의 경로를 뜻한다. (와일드카드 `*`는 단일 계층 또는 전체 접미사 매핑에 쓸 수 있다.)

### 실무에서 빛나는 Refspec 응용 패턴 3가지

1. **GitHub PR 브랜치 한 번에 Fetch하기**: [6장(#10)](/posts/pro-git-internals-ch6/)에서 다룬 것처럼 GitHub은 모든 PR을 읽기 전용 숨김 네임스페이스 `refs/pull/<번호>/head`에 보관한다. 아래 Refspec 한 줄을 `.git/config`에 추가하면 `git fetch origin` 만으로 모든 PR을 `origin/pr/<번호>`로 받아 즉시 체크아웃할 수 있다.
   ```ini
   fetch = +refs/pull/*/head:refs/remotes/origin/pr/*
   ```
2. **특정 네임스페이스만 부분 Fetch·Push 하기**: 사내 팀별(`refs/heads/qa/*`) 브랜치만 골라 `git fetch origin refs/heads/qa/*:refs/remotes/origin/qa/*` 형태로 좁혀 받거나, 로컬 `main`을 원격의 다른 브랜치 이름(`git push origin main:refs/heads/qa/main`)으로 밀어 넣을 수 있다.
3. **빈 `<src>` 로 원격 브랜치 삭제하기**: `git push origin --delete topic`의 내부 동작은 사실 **`git push origin :topic`** 이다. `<src>` 자리를 비워 두고 `<dst>`만 넘기면 "아무것도 없는 상태(null)를 원격 `topic`으로 푸시하라"는 뜻으로 해석되어 원격 브랜치가 삭제된다.

---

## 3. Transfer Protocols: 와이어 레벨 패킷 협상 (10.6)

[4장(#7)](/posts/pro-git-internals-ch4/)에서 HTTPS와 SSH 프로토콜의 보안·인증 관점 차이를 비교했다면, Pro Git 10.6은 네트워크 선로(Wire) 위에서 클라이언트와 서버가 데이터를 주고받는 **Dumb vs Smart 전송 프로토콜의 내부 동작**을 다룬다.

### 3.1 Dumb Protocol (읽기 전용 정적 웹 서버)

**Dumb Protocol**은 서버에 Git 전용 데몬이 전혀 돌지 않고, Apache나 Nginx 같은 정적 HTTP 서버가 `.git/` 디렉토리 파일만 그대로 노출하는 구식 방식이다.

1. 클라이언트가 먼저 `GET /repo.git/info/refs`와 `GET /repo.git/HEAD`를 요청한다. (그래서 서버 쪽에서 커밋이 생길 때마다 `git update-server-info` 후크를 돌려 정적 인덱스 파일을 갱신해 두어야 한다.)
2. 클라이언트는 Commit → Tree → 필요한 Blob 또는 전체 `.pack` 파일을 찾기 위해 **수십~수백 번의 HTTP GET 왕복 요청**을 일일이 날려 파일을 긁어 온다.
3. 서버가 "클라이언트가 이미 갖고 있는 객체가 무엇인지" 지능적으로 계산할 수 없으므로 네트워크 낭비가 극심하고, HTTP 정적 서빙 특성상 **Push(쓰기)는 불가능하다.**

### 3.2 Smart Protocol (`git-upload-pack` · `git-receive-pack`)

오늘날 우리가 GitHub·GitLab·SSH로 쓰는 방식은 모두 **Smart Protocol**이다. 양쪽 끝에 Git 프로세스가 떠서 실시간으로 대화한다.

- **Clone / Fetch / Pull 시**: 클라이언트의 `git-fetch-pack`이 서버의 **`git-upload-pack`** 프로세스와 통신한다.
- **Push 시**: 클라이언트의 `git-send-pack`이 서버의 **`git-receive-pack`** 프로세스와 통신한다.

실제 Fetch 시 와이어 위에서는 다음과 같은 **`pkt-line` 패킷 협상**이 순식간에 일어난다.

1. **참조 광고 (Ref Advertisement)**: 서버(`git-upload-pack`)가 자신의 모든 브랜치·태그 SHA-1 해시와 지원 기능(`multi_ack`, `thin-pack`, `side-band-64k` 등)을 클라이언트에게 알린다. *(Git Wire Protocol v2에서는 클라이언트가 먼저 원하는 prefix만 필터링 요청해 거대 저장소의 Ref 광고 부하를 대폭 줄였다.)*
2. **`want` / `have` 공통 조상 협상**: 클라이언트는 내가 받아야 할 원격 커밋 해시 목록을 **`want <해시>`** 로 보내고, 내가 이미 로컬에 보유하고 있는 최신 커밋 해시 목록을 **`have <해시>`** 로 보낸다.
3. **동적 Thin Packfile 스트리밍**: 서버는 `want`와 `have`를 대조해 **"클라이언트가 이미 가진 부모 커밋 이후로 부족한 객체"** 만 정확히 추려낸 뒤, 서버 메모리/CPU에서 즉석으로 최소 크기의 Packfile(심지어 클라이언트가 이미 가진 객체를 베이스로 삼는 Thin Pack)을 생성해 단일 스트림으로 쏴 준다.

덕분에 수 기가바이트짜리 리눅스 커널 저장소라도 어제 이후 커밋 3개만 새로 올라왔다면 `git pull` 시 단 몇 킬로바이트의 패킷만 오가며 1초 만에 동기화가 끝난다.

---

## 마치며

이번 **10장 2부(`#18`)** 에서는 Git 저장소가 디스크와 네트워크 비용을 극한으로 아끼는 내부 메커니즘을 확인했다.

- **Packfile (10.4)**: 최신 스냅샷을 온전히 남기고 과거 이력을 **역방향 델타**로 압축해 읽기 성능과 용량 절감을 동시에 달성한다.
- **Refspec (10.5)**: `+<src>:<dst>` 패턴으로 원격 PR 네임스페이스를 가져오거나 빈 `<src>`(`:branch`)로 원격 브랜치를 삭제할 수 있다.
- **Smart Protocol (10.6)**: 서버의 `git-upload-pack` 과 클라이언트가 `want`/`have` 패킷을 주고받아 꼭 필요한 객체만 즉석 Packfile로 전송한다.

이제 시리즈의 대미를 장식할 마지막 **[10장 3부(#19)](/posts/pro-git-internals-ch10-part3/)** 에서는 **`git gc`와 `git maintenance`**, Reflog마저 날아간 커밋을 **`git fsck`로 바닥에서 건져 올리는 데이터 복구 실습**, 그리고 실무·CI 디버깅의 치트키인 **Git 환경 변수 총정리**를 끝으로 Pro Git 톺아보기 시리즈 전체를 완결짓는다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.4 Packfiles](https://git-scm.com/book/en/v2/Git-Internals-Packfiles)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.5 The Refspec](https://git-scm.com/book/en/v2/Git-Internals-The-Refspec)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.6 Transfer Protocols](https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols)
- [Git Official Documentation - git-verify-pack(1) & git-pack-objects(1)](https://git-scm.com/docs/git-verify-pack)
- [Git Official Documentation - Git Protocol v2](https://git-scm.com/docs/protocol-v2)
