+++
date = '2026-11-29T09:00:00+09:00'
draft = false
title = 'Git 원격 저장소 비교 #10: GitHub·GitLab·셀프호스팅'
description = 'Pro Git 6장을 현대적으로 확장해 GitHub·GitLab·Bitbucket 3대 클라우드 서비스와 Gitea·Forgejo·OneDev 셀프호스팅 플랫폼의 아키텍처와 선택 기준을 비교한다.'
tags = ['Git', 'ProGit', 'GitHub', 'GitLab', 'Bitbucket', 'Gitea', 'Forgejo', 'OneDev', 'SelfHosted', 'DevOps', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#9)](/posts/pro-git-internals-ch5-part2/)을 끝으로 Git의 분산 협업 워크플로우와 메인테이너의 프로젝트 운영 원리(**Pro Git 5장**)를 마쳤다. 이제 책의 순서대로라면 **Pro Git 6장(GitHub)** 을 다룰 차례다.

그런데 막상 6장 원문을 펼쳐 보면 전체 분량의 상당 부분이 **GitHub 무료 계정 가입하기, 2단계 인증 켜기, Fork 버튼과 Pull Request 화면 클릭하기** 등 2014년 기준의 웹 UI 튜토리얼로 채워져 있다. 이미 1~5장에서 내부 원리와 CLI 협업 흐름을 탄탄히 닦아 온 우리에게는 화면 버튼 설명을 그대로 따라 적기보다, **6장이 짚고 넘어가는 원격 호스팅 플랫폼의 핵심 내부 메커니즘만 간추리고 오늘날 선택할 수 있는 주요 원격 Git 저장소 서비스(클라우드 vs 셀프호스팅)를 비교·정리하는 편**이 훨씬 실무적인 가치가 크다.

이번 글에서는 먼저 Pro Git 6장의 핵심 구조(`refs/pull/*`, Webhooks)를 짧게 요약한 뒤, 3대 매니지드 서비스인 **GitHub · GitLab · Bitbucket**과 가볍고 강력한 차세대 셀프호스팅 대안인 **Gitea · Forgejo · OneDev**까지 아키텍처와 적합 상황을 깊이 있게 비교해 본다.

---

## 1. Pro Git 6장 원문 핵심 요약과 꼭 알아야 할 내부 메커니즘

Pro Git 6장(*GitHub*)은 크게 다섯 가지 주제(6.1~6.5)를 다룬다.

1. **계정 및 보안 설정(6.1)**: SSH 공개키 등록, 2FA 활성화, 알림(Notifications) 라우팅 규칙
2. **Fork & Pull Request 기여(6.2)**: 오픈소스 저장소 Fork 후 `git commit` · `@멘션` · 코드 리뷰 피드백 반영 (이는 앞서 5장에서 CLI 원리로 이미 다루었다)
3. **프로젝트 관리(6.3)**: PR 병합(Merge / Squash / Rebase), 이메일 회신 연동, 팀 리뷰 게이트 운영
4. **Organization과 Team 권한 관리(6.4)**: 회사나 커뮤니티 단위의 조직 계정을 생성하고 저장소별 Read/Write/Admin 권한을 팀 단위로 묶어 관리하는 모델
5. **스크립팅과 자동화(6.5)**: Webhook 이벤트와 REST/GraphQL API를 결합해 CI/CD 파이프라인 및 사내 봇과 연동하는 원리

이 튜토리얼 성격을 덜어 내고 엔지니어링 관점에서 반드시 기억할 내부 메커니즘은 **PR 특수 참조(`refs/pull/*`)** 와 **Webhook 파이프라인** 두 가지다.

### 서버가 몰래 만들어 두는 PR 특수 참조: `refs/pull/<id>/head` & `merge`

원격 저장소에 `git ls-remote origin`을 실행해 보면 로컬 브랜치(`refs/heads/*`)나 태그(`refs/tags/*`) 외에 **`refs/pull/123/head`** 와 **`refs/pull/123/merge`** 같은 숨은 참조가 생성되어 있는 것을 볼 수 있다.

- **`refs/pull/<id>/head`**: 기여자가 보낸 Pull Request 브랜치의 가장 최신 커밋을 가리킨다. 덕분에 메인테이너는 상대방의 Fork 저장소를 원격지로 등록하지 않고도 `git fetch origin pull/123/head:pr-123` 한 줄로 남의 PR 코드를 즉시 로컬에 내려받아 테스트할 수 있다.
- **`refs/pull/<id>/merge`**: GitHub 서버가 대상 브랜치(`main`)와 PR의 `head` 커밋을 **서버 백그라운드에서 가상으로 3-Way Merge해 보고 미리 만들어 두는 테스트 스냅샷**이다. 웹 UI에서 `"This branch has no conflicts with the base branch"`를 실시간으로 판단해 주고, CI/CD 시스템이 **"이 PR을 실제로 머지했을 때의 코드 상태"** 를 미리 빌드·테스트할 수 있는 비결이 바로 이 `merge` 참조 덕분이다.

---

## 2. 3대 매니지드 원격 저장소 서비스 비교: GitHub vs GitLab vs Bitbucket

오늘날 팀이나 기업이 별도 서버 구축 없이 SaaS 형태로 선택할 수 있는 대표적인 원격 Git 저장소 호스팅 플랫폼은 **GitHub**, **GitLab**, **Bitbucket** 세 곳으로 압축된다. 모두 기본적인 Git 호스팅과 PR/MR(Merge Request), 이슈 트래커를 제공하지만 지향하는 생태계와 워크플로우 철학이 뚜렷하게 다르다.

{{< figure src="cloud-git-platforms-comparison.svg" alt="Pro Git 6장 PR 특수 참조(refs/pull) 및 Webhook 파이프라인과 3대 클라우드 Git 호스팅 플랫폼(GitHub, GitLab, Bitbucket) 비교 다이어그램" caption="PR 숨은 참조(refs/pull/head, merge) 및 Webhook 구조(상단)와 3대 클라우드 원격 저장소 서비스 비교(하단)" width="820" height="540" >}}

### ① GitHub: 글로벌 오픈소스와 AI 개발 생태계의 표준

전 세계 개발자와 오픈소스 생태계의 사실상 표준 플랫폼이다. 저장소 접근성과 커뮤니티 네트워크 효과 면에서 압도적이다.

- **핵심 자동화 (`GitHub Actions`)**: YAML 기반 CI/CD 파이프라인을 제공하며, 수만 개에 달하는 커뮤니티 마켓플레이스 액션을 그대로 조합해 빌드·테스트·배포를 수 분 만에 구성할 수 있다.
- **차별화 강점**: **GitHub Copilot**(AI 페어 프로그래밍), 브라우저 기반 클라우드 IDE(**Codespaces**), 의존성 취약점 자동 PR(**Dependabot**)과 시맨틱 코드 보안 스캔(**CodeQL**) 등 개발자 경험(DX)과 보안 생태계가 가장 촘촘하게 엮여 있다.
- **적합한 상황**: 오픈소스 프로젝트, 개발자 채용·협업 범용성이 중요한 IT 스타트업, 빠른 프로덕트 이터레이션과 AI 보조 워크플로우를 적극 활용하려는 SaaS 제품 팀에 가장 이상적이다.

### ② GitLab: 단일 애플리케이션 기반의 올인원 DevSecOps

GitLab은 소스 코드 관리와 이슈, CI/CD, 컨테이너 레지스트리, 보안 스캐닝(SAST/DAST), 심지어 배포 모니터링까지 **"단 하나의 통합 플랫폼(Single Application)"** 안에서 끝낸다는 철학을 추구한다.

- **핵심 자동화 (`GitLab CI/CD`)**: 별도 마켓플레이스 조합보다 일관된 내장 파이프라인(`.gitlab-ci.yml`), 자사 Runner 아키텍처, 내장 패키지·도커 레지스트리가 유기적으로 통합되어 있어 대규모 파이프라인 관리가 매우 깔끔하다.
- **차별화 강점**: SaaS(`gitlab.com`)뿐만 아니라 무료 오픈소스 버전(**GitLab CE**)과 엔터프라이즈 버전(**GitLab EE**)을 통해 **사내 폐쇄망 온프레미스 서버에 SaaS와 똑같은 UI·기능을 100% 그대로 설치**할 수 있다.
- **적합한 상황**: 외부 클라우드 반출이 금지된 금융·공공·방산·대기업 연구소, 혹은 여러 외부 SaaS 도구를 섞지 않고 단일 DevSecOps 체계로 감사(Audit)·권한·보안을 통제해야 하는 조직에 최적이다.

### ③ Bitbucket: Atlassian(Jira · Confluence) 기반 엔터프라이즈 허브

Atlassian 생태계의 Git 저장소 서비스다. 단독 저장소 매력도보다 회사 내부에 이미 구축된 기획·협업 도구와의 **시너지**에서 독보적인 강점을 발휘한다.

- **핵심 자동화 (`Bitbucket Pipelines`)**: 도커 컨테이너 기반의 가벼운 YAML CI/CD를 제공하며, 배포 상태가 연결된 Jira 이슈 보드에 실시간으로 반영된다.
- **차별화 강점**: 브랜치 이름이나 커밋 메시지에 Jira 티켓 키(`PROJ-123`)만 적으면 Jira 이슈 카드에 **관련 브랜치, 커밋, PR 리뷰 상태, 스테이징/운영 배포 여부까지 자동 연결**된다. Confluence 문서 연동과 Atlassian Guard 기반 전사 SSO 통제도 매끄럽다.
- **적합한 상황**: 전사 표준 협업 도구로 **Jira와 Confluence를 깊이 도입해 운영 중인 기업 조직**에서 계정·권한·업무 추적 체계를 하나로 묶고자 할 때 가장 적은 마찰로 도입할 수 있다.

---

## 3. 왜 가벼운 셀프호스팅인가? Gitea vs Forgejo vs OneDev 전격 해부

사내 보안 규정이나 비용 통제, 개인 홈랩(Homelab) 운영을 위해 자체 서버에 Git을 직접 띄우려 할 때, 많은 팀이 가장 먼저 떠올리는 것은 **GitLab CE**다. 하지만 막상 GitLab CE를 서버에 올리려고 보면 **Rails, Sidekiq, Gitaly, PostgreSQL, Redis, Puma 등이 결합된 거대한 아키텍처 탓에 유휴 상태에서도 최소 4GB~8GB 이상의 RAM**을 요구한다는 벽에 부딪힌다.

단지 소규모 팀이나 개인 서버에서 깔끔한 Git 호스팅과 CI/CD, PR 리뷰 환경을 구축하려는데 8GB 메모리를 쓰기엔 부담스럽다. 이때 등장하는 현대적인 경량·중량 셀프호스팅 3대장이 바로 **Gitea**, **Forgejo**, **OneDev**다.

{{< figure src="self-hosted-git-gitea-forgejo-onedev.svg" alt="셀프호스팅 Git 서버 3종(Gitea, Forgejo, OneDev)의 아키텍처, 거버넌스, 기능 및 RAM 발자국 비교 다이어그램" caption="차세대 셀프호스팅 Git 서버 3대장(Gitea · Forgejo · OneDev) 아키텍처 비교(상단) 및 GitLab 대비 RAM 요구량 스펙트럼(하단)" width="820" height="560" >}}

### ① Gitea: Go 단일 바이너리로 돌아가는 초경량 표준

Gogs 프로젝트에서 포크되어 성장한 **Go 언어 기반 초경량 Git 호스팅 서버**다.

- **아키텍처와 리소스**: 의존성 없이 컴파일된 단일 Go 바이너리(및 SQLite/PostgreSQL/MySQL 선택)로 동작하며, **메모리 사용량이 50~100MB 수준**에 불과해 1GB RAM의 저가형 VPS나 시놀로지 NAS, 라즈베리 파이에서도 거뜬히 돌아간다.
- **기능과 거버넌스**: GitHub과 매우 닮은 직관적인 UI를 제공하며, GitHub Actions 문법과 호환되는 **`Gitea Actions`**, 내장 패키지 레지스트리(Docker, npm, Maven, PyPI 등)까지 갖추었다. 프로젝트는 상업 법인 **Gitea Ltd.** 주도로 운영되며(MIT 라이선스 + CLA), 기업용 엔터프라이즈 기술 지원과 클라우드 호스팅 로드맵을 함께 가져가는 오픈코어(Open-Core) 전략을 취한다.

### ② Forgejo: 비영리 재단 거버넌스와 순수 FOSS를 지키는 하드 포크

**Forgejo**는 2022년 말 Gitea의 상업 법인 전환 및 거버넌스 변화에 반발한 커뮤니티 핵심 기여자들이 **하드 포크(Hard Fork)** 하여 독립한 프로젝트다. 독일의 비영리 자유 소프트웨어 단체인 **Codeberg e.V.** 산하에서 운영된다.

- **아키텍처와 라이선스 차이**: Gitea에서 분기했기 때문에 동일한 **Go 단일 바이너리 기반 ~50~100MB의 초경량 리소스 발자국**을 공유한다. 하지만 라이선스를 MIT에서 카피레프트 성향의 **GPL-3.0-or-later**로 전환하고 기여자 동의서(CLA)를 없애, 향후 특정 기업이 코드를 사유화하거나 독점 엔터프라이즈 기능을 잠그는 위험을 원천 차단했다.
- **차별화 기능**: GitHub Actions 호환 **`Forgejo Actions`** 와 독립 보안 권고 체계를 유지하는 한편, 서로 다른 자가 호스팅 Git 서버 인스턴스끼리 별·이슈·PR을 주고받을 수 있는 **ActivityPub 기반 연합(Federation)** 기능에 적극 투자하고 있다.

### ③ OneDev: GitLab보다 가볍고 GitHub보다 깊은 올인원 + 코드 인텔리전스

**OneDev**는 Go 기반인 Gitea/Forgejo와 전혀 다른 기술적 노선을 걷는 독립 오픈소스 프로젝트다(**Java / JVM 스택**, MIT 기반 오픈소스 + 엔터프라이즈 확장 모듈 구분).

- **가장 강력한 무기 — 언어 인지형 코드 인텔리전스(Code Intelligence)**: OneDev는 단순 텍스트 검색이 아니라 **ANTLR 파서와 Apache Lucene**을 서버 내부에 내장해 Java, Go, Python, C/C++, JavaScript/TypeScript 등 주요 언어의 AST 심볼을 직접 색인한다. 그 결과 **별도 로컬 IDE 없이 웹 브라우저 UI에서 과거 임의 커밋 시점의 함수·클래스 정의로 바로 점프(Jump to Definition)하고 참조 검색까지 수행**할 수 있다.
- **하이브리드 CI/CD와 Kubernetes 네이티브 빌드 팜**: YAML 파일을 손으로 다 치지 않아도 **웹 GUI에서 파이프라인 DAG와 조건을 시각적으로 조립**(YAML 자동 동기화)할 수 있으며, Docker 또는 **Kubernetes 네이티브 동적 빌드 팜**을 내장해 각 빌드 스텝을 격리된 Pod로 자동 스케일링 실행한다.
- **고도화된 이슈 트래킹과 리소스 균형**: 사용자 정의 상태 머신(State Machine), 부모-자식 태스크 계층, 칸반 보드가 PR 리뷰와 깊게 통합되어 있어 Jira 수준의 이슈 관리가 가능하다. Java 기반 심볼 인덱싱을 포함하므로 유휴 메모리는 **~400MB~2GB RAM** 정도로 Gitea/Forgejo보다는 높지만, 여전히 4~8GB+를 요구하는 GitLab CE보다는 훨씬 가볍게 **"코드·CI·이슈가 하나로 통합된 서버"** 를 운영할 수 있다.

---

## 4. 한눈에 보는 6대 Git 호스팅 플랫폼 비교와 선택 가이드

지금까지 살펴본 클라우드 3대장과 셀프호스팅 3대장의 핵심 스펙을 하나의 표로 정리하면 다음과 같다.

| 플랫폼 | 주요 운영 형태 | 기술 스택 / RAM 요구량 | CI/CD 엔진 | 핵심 차별화 포인트 | 최적 추천 시나리오 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **GitHub** | 매니지드 SaaS *(유료 Enterprise Server)* | 클라우드 관리형 | **GitHub Actions** | 오픈소스 표준, Copilot, 방대한 마켓플레이스 | 오픈소스, 스타트업, SaaS 제품 팀 |
| **GitLab** | SaaS + 풀스택 셀프호스팅 | Ruby/Go 복합 *(자체 4~8GB+)* | **GitLab CI/CD** | 단일 앱 올인원 DevSecOps, 완벽한 폐쇄망 구축 | 금융·공공·대기업 사내 표준 플랫폼 |
| **Bitbucket** | 매니지드 SaaS *(Data Center)* | 클라우드 관리형 | **Bitbucket Pipelines** | Jira · Confluence 네이티브 티켓 자동 연동 | Atlassian 도구 중심 기업 조직 |
| **Gitea** | 셀프호스팅 *(경량)* | **Go** *(~50~100MB)* | **Gitea Actions** | 초경량 단일 바이너리, GitHub 유사 UX, 상용 지원 옵션 | 저사양 VPS·NAS, 빠른 사내 Git 구축 |
| **Forgejo** | 셀프호스팅 *(경량)* | **Go** *(~50~100MB)* | **Forgejo Actions** | 비영리(Codeberg) 거버넌스, **GPLv3+**, ActivityPub 연합 | 순수 FOSS 철학, 프라이버시 중시 팀/홈랩 |
| **OneDev** | 셀프호스팅 *(올인원)* | **Java/JVM** *(~0.4~2GB)* | **GUI/YAML CI + K8s 팜** | **웹 심볼 정의 점프(ANTLR)**, Jira급 칸반 이슈 보드 | GitLab 대체 중량급 올인원(코드검색·K8s CI) |

### 우리 팀과 인프라에는 무엇을 고를까?

1. **별도 서버 관리 없이 표준 협업과 AI 생태계를 누리고 싶다면 → GitHub**
2. **이미 사내 워크플로우가 Jira·Confluence에 묶여 있다면 → Bitbucket**
3. **고성능 서버 자원이 충분하고 사내 폐쇄망에 엔터프라이즈 DevSecOps 일체를 구축해야 한다면 → GitLab (CE/EE)**
4. **작은 VM이나 홈랩 서버에 가장 빠르고 익숙한 초경량 Git 서버를 올리고 싶다면 → Gitea** *(기업 지원/로드맵 선호)* 또는 **Forgejo** *(순수 비영리 FOSS · GPLv3 선호)*
5. **GitLab처럼 무거운 풀스택은 부담스럽지만, 웹에서 코드 심볼 점프와 Kubernetes 기반 CI, 칸반 보드까지 한 번에 해결하고 싶다면 → OneDev**

---

## 마치며

Pro Git 6장의 원문 제목은 *GitHub*이지만, 책이 강조하고자 한 본질은 특정 회사의 UI 버튼 위치가 아니라 **"원격 저장소 플랫폼이 Git의 분산 프로토콜 위에 어떻게 PR 참조(`refs/pull/*`), 권한 경계(Organization), 그리고 Webhook 자동화를 얹어 현대 소프트웨어 공장의 중심이 되었는가"** 에 있다.

클라우드 SaaS로 속도를 택하든, 가벼운 Go/JVM 기반 셀프호스팅 서버로 데이터 주권과 인프라 효율을 택하든, 앞선 1~5장에서 익힌 Git 객체 모델과 브랜치·리모트 트래킹 원리를 쥐고 있다면 어떤 플랫폼으로 옮겨 가도 흔들리지 않고 팀에 맞는 최적의 개발 환경을 설계할 수 있다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - Chapter 6: GitHub](https://git-scm.com/book/en/v2/GitHub-Account-Setup-and-Configuration)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 6.3 GitHub - Maintaining a Project](https://git-scm.com/book/en/v2/GitHub-Maintaining-a-Project)
- [Gitea Official Documentation](https://docs.gitea.com/)
- [Forgejo Official Website & Documentation](https://forgejo.org/)
- [OneDev Official Website (Code Intelligence & K8s CI/CD)](https://onedev.io/)
