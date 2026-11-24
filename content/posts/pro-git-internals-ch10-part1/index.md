+++
date = '2026-12-06T09:00:00+09:00'
draft = false
title = 'Git 내부 구조 #17: Blob·Tree·Commit과 Refs'
description = 'Pro Git 10장 1부(10.1~10.3)를 바탕으로 Blob·Tree·Commit·Tag 객체 바이너리 구조와 Plumbing 수동 커밋, Refs·HEAD·Reftable을 정리한다.'
tags = ['Git', 'ProGit', 'GitInternals', 'GitObjects', 'Plumbing', 'Refs', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#16)](/posts/pro-git-internals-ch8-part2/)에서는 로컬과 서버를 아우르는 Git Hooks와 `pre-commit` 프레임워크 기반의 2중 품질 방어선을 다루며 8장을 마쳤다. 참고로 Pro Git 9장(Git and Other Systems)은 SVN·Mercurial·Perforce 등 레거시 VCS 마이그레이션을 다루는 장이므로 건너뛰고, 이번 글부터 전체 시리즈의 최종 관문인 **Pro Git 10장(Git Internals) 3부작(`#17`~`#19`)** 으로 들어간다.

1장부터 8장까지 살펴본 스냅샷 모델, 브랜치 포인터, 태그, Reset, Stash가 `.git/` 디렉토리 안에서 실제로 어떤 바이트와 파일 구조로 움직이는지 궁금했다면, **10장 1부(`#17`, Pro Git 10.1~10.3)** 가 그 모든 퍼즐을 맞추는 핵심 열쇠다. 이번 편에서는 특히 Git 저장소의 심장부인 **4가지 Git Object(`Blob`, `Tree`, `Commit`, `Tag`)의 바이너리 구조와 Merkle DAG 원리**를 바닥까지 파헤친다.

---

## 1. Porcelain vs Plumbing과 `.git` 디렉토리 해부 (10.1)

우리가 일상적으로 쓰는 `git add`, `git commit`, `git switch`, `git merge` 같은 명령은 사용자 친화적인 도자기(Porcelain)에 비유해 **Porcelain 명령어**라고 부른다. 반면 Git 초기 개발자들이 바닥 배관 공사용으로 만들어 둔 `git hash-object`, `git cat-file`, `git update-index`, `git write-tree`, `git commit-tree`, `git update-ref` 같은 저수준 명령어는 **Plumbing(배관) 명령어**라 부른다.

이 계층 분리가 시사하는 바는 명확하다. **Git의 본질은 "버전 관리 도구"가 아니라 "콘텐츠 주소 지정 파일시스템(Content-Addressable Filesystem)"이고, 그 위에 버전 관리 UI를 얹었을 뿐이다.**

`git init` 직후 생성되는 `.git/` 디렉토리를 열어 보면 그 뼈대가 그대로 드러난다.

| 항목 | 유형 | 역할 및 내부 동작 |
| :--- | :--- | :--- |
| **`HEAD`** | 파일 | 현재 체크아웃된 브랜치(`ref: refs/heads/main`) 또는 커밋 해시(Detached HEAD) 저장 |
| **`objects/`** | 디렉토리 | Git의 모든 데이터(Blob, Tree, Commit, Annotated Tag, Packfile)가 저장되는 객체 DB |
| **`refs/`** | 디렉토리 | 브랜치(`heads/`), 태그(`tags/`), 원격 추적(`remotes/`) 포인터(41바이트 파일) 저장소 |
| **`index`** | 파일 | Staging Area의 실제 바이너리 파일. 다음 커밋에 들어갈 Tree의 초안 유지 |
| **`config`** | 파일 | 해당 저장소 전용 설정(`.git/config`, `#15`에서 다룬 Local 설정 계층) |
| **`hooks/`** | 디렉토리 | 클라이언트·서버 이벤트 훅 스크립트(`#16` 참조) |
| **`info/`** | 디렉토리 | 추적하지 않을 개인 무시 패턴(`.git/info/exclude`) 및 팩 메타정보 저장 |
| **`description`** | 파일 | GitWeb 등 레거시 웹 UI용 저장소 설명 문자열 |

---

## 2. Git Objects 심층 해부: 바이트에서 Merkle DAG까지 (10.2)

Git 객체 데이터베이스(`.git/objects/`)는 키-값(Key-Value) 저장소다. 사용자가 임의의 데이터(파일 내용, 디렉토리 구조, 커밋 메타데이터)를 집어넣으면, Git은 그 콘텐츠를 바탕으로 **유일한 40자리 SHA-1 해시 키**를 발급해 돌려준다. 나중에 그 키만 제시하면 언제든 원래 데이터를 1바이트의 오차도 없이 꺼낼 수 있다.

{{< figure src="git-object-storage-pipeline.svg" alt="Git Object 바이너리 저장 파이프라인과 Blob, Tree, Commit, Annotated Tag 4가지 객체 구조 다이어그램" caption="원본 데이터에 타입과 크기를 붙인 뒤 SHA-1 해싱 및 zlib 압축을 거쳐 .git/objects에 저장하는 파이프라인(상단)과 4대 객체 구조(하단) (Pro Git 10.2 재구성)" width="820" height="580" >}}

### 2.1 바이너리 저장 원리: 헤더 + 널 바이트 + zlib 압축

Git이 데이터를 객체로 저장할 때 디스크에는 정확히 다음 규칙을 거친 바이트가 기록된다.

1. **헤더 생성**: `<객체 타입> <원본 바이트 크기>\0` 문자열을 원본 데이터 앞에 붙인다. 예를 들어 17바이트짜리 문자열 `"what is up, doc?\n"`을 Blob으로 저장한다면 Git이 실제로 조립하는 전체 바이트는 **`blob 17\0what is up, doc?\n`** 이다. 여기서 널 바이트(`\0`, `0x00`)는 ASCII 텍스트 헤더와 뒤따르는 임의의 바이너리 페이로드를 가르는 절대적인 경계선 역할을 한다.
2. **SHA-1 해싱**: 헤더가 결합된 전체 바이트 시퀀스에 SHA-1 해시 함수를 적용해 160비트(16진수 40자리) 체크섬을 계산한다.
3. **zlib 압축 및 샤딩 저장**: 헤더를 포함한 전체 바이트를 zlib deflate 알고리즘으로 압축한 뒤, 해시의 **앞 2자리를 디렉토리 이름**, **나머지 38자리를 파일 이름**으로 삼아 `.git/objects/71/08f7ecb345ee9d0084193f147cdad4d2998293` 경로에 저장한다.

앞 2자리(`00`~`ff`, 최대 256개 버킷)로 폴더를 쪼개는 이유는 실용적이다. 수십만 개의 객체 파일이 단일 디렉토리에 몰리면 OS 파일시스템(ext4, APFS, NTFS 등)의 디렉토리 탐색과 inode 조회 성능이 급격히 떨어지기 때문이다.

터미널에서 `git hash-object`를 호출한 뒤 Python의 표준 `hashlib`·`zlib` 라이브러리만으로 같은 계산을 돌리면 **단 1바이트의 차이도 없이 정확히 동일한 해시**가 나온다.

```bash
$ printf 'what is up, doc?\n' | git hash-object -w --stdin
7108f7ecb345ee9d0084193f147cdad4d2998293

$ python3 -c "
import hashlib, zlib
content = b'what is up, doc?\n'
header = f'blob {len(content)}\0'.encode()
store = header + content
print('SHA-1:', hashlib.sha1(store).hexdigest())
print('Compressed bytes:', len(zlib.compress(store)))
"
SHA-1: 7108f7ecb345ee9d0084193f147cdad4d2998293
Compressed bytes: 33
```

이 단순한 설계 덕분에 Git은 두 가지 강력한 속성을 공짜로 얻는다.

- **내용 기반 중복 제거(Deduplication)**: 서로 다른 폴더에 똑같은 라이선스 파일이나 이미지가 100개 들어 있어도, 내용이 같으면 SHA-1 해시가 같으므로 `.git/objects`에는 단 1개의 Blob 파일만 저장된다.
- **불변성(Immutability)과 SHA-256 전환**: 객체는 한 번 디스크에 쓰이면 절대 제자리 수정되지 않는다. 내용이 1비트만 바뀌어도 완전히 다른 해시 주소에 새 객체로 기록된다. 한편 2017년 SHAttered 충돌 공격 이후 학계에서 SHA-1의 충돌 저항성이 깨지면서, 현대 Git(`2.29+`, 정식 지원 안정화 `2.45+`)은 64자리 해시를 쓰는 **`git init --object-format=sha256`** 도 지원해 미래의 충돌 공격에 대비하고 있다.

---

### 2.2 4가지 Git Object 타입 완전 분해

Git의 객체 DB 안에는 오직 **네 종류의 객체**만 존재한다. 우리가 아는 브랜치도, 커밋 이력도, 디렉토리 계층도 결국 이 네 객체의 조합이다.

#### ① Blob (`blob`) — 오직 내용만 기억하는 순수 바이트 덩어리

**Blob(Binary Large Object)** 은 파일의 원시 바이트 내용만 저장한다. 놀랍게도 Blob 객체 내부에는 **파일명도, 폴더 경로도, 파일 실행 권한도, 수정 시각(mtime)도 일절 들어 있지 않다.** 텍스트 소스 코드든 PNG 이미지든 PDF든 상관없이 그저 바이트 열일 뿐이다.

이 때문에 `git mv old_name.py new_name.py`로 파일 이름을 바꾸거나 같은 파일을 다른 경로에 복사하더라도 Git은 **새로운 Blob을 전혀 만들지 않는다.** 오직 파일 이름과 경로 목록을 관리하는 상위 Tree 객체만 새로 하나 생길 뿐이다.

#### ② Tree (`tree`) — 디렉토리 스냅샷과 Merkle DAG의 핵심

Blob이 파일 내용만 저장한다면 "이 파일의 이름이 `main.py`이고 실행 권한을 가졌다"는 사실은 누가 기억할까? 바로 **Tree 객체**다. 유닉스 파일시스템의 디렉토리(Directory Entry)에 대응하는 개념으로, 한 폴더 안에 있는 하위 Blob들과 하위 디렉토리(또 다른 Tree) 목록을 한 줄씩 매핑한다.

Tree 엔트리의 각 줄은 `[파일 모드(mode)] [객체 타입] [SHA-1 해시] [파일/폴더명]` 구조를 가진다. 이때 `mode`에는 유닉스 스타일의 6자리 8진수 코드가 들어간다.

- **`100644`**: 일반 파일(읽기/쓰기, 실행 권한 없음)
- **`100755`**: 실행 가능 파일(쉘 스크립트, 바이너리 등에 `chmod +x` 적용 시)
- **`120000`**: 심볼릭 링크(이때 연결된 Blob은 대상 파일 자체가 아니라 링크 경로 문자열을 저장)
- **`040000`**: 하위 디렉토리를 가리키는 또 다른 `tree` 객체
- **`160000`**: Git 서브모듈을 가리키는 Gitlink([7장 4부(#14)](/posts/pro-git-internals-ch7-part4/)에서 다룬 커밋 참조 엔트리)

*(참고로 사람이 `git cat-file -p`로 볼 때는 위처럼 텍스트로 예쁘게 출력되지만, 실제 디스크의 Tree 바이너리는 파싱 속도와 공간 효율을 위해 `[mode] [이름]\0[20바이트 raw SHA-1]` 형식으로 정렬해 압축 저장한다.)*

여기서 Tree 객체가 만드는 자료구조가 바로 **Merkle DAG(머클 방향성 비순환 그래프)** 다. Tree는 자식으로 하위 Tree를, 하위 Tree는 또 다른 Blob을 가리키며 전체 프로젝트 스냅샷 트리를 이룬다. 이 구조의 위력은 두 가지다.

1. **연쇄 무결성 검증**: 깊은 하위 경로 `src/auth/jwt.py`에서 **단 1바이트만 수정**해도 `jwt.py`의 Blob 해시가 바뀌고, 그것을 품은 `auth` Tree 해시가 바뀌고, `src` Tree와 최상위 루트 Tree, 최종 Commit 해시까지 도미노처럼 연쇄 변경된다. 과거 커밋 한 줄을 몰래 고치는 것이 원천 불가능한 이유다.
2. **초고속 스냅샷 비교(`O(1)` 서브트리 스킵)**: 1만 개의 파일이 있는 프로젝트에서 `git diff`나 `git merge`를 실행할 때 Git은 모든 파일을 다 열어보지 않는다. 하위 폴더 Tree의 SHA-1 해시 20바이트가 이전 커밋과 똑같으면 **"이 폴더 아래 수천 개 파일은 1비트도 안 변했다"** 고 확신하고 그 서브트리 전체를 즉시 건너뛴다.

#### ③ Commit (`commit`) — 스냅샷에 맥락과 시간을 입히는 메타데이터

Tree 객체만으로는 "누가, 언제, 왜, 어떤 커밋의 뒤를 이어" 이 스냅샷을 찍었는지 알 수 없다. **Commit 객체**가 이 역할을 맡는다.

Commit 객체의 페이로드는 놀라울 정도로 간결한 평문 텍스트다.

- **`tree <해시>`**: 이 커밋 시점의 프로젝트 전체를 대표하는 최상위 루트 Tree 딱 1개.
- **`parent <해시>`**: 직전 부모 커밋의 해시. 최초(Root) 커밋에는 `parent`가 0개, 일반 커밋에는 1개, 머지 커밋에는 2개 이상(Octopus Merge 시 3개 이상)의 `parent` 줄이 들어간다.
- **`author` / `committer`**: 이름, 이메일, Unix epoch 초 단위 타임스탬프와 타임존(`+0900`). 실제 코드를 짠 원작자(`author`)와 리베이스·체리픽·패치 적용을 수행한 사람(`committer`)이 분리되어 있어 이 두 값이 달라질 수 있다.
- **`gpgsig`** *(선택)*: GPG 또는 SSH 커밋 서명([7장 1부(#11)](/posts/pro-git-internals-ch7-part1/) 참조)이 활성화된 경우 포함되는 암호화 서명 블록.
- **빈 줄 다음의 커밋 메시지**: 제목과 본문.

덕분에 커밋이 10만 개 쌓여도 각 Commit 객체 자체는 수백 바이트에 불과하며, 변경되지 않은 파일과 폴더의 Blob·Tree는 이전 커밋들과 그대로 공유한다.

#### ④ Annotated Tag (`tag`) — 릴리즈를 위한 불변의 꼬리표 객체

3부작의 [2장 2부(#3)](/posts/pro-git-internals-ch2-part2/)에서 보았듯 Git 태그에는 두 종류가 있다. 브랜치처럼 커밋 해시만 적어 둔 Lightweight Tag와 달리, **Annotated Tag(`git tag -a`)는 독립된 4번째 객체 타입 `tag`를 생성한다.**

내부에는 대상 객체(`object <해시>`), 대상 타입(`type commit`), 태그 이름(`tag v1.0.0`), 작성자(`tagger`), 릴리즈 노트 메시지, 그리고 선택적 암호화 서명이 담긴다. 따라서 릴리즈 서명이 필요한 프로덕션 배포에서는 반드시 Annotated Tag 객체를 만들어야 한다.

#### 4가지 Git Object 한눈에 비교하기

| 객체 타입 | 저장하는 핵심 정보 | 파일명·권한 포함 | 가리키는 대상(하위 참조) | 대표 Plumbing 생성 명령 |
| :--- | :--- | :--- | :--- | :--- |
| **`blob`** | 파일 원시 바이트 내용 | 미포함 | 없음 (리프 노드) | `git hash-object -w` |
| **`tree`** | 파일명·mode·하위 객체 목록 | **포함** (`Tree`가 관리) | `blob`, 하위 `tree` | `git write-tree` |
| **`commit`** | 루트 Tree, 부모 커밋, 작성자, 메시지 | 미포함 | 루트 `tree` 1개, `parent` 0개+ | `git commit-tree` |
| **`tag`** | 태그명, Tagger, 릴리즈 메시지, 서명 | 미포함 | 주로 `commit` (임의 객체 가능) | `git tag -a` / `mktag` |

---

### 2.3 [터미널 실습] `git add` · `git commit` 없이 수동으로 커밋 2개 만들기

이제 고수준 Porcelain 명령(`git add`, `git commit`)을 단 한 번도 치지 않고, 오직 Plumbing 명령어만 조립해 두 개의 커밋과 하위 폴더(`bak/`)까지 갖춘 저장소 이력을 직접 만들어 보자.

#### 1단계: Blob 2개 생성 후 `cat-file`로 역압축 확인하기

```bash
$ mkdir plumbing-lab && cd plumbing-lab && git init -b main

# working tree에 파일조차 만들지 않고 표준 입력으로 직접 Blob 2개 저장
$ printf 'version 1\n' | git hash-object -w --stdin
83baae61804e65cc73a7201a7252750c76066a30

$ printf 'version 2\n' | git hash-object -w --stdin
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a

# cat-file로 타입(-t), 원본 바이트 크기(-s), 압축 해제 내용(-p) 확인
$ git cat-file -t 83baae61804e65cc73a7201a7252750c76066a30
blob
$ git cat-file -s 83baae61804e65cc73a7201a7252750c76066a30
10
$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30
version 1
```

#### 2단계: Staging Area(`update-index`)에 등록해 첫 Tree와 Commit 만들기

```bash
# Blob(83baae6...)을 test.txt라는 이름(mode 100644)으로 Index에 등록
$ git update-index --add --cacheinfo 100644 \
  83baae61804e65cc73a7201a7252750c76066a30 test.txt

# 현재 Index 상태를 Tree 객체로 굽기
$ git write-tree
d8329fc1cc938780ffdd9f94e0d364e0ea74f579

# Tree 객체 내용 들여다보기
$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579
100644 blob 83baae61804e65cc73a7201a7252750c76066a30	test.txt

# 루트 Tree를 감싸는 첫 번째 Commit 객체 생성
$ echo 'first commit' | git commit-tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
d695e7464f4cb84b8d70d9f98b24a07f597d568f

$ git cat-file -p d695e7464f4cb84b8d70d9f98b24a07f597d568f
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
author Goru <goru@example.com> 1796515200 +0900
committer Goru <goru@example.com> 1796515200 +0900

first commit
```

*(참고: 실제 터미널에서는 사용자의 `user.name`과 현재 시각에 따라 Commit 해시가 달라진다. 위 `d695e74...`는 작성자 이름·시각을 고정해 재현한 예시 해시다.)*

#### 3단계: `read-tree --prefix`로 중첩 Tree 만들고 두 번째 커밋 잇기

이번에는 `test.txt`를 `version 2` Blob(`1f7a7a4...`)으로 교체하고, 아까 만든 첫 번째 Tree(`d8329fc...`) 전체를 `bak/` 하위 폴더로 집어넣은 뒤 부모 커밋(`-p`)을 연결해 보자.

```bash
# 1. test.txt를 version 2 Blob으로 갱신
$ git update-index --add --cacheinfo 100644 \
  1f7a7a472abf3dd9643fd615f6da379c4acb3e3a test.txt

# 2. 기존 Tree를 Index의 bak/ 접두사 아래에 하위 트리로 읽어들이기
$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579

# 3. 두 번째 Tree 생성 및 내부 확인 (040000 tree 엔트리가 생겼다!)
$ git write-tree
b9c6a44acc8cf4303f3b8a7520e15df999e6057d

$ git cat-file -p b9c6a44acc8cf4303f3b8a7520e15df999e6057d
040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579	bak
100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a	test.txt

# 4. 직전 커밋을 부모(-p)로 지정해 두 번째 Commit 객체 생성
$ echo 'second commit' | git commit-tree b9c6a44acc8cf4303f3b8a7520e15df999e6057d \
  -p d695e7464f4cb84b8d70d9f98b24a07f597d568f
de47e47e5c5539ede68a21170ee6b13be4952084
```

여기까지 마치고 `git log --stat de47e47`을 쳐 보면 `git add`와 `git commit`으로 만든 것과 토씨 하나 다르지 않은 완벽한 커밋 히스토리가 출력된다.

---

## 3. Git References와 `HEAD` 심볼릭 참조 (10.3)

아무리 훌륭한 객체 그래프를 만들었어도 사람이 40자리 해시(`de47e47e5c5539ede68a21170ee6b13be4952084`)를 외워서 작업할 수는 없다. 그래서 Git은 마지막 커밋의 SHA-1 해시를 사람이 읽을 수 있는 이름에 적어 둔 **참조(Reference, 줄여서 Ref)** 파일을 `.git/refs/` 아래에 관리한다.

{{< figure src="git-object-graph-and-refs.svg" alt="HEAD 심볼릭 참조와 Detached HEAD, refs/heads, refs/tags, refs/remotes가 Commit, Tree, Blob 객체 그래프를 가리키는 구조와 Loose, packed-refs, Reftable 참조 저장소 진화 다이어그램" caption="HEAD 심볼릭 참조와 3종류 Ref가 Object Graph를 가리키는 구조(상단) 및 참조 저장소 백엔드의 진화(하단) (Pro Git 10.3 재구성)" width="820" height="520" >}}

### 3.1 `git update-ref`로 안전하게 브랜치 연결하기

방금 수동으로 만든 두 번째 커밋(`de47e47...`)은 아직 어떤 브랜치도 가리키고 있지 않다. 이 커밋을 `main` 브랜치로 삼으려면 `.git/refs/heads/main` 파일에 40자리 해시와 줄바꿈(총 41바이트)을 기록하면 된다.

실무나 스크립트에서 파일을 `echo`로 덮어쓰는 대신 반드시 **`git update-ref`** 를 써야 하는 이유는, 이 명령이 동시성 잠금(`.lock` 파일)과 Reflog 기록, 그리고 `packed-refs` 정합성까지 원자적으로 처리해 주기 때문이다.

```bash
# main 브랜치를 방금 만든 두 번째 커밋에 연결
$ git update-ref refs/heads/main de47e47e5c5539ede68a21170ee6b13be4952084

$ git log --stat --oneline
de47e47 (HEAD -> main) second commit
 bak/test.txt | 1 +
 test.txt     | 2 +-
 2 files changed, 2 insertions(+), 1 deletion(-)
d695e74 first commit
 test.txt | 1 +
 1 file changed, 1 insertion(+)
```

마찬가지로 원격 추적 브랜치(`.git/refs/remotes/origin/main`, 읽기 전용 스냅샷)나 Lightweight 태그(`.git/refs/tags/v0.1`) 역시 모두 같은 구조의 Ref 파일이다.

### 3.2 `HEAD` 심볼릭 참조(`symbolic-ref`)와 Detached HEAD

`.git/HEAD` 파일은 일반적인 Ref와 달리 SHA-1 해시 대신 **"다른 Ref의 경로"** 를 담는 **심볼릭 참조(Symbolic Reference)** 인 경우가 대부분이다.

```bash
# 현재 HEAD가 가리키는 심볼릭 대상 확인
$ git symbolic-ref HEAD
refs/heads/main

# 특정 브랜치(예: topic)로 HEAD가 가리키는 대상 안전하게 바꾸기
$ git symbolic-ref HEAD refs/heads/topic
```

- **정상 브랜치 상태**: `.git/HEAD` 파일에 `ref: refs/heads/main`이 들어 있다. 이때 커밋을 하면 Git은 `refs/heads/main`이 가리키던 커밋을 부모로 삼고, `refs/heads/main` 파일의 해시만 새 커밋으로 전진시킨다.
- **Detached HEAD 상태**: `git checkout <커밋해시>`나 태그를 체크아웃하면 `.git/HEAD` 안에 `ref:` 없이 **40자리 SHA-1 해시가 직접 기록**된다. 어떤 브랜치 이름에도 매달려 있지 않은 고립 상태이므로, 여기서 만든 새 커밋을 잃지 않으려면 `git switch -c <새브랜치>`로 Ref를 붙여 주어야 한다.

### 3.3 참조 저장소의 현대적 진화: Loose → `packed-refs` → Reftable

전통적인 `.git/refs/` 디렉토리 방식은 직관적이지만, 수만 개의 태그·브랜치가 생기는 대형 프로젝트나 모노레포에서는 두 가지 치명적 약점을 드러냈다.

1. **파일시스템 I/O 병목과 전체 재작성 부담**: 브랜치 수만큼 자잘한 파일이 쏟아지자 Git은 이를 `.git/packed-refs`라는 단일 텍스트 파일로 압축하는 기법을 도입했다. 하지만 브랜치 하나만 지워도 거대한 `packed-refs` 파일 전체를 다시 써야 하는 한계가 남았다.
2. **macOS·Windows 대소문자 미구분 충돌**: `refs/heads/Feature`와 `refs/heads/feature`는 리눅스(ext4)에서는 두 개의 다른 파일이지만, 기본 macOS(APFS Case-Insensitive)나 Windows(NTFS)에서는 같은 파일로 덮어써져 저장소가 꼬이는 고질병이 있었다.

이를 근본적으로 해결하기 위해 **Git 2.45+** 부터 정식 실험적 백엔드로 탑재된 포맷이 바로 **Reftable(`--ref-format=reftable`)** 이다. Reftable은 참조를 OS 디렉토리 파일이 아닌 정렬된 바이너리 테이블(`.git/reftable/*.ref`)로 관리함으로써:
- 수십만 개의 참조에서도 `O(log N)` 이진 탐색으로 즉시 조회하고,
- 여러 브랜치를 한꺼번에 갱신할 때 **원자적(Atomic) 트랜잭션**을 보장하며,
- OS 파일시스템의 대소문자·특수문자 제약에서 완전히 자유롭다.

---

## 마치며

이번 **10장 1부(`#17`)** 에서는 고수준 명령 뒤에 숨어 있는 Plumbing 계층과 `.git/objects/` 의 세계를 해부했다.

- 모든 데이터는 `<type> <size>\0<content>` 바이트를 SHA-1 해싱하고 zlib으로 압축해 저장하는 콘텐츠 주소 지정 파일시스템 위에 놓여 있다.
- **Blob**(내용) · **Tree**(경로/권한 및 Merkle DAG) · **Commit**(스냅샷 이력) · **Tag**(릴리즈 메타데이터) 네 객체의 조합만으로 버전 관리 전체가 성립한다.
- **`HEAD`와 `.git/refs/`** 는 사람이 읽을 수 있는 이름을 40자리 해시에 매달아 주는 가벼운 포인터이며, 현대 Git에서는 대규모 모노레포를 위해 **Reftable**로 진화하고 있다.

다음 **[10장 2부(#18)](/posts/pro-git-internals-ch10-part2/)** 에서는 "모든 버전을 통째로 스냅샷으로 저장하는데도 Git 저장소 용량이 왜 이렇게 작을까?"라는 의문을 풀어 주는 **Packfile의 역방향 델타 압축 반전**, **`+<src>:<dst>` Refspec 규칙**, 그리고 **Smart Protocol의 실제 패킷 협상 과정**을 다룬다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.1 Plumbing and Porcelain](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.2 Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 10.3 Git References](https://git-scm.com/book/en/v2/Git-Internals-Git-References)
- [Git Official Documentation - git-hash-object(1) & git-cat-file(1)](https://git-scm.com/docs/git-cat-file)
- [GitLab Engineering Blog - A beginner's guide to the Git reftable format](https://about.gitlab.com/blog/2024/05/30/a-beginners-guide-to-the-git-reftable-format/)
