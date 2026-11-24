+++
date = '2026-11-24T10:00:00+09:00'
draft = false
title = 'Git 브랜칭 전략 비교: GitFlow부터 TBD까지'
description = 'GitFlow, GitHub Flow, GitLab Flow, Trunk-Based Development 등 7대 Git 브랜칭 전략의 구조와 장단점, 팀 상황별 선택 기준을 도식과 함께 정리한다.'
tags = ['Git', 'ProGit', 'BranchingStrategy', 'GitFlow', 'GitHubFlow', 'GitLabFlow', 'TrunkBased', 'DevOps', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[Pro Git 3장 시리즈(1~3부)]({{< ref "/posts/pro-git-internals-ch3-part3" >}})에서는 `.git/refs/heads/` 아래 놓인 41바이트 커밋 포인터의 본질부터 Fast-forward · 3-Way Merge, `git branch --merged` 기반 관리, Long-Running vs Topic 워크플로우, 그리고 `git rebase`의 패치 재적용 원리까지 살펴보았다.

Git이 다른 중앙 집중형 버전 관리 도구(SVN 등)와 결정적으로 다른 점은 **브랜치 생성과 병합 비용이 사실상 0에 가깝다**는 것이다. 이 가벼운 포인터 모델 덕분에 개발 조직은 자신의 **팀 규모, 릴리즈 주기, QA 프로세스, CI/CD 성숙도**에 맞춰 전혀 다른 협업 아키텍처를 설계할 수 있게 되었다.

그렇다면 실무에서 널리 쓰이는 대표 브랜칭 전략에는 어떤 것들이 있고, 우리 팀의 현재 상황에는 어떤 전략이 맞을까? 이번 글에서는 Prateek Jain의 *[The Ultimate Guide to Git Branching Strategies](https://blog.prateekjain.dev/the-ultimate-guide-to-git-branching-strategies-6324f1aceac2?sk=738af8bd3ffaae39788923bbedf771ca)* 가이드를 바탕으로, Pro Git 3장에서 배운 내부 원리 위에 **실무 7대 브랜칭 전략의 구조·장단점·적합 상황**을 시각적 도식과 함께 체계적으로 정리한다.

> **참고**: 이 글은 [Pro Git 3장 3부(#6)]({{< ref "/posts/pro-git-internals-ch3-part3" >}})에서 다룬 3-Way Merge와 Rebase 철학을 실전 협업 아키텍처로 확장하는 심화 가이드다.

---

## 1. 왜 브랜칭 전략이 필요하며, 모든 전략의 공통 재료는 무엇인가?

브랜칭 전략 없이 모든 팀원이 `main` 브랜치 하나에만 뒤섞여 커밋을 쌓기 시작하면 코드는 순식간에 배포 불가능한 상태가 된다. 명확한 브랜칭 전략은 실무 팀에 네 가지 안전장치를 제공한다.

- **변경 격리와 충돌 감소**: 완성되지 않은 실험 코드와 운영 배포 코드를 물리적으로 분리해 개발 중 장애 유입을 막고, 대형 병합 충돌(Merge Conflict)을 통제 가능한 단위로 쪼갠다.
- **CI/CD 품질 게이트 연결**: 특정 브랜치(`main`, `release/*`, PR 타겟)로 코드가 들어갈 때만 테스트·린트·보안 스캔·배포 파이프라인을 자동으로 트리거할 수 있다.
- **협업 예측 가능성 확보**: "지금 운영 서버에 올라간 코드는 어디 있는가?", "긴급 버그를 고치려면 어디서 분기해야 하는가?"에 대해 팀 전체가 설명 없이 같은 답을 내놓을 수 있다.

사실 실무에 존재하는 수많은 브랜칭 전략은 외워야 할 별개의 규칙이 아니다. 모두 Pro Git 3.4([3장 2부]({{< ref "/posts/pro-git-internals-ch3-part2" >}}))에서 다룬 **두 가지 기본 빌딩 블록**의 조합일 뿐이다.

1. **Long-Running 브랜치(영속 안정성 사일로)**: `main`, `develop`, `staging`, `production`처럼 프로젝트 수명 내내 지우지 않고 유지하며 검증 단계를 구분하는 축이다.
2. **Topic / Feature 브랜치(단기 격리 단위)**: `feature/*`, `bugfix/*`, `hotfix/*`처럼 단일 이슈를 해결하기 위해 만들었다가 병합 후 즉시 삭제하는 축이다.

여기에 팀이 **병합 시 어떤 히스토리 전략**(`--no-ff` Merge 커밋 보존 vs `rebase` 선형 이력 vs `--squash` 단일 커밋 압축)을 결합하느냐에 따라 아래의 대표 전략들이 완성된다.

---

## 2. 릴리즈 중심 vs 상시 배포: GitFlow와 GitHub Flow

가장 고전적인 대립 구도이자, 지금도 가장 많은 팀이 첫 기준으로 삼는 두 모델이 **GitFlow**와 **GitHub Flow**다. 두 전략은 **"우리가 소프트웨어를 정기 버전으로 끊어 배포하는가, 아니면 하루에도 몇 번씩 상시 배포하는가"** 라는 질문에 따라 갈린다.

{{< figure src="gitflow-vs-github-flow.svg" alt="GitFlow 5개 브랜치 트랙과 GitHub Flow 단일 main 상시 배포 파이프라인 비교 다이어그램" caption="GitFlow(상단: main · hotfix · release · develop · feature 5종 트랙)와 GitHub Flow(하단: 단일 main · 단기 PR 4단계 루프)의 구조 및 적합 상황 비교" width="820" height="560" >}}

### ① GitFlow (릴리즈 중심 5종 브랜치 모델)

2010년 Vincent Driessen이 제안한 GitFlow는 **엄격한 릴리즈 주기와 공식적인 QA 단계**를 가정한 가장 구조화된 전략이다. 장기 브랜치 2개(`main`, `develop`)와 보조 브랜치 3종(`feature/*`, `release/*`, `hotfix/*`)을 명확히 구분한다.

- **`main` (영속)**: 언제든 실제 운영 환경에 나간 정식 릴리즈 스냅샷만 보관하며, 병합마다 `v1.0.0`, `v1.1.0` 같은 버전 태그를 남긴다.
- **`develop` (영속)**: 다음 정식 릴리즈를 향해 완성된 기능들이 모이는 통합 브랜치다.
- **`feature/*` (단기)**: `develop`에서 분기해 새 기능을 개발한 뒤, 완료되면 `--no-ff` 옵션 등으로 다시 `develop`에 병합하고 삭제한다.
- **`release/*` (단기)**: `develop`이 다음 버전 목표치를 채우면 `release/1.1.0` 형태로 분기해 코드 동결(Code Freeze)에 들어간다. 여기서는 신규 기능을 받지 않고 오직 QA, 버전 번호 갱신, 릴리즈 직전 버그 수정만 수행한다. 완료되면 **`main`과 `develop` 양쪽에 모두 병합**해 QA 수정 사항이 유실되지 않게 한다.
- **`hotfix/*` (단기)**: 운영 중인 `main`에서 치명적 장애가 터졌을 때 `develop`의 미완성 기능을 기다리지 않고 `main` 태그에서 즉시 분기한다. 패치 후에도 마찬가지로 **`main`과 `develop` 양쪽에 모두 병합**해 다음 버전에서 같은 버그가 재발하지 않도록 막는다.

**장점과 단점**:
- **장점**: QA 안정화 작업(`release/*`)과 차기 기능 개발(`develop` · `feature/*`)이 서로를 가로막지 않는다. 누가 어떤 버전의 코드를 검증 중인지 브랜치 이름만으로 100% 구분된다.
- **단점**: `release/*`와 `hotfix/*`를 두 군데로 이중 병합하는 관리 오버헤드가 크고, 브랜치 수명이 길어질수록 `develop` 통합 시 대규모 충돌 위험이 커진다. 실제로 원저자 Vincent Driessen 역시 2020년 원문에 회고를 추가해, "하루에도 수십 번 배포하는 현대 웹 앱(CD)에는 GitFlow보다 GitHub Flow처럼 단순한 모델이 훨씬 적합하다"고 직접 밝힌 바 있다.

**어떤 상황에 적합한가?**
- **모바일 앱(iOS/Android)**: 앱 스토어 심사 기간 동안 배포 후보(`release/*`)를 동결해 QA를 돌리면서 동시에 다음 스프린트 기능을 개발해야 하는 경우
- **정기 릴리즈 · 엔터프라이즈 소프트웨어**: 분기/월별 정식 버전 배포와 엄격한 승인 감사 기록(Audit Trail)이 요구되는 금융·대규모 패키지 제품

### ② GitHub Flow (단일 `main` 기반 상시 배포 모델)

GitHub Flow는 GitFlow의 복잡성을 과감하게 걷어내고, **지속적 전달(Continuous Delivery)** 을 극대화하기 위해 탄생한 경량 모델이다.

1. **`main`은 항상 배포 가능(Always Deployable)한 상태**를 유지하는 유일한 장기 브랜치다.
2. 새 기능이나 버그 수정이 필요하면 언제든 `main`에서 의미 있는 이름의 단기 브랜치(`feature/*` 또는 `fix/*`)를 분기한다.
3. 로컬에서 작은 단위로 커밋하고 원격 브랜치에 푸시해 작업을 공유한다.
4. **Pull Request(PR)** 를 열어 자동화된 CI 테스트를 돌리고 팀원의 코드 리뷰를 받는다(필요 시 PR 전용 프리뷰 환경에 배포해 검증한다).
5. 리뷰 승인 및 CI 통과가 끝나면 `main`에 병합하고, **즉시 프로덕션에 배포(CD)** 한 뒤 작업 브랜치를 삭제한다.

**장점과 단점**:
- **장점**: 규칙이 단순해 온보딩이 몇 분이면 끝나며, 작업 단위가 작아 PR 피드백 속도가 빠르고 충돌이 거의 발생하지 않는다.
- **단점**: 버전 동결용 `release/*`나 다중 버전 유지보수 트랙이 없으므로, `main`에 들어가는 코드가 항상 운영 품질을 보장하도록 **탄탄한 자동화 테스트 스위트**가 뒷받침되어야 한다.

**어떤 상황에 적합한가?**
- **웹 서비스 및 SaaS 제품**: 단일 운영 버전만 유지하면서 하루에도 여러 번 기능을 배포하는 스타트업·애자일 제품 팀

---

## 3. 다중 배포 환경과 다중 버전 유지보수: GitLab Flow · Environment · Release Branching

현실에는 "GitFlow는 너무 무겁지만, GitHub Flow처럼 `main` 병합이 곧바로 운영 배포로 이어질 수는 없는" 팀이 매우 많다. 배포 승인 게이트나 다수의 서버 환경(`dev` → `staging` → `prod`), 혹은 과거 버전 동시 지원(LTS)이 필요한 경우다. 이때 등장하는 전략이 **GitLab Flow**, **Environment Branching**, 그리고 **Release Branching**이다.

{{< figure src="gitlab-environment-and-release-flow.svg" alt="GitLab Flow 환경별 승격과 Upstream-First 원칙 및 Release Branching cherry-pick 백포팅 비교 다이어그램" caption="GitLab Flow / Environment Branching(상단: feature → main → staging → production 및 Upstream-First 핫픽스 전파)과 Release Branching(하단: 다중 release 브랜치 cherry-pick 백포팅)" width="820" height="540" >}}

### ① GitLab Flow & Environment Branching (환경별 승격과 Upstream-First)

**Environment Branching**은 `main` 외에 실제 서버 배포 환경과 1:1로 대응되는 브랜치(`staging` / `pre-production`, `production` 등)를 두는 모델이며, **GitLab Flow**는 여기에 **CI/CD 규율과 Upstream-First 원칙**을 결합한 전략이다.

- `feature/*`에서 작업해 코드 리뷰와 테스트를 통과하면 먼저 `main`에 통합한다.
- 일정 주기나 배포 승인 시점에 `main`을 `staging`(또는 `pre-production`)으로 병합해 통합 검증 환경에 배포한다.
- 스테이징 검증이 끝나면 `staging`을 `production` 브랜치로 병합(승격)해 실서버에 반영한다.

여기서 절대 어기면 안 되는 규칙이 **Upstream-First(상류 우선 원칙)** 다. 만약 `production` 환경에서 긴급 버그가 발생했다고 해서 `production` 브랜치에 직접 커밋을 넣어 고치면 어떻게 될까? 당장은 장애가 해결되지만, 다음번에 `main` → `staging` → `production`으로 새 기능을 승격시키는 순간 그 핫픽스가 누락되어 장애가 재발(Regression)할 수 있다. 따라서 GitLab Flow에서는 **모든 버그 수정도 반드시 최상류인 `main`에서 먼저 커밋·테스트한 뒤, `staging`을 거쳐 `production`으로 한 방향으로만 내려보내야(또는 `cherry-pick` 해야) 한다.**

*(참고: GitLab Flow는 환경 기반 모델 외에도 외부 배포용 버전 트랙이 필요한 팀을 위해 아래의 Release Branch 기반 모델 역시 공식 변형으로 함께 제공한다.)*

**어떤 상황에 적합한가?**
- **다중 인프라 · 규제 승인이 필요한 웹/백엔드 서비스**: 스테이징 QA 승인, 변경 관리 위원회(CAB) 검토, 배포 윈도우 등 "코드 병합 시점"과 "운영 배포 시점"을 분리해야 하는 기업 환경

### ② Release Branching (다중 버전 동시 지원 및 `cherry-pick` 백포팅)

SaaS처럼 서버 한 곳만 업데이트하면 끝나는 서비스와 달리, 고객사에 설치형 바이너리를 납품하거나 오픈소스 라이브러리/SDK를 배포하는 팀은 **`v1.x`와 `v2.x`를 동시에 유지보수**해야 하는 일이 흔하다. **Release Branching**은 바로 이 문제를 해결한다.

- 평소 모든 신규 개발은 `main`에서 진행한다.
- 특정 마일스톤(`v1.0`, `v2.0`)에 도달하면 해당 커밋에서 `release/1.0`, `release/2.0` 브랜치를 분기해 동결하고 태그(`v1.0.0`, `v2.0.0`)를 찍는다.
- `main`에서 계속 차기 기능(`C3`, `C4`)이 개발되던 중 구버전에도 영향을 주는 보안 취약점이나 버그가 수정(`B1`)되면, **신규 기능(`C3`, `C4`)을 섞지 않고 오직 수정 커밋 `B1`만 `git cherry-pick`으로 `release/2.0`과 `release/1.0`에 이식(Backporting)** 해 패치 버전(`v2.0.1`, `v1.0.2`)을 발행한다.

**어떤 상황에 적합한가?**
- **온프레미스(설치형) 엔터프라이즈 소프트웨어, 오픈소스 프레임워크 · SDK · DB 엔진**: 구버전 LTS(Long-Term Support) 고객에게 신규 기능 없이 보안/버그 패치만 제공해야 하는 제품

---

## 4. 고성숙 CI/CD의 종착지: Trunk-Based Development (TBD)와 Forking Workflow

### ① Trunk-Based Development (TBD)

구글, 메타, 넷플릭스 등 최상위 DORA(DevOps Research and Assessment) 지표를 달성하는 엔지니어링 조직들이 표준처럼 채택하는 전략이 **Trunk-Based Development**(트렁크 기반 개발)다. 이름 그대로 모든 개발자가 하나의 중심 브랜치(`main` 또는 `trunk`)에 **매일, 혹은 하루에도 여러 번 코드를 통합**하는 것을 원칙으로 삼는다.

{{< figure src="trunk-based-and-decision-matrix.svg" alt="Trunk-Based Development 초단기 통합 · Feature Flag 분리 구조와 7대 브랜칭 전략 선택 가이드 매트릭스" caption="Trunk-Based Development의 초단기 통합 · Feature Flag 기반 배포/출시 분리(상단)와 팀 상황에 따른 7대 브랜칭 전략 선택 가이드(하단)" width="820" height="560" >}}

GitHub Flow와 비슷해 보이지만 실무 적용 강도에서 결정적 차이가 있다.
- **극단적으로 짧은 브랜치 수명**: 소규모 팀은 `trunk`에 직접 커밋하기도 하고, 중대형 팀에서도 PR용 단기 브랜치의 수명을 **1~2일 이내**로 강하게 제한한다. 며칠씩 묵혀 둔 거대 Feature 브랜치 자체가 금지된다.
- **Feature Flag(기능 토글)로 배포와 출시 분리**: "아직 다 안 만든 기능을 매일 `main`에 합치면 운영 장애가 나지 않을까?"라는 의문은 **Feature Flag**로 해결한다. 미완성 코드도 인터페이스가 깨지지 않게 작게 쪼갠 뒤 `flag: OFF` 상태로 감싸 `main`에 병합하고 운영 서버까지 배포(Deploy)한다. 기능이 완전히 완성되면 재배포 없이 설정 값만 `ON`으로 켜서 사용자에게 출시(Release)하며, 문제가 생기면 1초 만에 `OFF`로 되돌린다. 대규모 리팩토링 시에는 추상화 계층 뒤에 새 구현을 점진 교체하는 **Branch by Abstraction** 패턴을 함께 쓴다.

**장점과 단점**:
- **장점**: 며칠 뒤 수십 명이 부딪히는 'Merge Hell(병합 지옥)' 자체가 원천적으로 사라지며, 리드 타임과 배포 빈도 등 개발 생산성 지표가 압도적으로 높아진다.
- **단점**: `main`의 무결성을 지탱해 줄 **고속·고신뢰 자동화 테스트(Unit/Integration CI)**, 시니어·동료 간 빠른 코드 리뷰 SLA(예: 몇 시간 내 리뷰), 그리고 Feature Flag 관리 역량이 선결 조건이다. 테스트가 부실한 상태에서 TBD만 도입하면 `main`이 하루 종일 깨지는 역효과가 난다.

**어떤 상황에 적합한가?**
- **성숙한 CI/CD와 테스트 자동화를 갖춘 DevOps 조직**, 대규모 동시 개발자가 같은 모노레포/서비스를 빠르게 배포해야 하는 환경

### ② Forking Workflow (저장소 경계 기반 기여 모델)

앞선 전략들이 "하나의 공유 저장소 내부에서 브랜치를 어떻게 나눌 것인가"를 다루었다면, **Forking Workflow**는 저장소 접근 권한 자체를 분리하는 **토폴로지 전략**이다(자세한 이중 리모트 동작은 [Pro Git 5장 1부](/posts/pro-git-internals-ch5-part1/)에서도 상세히 다룬다).

- 기여자는 공식 저장소에 직접 쓰기 권한을 받는 대신, 서버 측에 자신만의 복제 저장소(**Fork**)를 생성한다.
- 로컬에는 공식 저장소(`upstream` 또는 `origin`)와 내 Fork(`origin` 또는 `myfork`)를 이중 리모트로 연결해 작업하고, Topic 브랜치를 내 Fork로 푸시한 뒤 공식 저장소 메인테이너에게 PR을 보낸다.

**어떤 상황에 적합한가?**
- **오픈소스 프로젝트**, 혹은 수백·수천 명이 일하면서도 핵심 모듈에 대한 직접 Push 권한을 소수 아키텍트/메인테이너에게만 제한해 거버넌스·보안을 지켜야 하는 대기업 내부 소스(InnerSource) 환경

---

## 5. 한눈에 보는 7대 브랜칭 전략 비교와 공통 실전 수칙

글 전체에서 다룬 7가지 개념(**Feature Branching**을 기본 축으로 포함한 실무 7대 전략)의 핵심 특성을 한 표로 압축하면 다음과 같다.

| 전략 / 모델 | 장기 브랜치 구성 | 배포 주기 특성 | CI/CD 및 테스트 요구 수준 | 관리 복잡도 | 가장 적합한 상황 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Feature Branching** *(기본 블록)* | `main` + 작업별 `feature/*` | 유동적 | 기본 CI 권장 | 낮음 | 소규모 팀의 기본 격리 출발점 |
| **GitFlow** | `main`, `develop` *(+ release/hotfix)* | 정기 · 계획 릴리즈 | 중간 *(수동 QA 병행 가능)* | **높음** | 앱 스토어 심사, 공식 QA, 정기 버전 배포 |
| **GitHub Flow** | `main` 단독 | 수시 · 지속 배포(CD) | 높음 *(PR 자동 검증 필수)* | **낮음** | 단일 프로덕션 웹/SaaS, 애자일 제품 팀 |
| **GitLab Flow** *(Environment)* | `main` + 환경별(`staging`, `prod`) | 승인 · 단계별 배포 | 높음 | 중간 | 환경별 승격 게이트, 인프라 검증 분리 팀 |
| **Release Branching** | `main` + `release/1.x`, `2.x` | 다중 버전 병행 패치 | 중간~높음 | 중간~높음 | 설치형(온프레미스), SDK, 다중 LTS 지원 |
| **Trunk-Based (TBD)** | `main(trunk)` 단독 | 초고빈도 CD *(일 수회+)* | **매우 높음** *(Feature Flag 필수)* | 낮음 *(규율 높음)* | 고성숙 DevOps 팀, 대규모 모노레포 빠른 배포 |
| **Forking Workflow** | 공식 저장소 + 개인 Fork | 오픈소스 · 게이트 병합 | 높음 *(외부 PR 검증)* | 중간 | 오픈소스 커뮤니티, 비신뢰 기여자 거버넌스 |

### 어떤 전략을 선택하든 반드시 지켜야 할 4가지 실전 원칙

Prateek Jain의 가이드와 Pro Git 3장을 관통하는 결론은 명확하다. **"완벽한 단 하나의 은탄환 전략은 없으며, 오버엔지니어링이 가장 위험하다."** 3인 스타트업이 GitFlow의 5종 트랙을 억지로 도입하면 회의와 병합에만 시간을 뺏기고, 반대로 테스트 코드도 없는 팀이 TBD를 흉내 내면 운영 장애가 일상화된다.

어떤 전략을 고르든 다음 네 가지 원칙을 함께 세워야 전략이 실제로 작동한다.

1. **브랜치 수명은 짧게, 통합은 자주**: 브랜치가 `main`/`develop`과 멀어질수록 3-Way Merge 충돌 비용은 기하급수적으로 커진다. 한 PR은 하루~이틀 단위로 작게 쪼개 통합한다.
2. **명명 규칙 표준화와 주기적 청소 (`--merged`)**: `feature/`, `fix/`, `hotfix/`, `release/` 접두사를 통일해 의도를 즉시 드러내고, 병합이 끝난 브랜치는 `git branch --merged`로 점검해 바로 삭제한다.
3. **자동화 품질 게이트와 코드 리뷰 의무화**: 브랜치 보호 규칙(Branch Protection)을 켜서 CI 테스트와 최소 1명 이상의 리뷰 승인 없이는 보호 브랜치에 들어갈 수 없도록 시스템으로 강제한다.
4. **팀의 Merge vs Rebase 철학 명문화**: "로컬 Topic 브랜치를 PR 전에 `git rebase`로 정돈할지, 병합 시 `--no-ff` Merge Commit을 남길지, `--squash`로 단일 커밋으로 압축할지"를 저장소 규칙으로 명확히 문서화해 일관된 히스토리를 유지한다.

---

## 마치며

**Pro Git 3장**에서 배운 Git 브랜치의 내부 원리는 단순히 `git checkout`과 `git merge` 명령어를 외우는 데서 끝나지 않는다. `HEAD`와 41바이트 브랜치 포인터가 어떻게 움직이고, Fast-forward · 3-Way Merge · Rebase · Cherry-pick이 커밋 DAG 그래프를 어떻게 변형하는지 이해하고 나면, 팀 규모가 3명에서 300명으로 커지고 배포 환경이 바뀌더라도 **왜 지금 이 전략을 쓰고 언제 다음 전략으로 진화해야 하는지** 스스로 판단할 수 있다.

지금 팀의 브랜치 규칙을 돌아보자. 현재의 배포 주기와 테스트 자동화 수준에 비해 너무 무거운 트랙을 끌고 있지는 않은가, 혹은 환경 분리가 필요한데 단일 브랜치에 기대고 있지는 않은가? 작은 PR과 짧은 브랜치 수명이라는 기본 원칙 위에서 팀의 현실에 가장 잘 맞는 도식을 선택해 보기 바란다.

---

## 참고 자료

- [Prateek Jain - The Ultimate Guide to Git Branching Strategies](https://blog.prateekjain.dev/the-ultimate-guide-to-git-branching-strategies-6324f1aceac2?sk=738af8bd3ffaae39788923bbedf771ca)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.4 Git Branching - Branching Workflows](https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 3.6 Git Branching - Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing)
- [Vincent Driessen - A successful Git branching model (GitFlow)](https://nvie.com/posts/a-successful-git-branching-model/)
- [Martin Fowler - Patterns for Managing Source Code Branches](https://martinfowler.com/articles/branching-patterns.html)
