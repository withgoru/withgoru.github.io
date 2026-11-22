+++
date = '2026-11-26T09:00:00+09:00'
draft = false
title = '[Pro Git 톺아보기 #7] Git 서버의 내부 원리(Bare 저장소·4대 프로토콜·git-shell)와 현대 GitHub이 HTTPS를 권장하는 이유'
description = 'Pro Git 4장(Git on the Server) 전체를 현대적 관점으로 재해석하여 Bare 저장소의 본질, Dumb HTTP에서 Smart HTTP(git-http-backend)로의 진화, 단일 git 계정과 git-shell의 보안 구조, 그리고 현대 GitHub이 HTTPS와 Fine-grained 토큰을 권장하는 이유까지 깊게 정리한다.'
tags = ['Git', 'ProGit', 'VCS', 'Internals', 'GitHub', 'Security', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 이론편(#6)]({{< ref "/posts/pro-git-internals-ch3-part3" >}})과 [1~3장 종합 실습편]({{< ref "/posts/pro-git-hands-on-ch1-to-ch3" >}})까지 살펴본 내용만으로도 일상적인 Git 작업은 대부분 수행할 수 있다. 하지만 본격적인 협업을 하려면 개발자 모두가 접근할 수 있는 공통 원격 저장소(Git Server)가 필요하다. 누군가의 개인 노트북에 직접 Push·Pull하면 그 사람이 컴퓨터를 끄는 순간 협업이 멈추고 현재 작업 중인 상태까지 뒤섞이기 때문이다.

**Pro Git 4장**(Git on the Server)은 바로 이 Git 서버가 내부적으로 어떻게 동작하는지 다룬다. 다만 2판이 집필된 2014년 당시의 레거시 환경(`git://` 데몬, `git instaweb`, 수동 Apache CGI 설정, 구형 RSA 키 등)을 그대로 따라 하기보다는, 오늘날의 **GitHub · GitLab · 사내 엔터프라이즈 Git 서버**를 지탱하는 변하지 않는 핵심 원리만 뽑아내고 **왜 현대 GitHub이 SSH보다 HTTPS(Smart HTTP)를 기본 권장하는지** 현대 보안·인프라 관점까지 엮어 정리해 본다.

---

## 1. 서버 저장소의 본질: 왜 Working Tree가 없는 Bare 저장소(`--bare`)인가? (Pro Git 4.2)

내 PC에서 개발하던 일반 Git 저장소와 서버에 올려 두는 Git 서버 저장소의 결정적인 차이는 단 하나다. 서버에 두는 원격 저장소는 작업 디렉토리(Working Directory)가 전혀 없는 **Bare 저장소**(Bare Repository)다.

일반 저장소에 누군가 원격으로 `git push`를 시도하면 Git이 기본적으로 이를 거부한다(`receive.denyCurrentBranch`). 누군가 그 서버 디렉토리에서 체크아웃해 놓은 파일(Working Tree)과 `HEAD`의 브랜치 포인터가 외부 푸시로 인해 갑자기 불일치해지는 데이터 훼손을 막기 위해서다. 서버 저장소는 개발자가 직접 코드를 편집하는 공간이 아니라 **팀원 간 스냅샷을 주고받는 중계 지점**이므로, 굳이 디스크에 소스 코드를 펼쳐 놓을 이유가 없다.

기존 프로젝트를 서버용 Bare 저장소로 내보낼 때는 `--bare` 옵션을 붙이며, 관례상 디렉토리 이름 끝에 `.git` 접미사를 붙인다.

```bash
# 기존 저장소의 .git 데이터만 추출해 Bare 저장소 생성
$ git clone --bare my_project my_project.git
Cloning into bare repository 'my_project.git'...
done.
```

이는 사실상 `$ cp -Rf my_project/.git my_project.git`을 실행해 프로젝트 루트 안에 있던 `.git` 디렉토리 알맹이만 밖으로 꺼내 놓은 것과 거의 같다(내부 `config`에 `bare = true`가 기록되는 차이만 존재한다).

만약 Linux 서버(`/srv/git`)를 두고 소규모 팀이 함께 SSH로 Push하도록 구성하려면, 복사한 Bare 저장소에 그룹 쓰기 권한을 열어 주는 `--shared` 옵션만으로 이미 온전한 협업 서버가 완성된다.

```bash
$ scp -r my_project.git user@git.example.com:/srv/git
$ ssh user@git.example.com "cd /srv/git/my_project.git && git init --bare --shared"
```

---

## 2. 4대 전송 프로토콜 해부: Dumb HTTP에서 Smart HTTP로의 진화 (Pro Git 4.1, 4.5, 4.6)

Git은 네트워크와 로컬 환경에서 데이터를 전송하기 위해 **Local, HTTP(Dumb / Smart), SSH, Git(`git://`)** 네 가지 프로토콜을 지원한다.

{{< figure src="protocols-and-smart-http.svg" alt="Git Dumb HTTP와 Smart HTTP 동작 비교 및 4대 전송 프로토콜 아키텍처 다이어그램" caption="정적 파일을 순회하는 Dumb HTTP와 git-http-backend가 upload-pack·receive-pack으로 Packfile을 협상하는 Smart HTTP의 동작 차이 및 4대 프로토콜 비교 (Pro Git 4.1, 4.5, 4.6 재구성)" width="820" height="520" >}}

### Local 프로토콜: 경로 직접 지정 vs `file://`

같은 서버나 공유 파일시스템(NFS 마운트 등)의 다른 디렉토리를 원격으로 삼는 방식이다. 이때 URL 앞에 `file://`을 붙이느냐 생략하느냐에 따라 Git의 내부 실행 경로가 달라진다.

```bash
# 1. 일반 경로: 하드링크 또는 직접 파일 복사로 매우 빠름
$ git clone /srv/git/project.git

# 2. file:// 접두사: 네트워크 전송용 Git 프로세스를 기동해 불필요한 객체 정리(Clean copy) 가능
$ git clone file:///srv/git/project.git
```

설정은 가장 단순하지만, 모든 사용자가 원격 디렉토리에 대한 셸·파일시스템 직접 접근 권한을 가지므로 실수로 `.git` 내부 파일을 훼손할 위험이 있고 원격지 접속이 불편하다.

### Dumb HTTP vs Smart HTTP (`git-http-backend`)

HTTP 프로토콜은 **Git 1.6.6**을 기점으로 완전히 다른 시대를 맞았다.

1. **Dumb HTTP (구형 정적 서빙 방식)**: 서버 쪽에 Git 바이너리가 전혀 돌지 않고 Apache·Nginx 같은 정적 웹 서버가 Bare 저장소 파일들을 일반 파일처럼 그대로 노출하는 방식이다. 클라이언트가 `info/refs`와 수많은 느슨한 객체 파일을 직접 `HTTP GET`으로 순회해야 하므로 **읽기 전용(Read-Only)이고 느리며**, 서버에 누군가 Push할 때마다 `hooks/post-update` 훅에서 `git update-server-info`를 실행해 인덱스 파일을 갱신해 주어야 했다.
2. **Smart HTTP (`git-http-backend`)**: Git 1.6.6부터 도입된 현대적 방식이다. 웹 서버 뒷단에서 Git과 함께 배포되는 **`git-http-backend`** CGI 스크립트(또는 현대 호스팅 서버의 동등한 HTTP 핸들러)가 요청을 받아, SSH처럼 클라이언트와 **보유 커밋 목록(`have` / `want`)을 먼저 협상한 뒤 부족한 객체만 하나의 Packfile로 압축해 송수신**한다.

특히 Smart HTTP는 **단 하나의 URL**로 익명 공개 읽기(Read)와 인증 기반 쓰기(Push)를 동시에 처리할 수 있다. Git 내부 전송은 데이터를 내려주는 **`git-upload-pack`**(`clone`/`fetch`)과 데이터를 받아 기록하는 **`git-receive-pack`**(`push`) 두 프로세스로 나뉘기 때문이다. Pro Git 4.6의 Apache 설정 예시에서도 이 분기점이 명확히 드러난다.

```apache
SetEnv GIT_PROJECT_ROOT /srv/git
SetEnv GIT_HTTP_EXPORT_ALL
ScriptAlias /git/ /usr/lib/git-core/git-http-backend/

<Files "git-http-backend">
    AuthType Basic
    AuthName "Git Access"
    AuthUserFile /srv/git/.htpasswd
    Require expr !(%{QUERY_STRING} -strmatch '*service=git-receive-pack*' || %{REQUEST_URI} =~ m#/git-receive-pack$#)
    Require valid-user
</Files>
```

쿼리 문자열이나 URI가 쓰기 요청(`git-receive-pack`)일 때만 인증을 요구하고, 읽기 요청(`git-upload-pack`)은 통과시키는 구조다. 오늘날 GitHub에서 브라우저로 보는 페이지 주소(`https://github.com/user/repo`)를 그대로 `git clone`과 `git push`에 쓸 수 있는 이유가 바로 Smart HTTP 덕분이다. *(참고로 구형 클라이언트 호환성을 위해 클라이언트가 Smart 헤더를 보내지 않으면 Dumb 방식으로 폴백하기도 한다.)*

### 레거시 `git://` 프로토콜(`git daemon`)이 역사 속으로 사라진 이유

Pro Git 4.5에서는 9418번 전용 포트로 통신하는 **`git daemon`**(`git://`)을 비중 있게 다룬다. 저장소 내부에 `git-daemon-export-ok`라는 빈 파일만 두면 암호화·인증 오버헤드 없이 가장 빠르게 익명 클론을 제공해 주어, 2010년대 중반까지 오픈소스 진영에서 널리 쓰였다.

하지만 TLS 암호화와 서버 신원 검증이 전혀 없기 때문에, **중간 라우터를 장악한 공격자가 `git clone git://...` 응답에 악성 코드를 삽입해 원격 코드 실행(MITM)을 유도**할 수 있다는 치명적 보안 약점이 있었다. 게다가 기업 방화벽에서 비표준 9418번 포트가 차단되는 경우도 많았다. 결국 **GitHub도 2022년 3월 `git://` 프로토콜 지원을 영구 종료**하며 사실상 역사의 뒤안길로 사라졌다.

---

## 3. SSH Git 서버의 내부 구조: 왜 항상 `git@` 계정으로 접속할까? (Pro Git 4.3 ~ 4.4)

우리가 GitHub이나 사내 Git 서버를 SSH로 클론할 때 URL을 유심히 보면 이상한 점을 발견한다. 내 아이디가 `goru`든 `john`이든 상관없이 **전 세계 모든 개발자가 항상 `git@github.com:...`처럼 똑같은 `git` 사용자 이름으로 접속**한다. 어떻게 서버는 `git`이라는 단 하나의 OS 계정만 열어 두고 수많은 개발자를 구별하고 시스템 셸 해킹까지 막는 것일까?

{{< figure src="ssh-vs-modern-https-architecture.svg" alt="전통적 SSH Git 서버(단일 git 계정과 git-shell)와 현대 GitHub Smart HTTPS 아키텍처 비교 다이어그램" caption="단일 git OS 계정·authorized_keys·git-shell로 격리하는 전통 SSH 서버 구조와 GCM·Fine-grained PAT·Port 443 기반의 현대 Smart HTTPS 구조 비교 (Pro Git 4.3~4.9 현대화 재구성)" width="820" height="540" >}}

### ① `authorized_keys` 매핑과 `Ed25519` 표준 키

서버에 개발자마다 리눅스 OS 계정을 하나씩 파주는 대신, 관리자는 서버에 **`git`이라는 공용 OS 계정 하나만 생성**한다. 그리고 개발자들이 보내온 SSH 공개키를 `/home/git/.ssh/authorized_keys` 파일(대규모 서비스는 키 매핑 DB)에 한 줄씩 추가한다. SSH 접속 시 어떤 공개키로 인증을 통과했는지 서버가 식별할 수 있으며, 커밋 기록에 남는 **Author와 Committer 정보는 OS 계정이 아니라 로컬 Git 설정(`user.name`, `user.email`)을 따르므로** 공용 `git` 계정을 써도 커밋 이력에는 전혀 영향을 주지 않는다.

> **현대적 키 생성 (`Ed25519`)**: Pro Git 4.3 집필 당시에는 `ssh-rsa` 키가 예시로 등장하지만, 오늘날 OpenSSH와 GitHub은 더 짧은 키 길이로도 훨씬 높은 보안성과 서명 속도를 제공하는 **`Ed25519`** 알고리즘을 공식 권장한다.
> ```bash
> $ ssh-keygen -t ed25519 -C "goru@example.com"
> ```

### ② `git-shell`과 `authorized_keys` 옵션으로 일반 셸 접속 차단하기

단순히 `git` 계정의 `authorized_keys`에 공개키만 넣어 두면, 개발자가 `ssh git@gitserver`를 입력해 서버의 일반 Bash 셸을 따낼 수 있다. 이를 원천 봉쇄하기 위해 Git은 전용 제한 셸인 `git-shell`을 기본 제공한다.

관리자가 `/etc/shells`에 `git-shell` 경로를 등록한 뒤 `git` 계정의 로그인 셸을 교체(`sudo chsh git -s $(which git-shell)`)하고, `authorized_keys`의 각 공개키 앞에 포트 포워딩·가상 터미널 할당 금지 옵션을 붙인다.

```text
no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... goru@example.com
```

이제 누군가 직접 SSH 셸 로그인을 시도하면 즉시 차단되지만, `git clone`이나 `git push`처럼 원격에서 `git-upload-pack` · `git-receive-pack`을 호출하는 작업은 정상 통과한다.

```bash
$ ssh git@gitserver
fatal: Interactive git shell is not enabled.
hint: ~/git-shell-commands should exist and have read and execute access.
Connection to gitserver closed.
```

---

## 4. 현대적 재해석: 왜 GitHub은 HTTPS(Smart HTTP)를 권장하며 인증은 어떻게 진화했나?

전통적인 셀프 호스팅 환경에서는 SSH가 오랫동안 왕좌를 지켰다. 하지만 **오늘날 GitHub 공식 문서에서는 클론과 푸시의 기본 권장 프로토콜로 SSH가 아니라 HTTPS(Smart HTTP)를 안내**한다. 4장의 프로토콜 원리를 바탕으로 그 이유를 짚어보면 현대 개발 인프라와 보안 모델의 변화가 한눈에 들어온다.

| 비교 항목 | SSH (`git@github.com:...`) | 현대 Smart HTTPS (`https://github.com/...`) |
| :--- | :--- | :--- |
| **기본 포트 및 방화벽** | TCP **22번** 포트 (사내망·공공망·프록시 차단 잦음) | TCP **443번** 포트 (웹 트래픽과 동일해 사실상 100% 통과) |
| **익명 공개 저장소** | 불가 (클론에도 공개키 등록 필요) | **가능** (단일 URL로 익명 클론 + 인증 푸시 모두 지원) |
| **권한 범위 (Blast Radius)** | 기본 SSH 키는 계정 내 **모든 저장소**에 접근 가능 | **Fine-grained PAT**로 특정 저장소·세밀한 권한·만료일 제한 |
| **로컬 인증 편의성** | 수동 키 생성 및 공개키 업로드 필요 | **GCM / `gh auth login`** 브라우저 OAuth + OS 키체인 자동화 |
| **CI/CD 자동화 친화성** | Deploy Key 또는 개인 키 시크릿 수동 관리 | 작업 단위 자동 생성·만료되는 **`GITHUB_TOKEN` (OIDC)** 활용 |

### ① 네트워크와 방화벽 호환성 (`443` vs `22`)

기업 보안망, 학교 네트워크, 프록시 서버 환경에서는 외부 SSH 접속(Port 22)을 차단하는 경우가 매우 흔하다. (GitHub이 굳이 `ssh.github.com` 443 포트로 우회하는 별도 안내를 두는 이유이기도 하다.) 반면 **HTTPS(Port 443)는 웹 브라우저 통신과 동일한 표준 암호화 포트**이므로 어떤 환경에서든 별도 포트 우회 설정 없이 안정적으로 동작한다.

### ② 계정 비밀번호 폐기(2021.08)와 Zero Trust 토큰 생태계

2014년 Pro Git 4장이 쓰일 당시만 해도 HTTPS의 단점으로 꼽히던 것이 "푸시할 때마다 계정 비밀번호를 쳐야 하는 번거로움"이었다. 그러나 **2021년 8월, GitHub은 Git HTTPS 작업에서의 계정 비밀번호 인증을 전면 폐기**하고 토큰 및 OAuth 중심으로 인증 체계를 완전히 재편했다.

- **넓은 폭발 반경 문제 해결 (Fine-grained PAT)**: 전통적인 사용자 SSH 키 하나가 유출되면 그 계정이 속한 모든 조직과 저장소가 위험에 노출된다. 반면 현대 HTTPS 기반의 **Fine-grained Personal Access Token**은 "*특정 1개 저장소의 `Contents: Read and write` 권한만 허용하고 30일 뒤 자동 만료*"처럼 최소 권한 원칙(Zero Trust)을 강제할 수 있다.
- **매끄러운 UX (GCM & `gh auth`)**: 오늘날 개발자가 매번 긴 토큰 문자열을 복사·붙여넣기할 필요도 없다. Git for Windows 등에 기본 내장된 **Git Credential Manager**(GCM)나 GitHub CLI(`gh auth login`)를 쓰면 브라우저 OAuth와 2FA 인증 한 번으로 단기 자격 증명이 macOS Keychain이나 Windows 자격 증명 관리자에 안전하게 저장된다.
- **CI/CD 환경의 일회성 토큰**: GitHub Actions 같은 파이프라인에서는 SSH 개인키를 시크릿에 박아 둘 필요 없이, 워크플로우 실행 순간에만 발급되었다가 작업 종료 즉시 폐기되는 HTTPS 기반 `GITHUB_TOKEN`을 네이티브로 사용한다.

### ③ 레거시 호스팅(4.7 GitWeb ~ 4.9 GitLab)과 현대 플랫폼의 본질

Pro Git 4장의 후반부(4.7~4.9)는 웹 브라우저로 저장소를 열람하는 `git instaweb`(GitWeb)부터 셀프 호스팅 `GitLab`, 그리고 GitHub·Bitbucket 같은 호스팅 서비스를 차례로 소개한다. 당시의 WEBrick 기반 `git instaweb`이나 수동 Apache 설정을 오늘날 실무에서 직접 띄울 일은 거의 없지만, 아키텍처의 본질만큼은 단 한 걸음도 변하지 않았다.

오늘날 우리가 매일 쓰는 GitHub, GitLab, Gitea 같은 거대한 플랫폼 역시 가장 아래층에는 여전히 **Working Tree 없는 Bare 저장소**, SSH 접속을 처리하는 **단일 `git` 게이트웨이**, 그리고 HTTPS 요청을 협상하는 **`git-http-backend` 스타일의 Smart HTTP 엔진**이 자리 잡고 있다. 그 위에 웹 기반 인증·Fine-grained 권한 제어, Pull Request, 코드 리뷰, CI/CD 파이프라인 레이어가 결합되었을 뿐이다.

---

## 마치며

이번 편에서는 **Pro Git 4장**(Git on the Server)을 통해 로컬 저장소 바깥의 서버 세계가 어떤 부품으로 조립되는지 살펴보았다.

- 서버의 원격 저장소는 체크아웃 충돌을 막기 위해 Working Tree를 없앤 **Bare 저장소**(`--bare`)다.
- 전송 계층은 정적 파일을 훑던 **Dumb HTTP**에서 `git-upload-pack`과 `git-receive-pack`을 동적 협상하는 **Smart HTTP**로 진화했고, 무인증 `git://` 데몬은 보안 위험 속에 역사 속으로 퇴장했다.
- SSH 서버는 `authorized_keys`와 `git-shell`을 통해 단일 `git` OS 계정으로 수많은 개발자를 격리하고, 현대 GitHub은 방화벽 호환성(`443`)과 최소 권한 토큰(`Fine-grained PAT`, GCM, `gh auth`)을 극대화하기 위해 **Smart HTTPS**를 기본으로 권장한다.

다음 편(`[Pro Git 톺아보기 #8]`)에서는 드디어 Pro Git **5장**(Distributed Git)으로 넘어가, 오픈소스·엔터프라이즈 팀에서 실제로 코드를 주고받는 **분산 워크플로우**(Centralized · Integration-Manager · Dictator & Lieutenants)와 대규모 기여를 관리하는 실전 전략을 다룬다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 4.1 Git on the Server - The Protocols](https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 4.2 Git on the Server - Getting Git on a Server](https://git-scm.com/book/en/v2/Git-on-the-Server-Getting-Git-on-a-Server)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 4.4 Git on the Server - Setting Up the Server](https://git-scm.com/book/en/v2/Git-on-the-Server-Setting-Up-the-Server)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 4.6 Git on the Server - Smart HTTP](https://git-scm.com/book/en/v2/Git-on-the-Server-Smart-HTTP)
- [GitHub Docs - About remote repositories (HTTPS vs SSH recommendation)](https://docs.github.com/en/get-started/git-basics/about-remote-repositories)
- [The GitHub Blog - Improving Git protocol security on GitHub (Removal of git://)](https://github.blog/security/application-security/improving-git-protocol-security-github/)
- [The GitHub Blog - Token authentication requirements for Git operations](https://github.blog/changelog/2021-08-12-git-password-authentication-is-shutting-down/)
