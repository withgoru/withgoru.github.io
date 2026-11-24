+++
date = '2026-12-05T09:00:00+09:00'
draft = false
title = 'Git Hooks 가이드 #16: pre-commit·Husky·정책'
description = 'Pro Git 8장 2부(8.3~8.4)를 바탕으로 Git Hooks 생명주기, pre-commit·Husky·Lefthook, commit-msg 실습과 서버 정책 강제를 정리한다.'
tags = ['Git', 'ProGit', 'GitHooks', 'PreCommit', 'Husky', 'Lefthook', 'DevOps', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

[지난 글(#15)](/posts/pro-git-internals-ch8-part1/)에서는 `git config`와 실무 `.gitattributes`를 통해 Git의 환경과 파일 처리 규칙을 표준화하는 방법을 살펴보았다. 이번 **8장 2부(`#16`, Pro Git 8.3~8.4)** 에서는 특정 Git 이벤트마다 스크립트를 실행해 코드 품질과 커밋 컨벤션을 강제하는 **Git Hooks**를 다룬다.

특히 실무에서 널리 쓰이는 **`pre-commit` 프레임워크(`pre-commit.com`)** 를 이용해 `pre-commit`과 `commit-msg` 단계에서 품질을 방어하는 실전 설정법과, 로컬 우회(`--no-verify`)를 막는 서버 측 정책 강제 전략까지 함께 정리한다.

---

## 1. Git Hooks의 실행 생명주기: 클라이언트 훅 vs 서버 훅 (8.3)

Git은 커밋이나 푸시 같은 주요 사건이 일어날 때마다 `.git/hooks/` 디렉토리에 놓인 실행 가능한 스크립트를 자동으로 호출한다. 이때 규칙은 단 하나다. **스크립트가 종료 코드(Exit Code) `0`을 반환하면 다음 단계로 진행하고, `0`이 아닌 값(`1` 등)을 반환하면 Git 작업을 즉시 중단시킨다.**

{{< figure src="git-hooks-lifecycle-and-precommit.svg" alt="Git Hooks 클라이언트 및 서버 실행 생명주기, pre-commit 프레임워크 구조, 그리고 팀 훅 매니저와 2중 방어선 다이어그램" caption="클라이언트와 서버 측 Git Hooks 전체 실행 흐름(상단) 및 pre-commit 프레임워크 구조와 2중 품질 방어선(하단) (Pro Git 8.3~8.4 재구성)" width="820" height="560" >}}

### ① 클라이언트(로컬) 훅 타임라인

1. **`pre-commit`**: 커밋 메시지를 입력하기도 전에, `git commit` 실행 직후 가장 먼저 돈다. 주로 Staged 파일에 대해 포맷터, 린터, 트레일링 공백이나 API 키 유출 여부를 검사한다.
2. **`prepare-commit-msg`**: 커밋 에디터가 열리기 직전 기본 메시지(예: 브랜치 이름에서 추출한 Jira 이슈 키 `CDP-32`)를 자동으로 채워 넣을 때 쓴다.
3. **`commit-msg`**: 사용자가 작성한 커밋 메시지 파일 경로를 첫 번째 인자(`$1`)로 받아 **Conventional Commits 등 컨벤션 준수 여부를 최종 검증**한다.
4. **`post-commit`**: 커밋 완료 후 실행된다(이미 커밋된 뒤라 종료 코드가 커밋을 취소시키지 않는다).
5. **`pre-push`**: `git push` 시 원격 서버와 연결한 뒤 객체를 전송하기 직전에 실행되며, 무거운 통합 테스트나 브랜치 보호를 로컬에서 한 번 더 거를 때 사용한다.

### ② 서버(원격 수신) 훅 타임라인

- **`pre-receive`**: 클라이언트가 Push한 모든 참조(Ref) 목록을 표준 입력으로 한꺼번에 받아 검증한다. 여기서 `non-zero`를 반환하면 **Push 전체가 한꺼번에 거절**된다.
- **`update`**: `pre-receive` 직후, Push로 갱신하려는 **각 브랜치마다 1회씩 개별 실행**된다. 특정 브랜치만 거부하고 다른 브랜치는 통과시키는 등 세밀한 제어가 가능해 Pro Git 8.4의 정책 강제 예제가 주로 사용하는 핵심 훅이다.
- **`post-receive`**: Push가 완전히 끝난 뒤 실행되며, CI/CD 빌드 트리거, 이메일·Slack 알림 발송에 쓰인다.

---

## 2. `.git/hooks/` 공유 불가 한계와 팀 훅 매니저 선택 기준

기본 `.git/hooks/` 디렉토리는 Git 저장소의 추적 대상이 아니므로, 다른 팀원이 `git clone`을 받아도 **내가 공들여 만든 훅 스크립트가 자동으로 복제되지 않는다.** 팀 전체에 동일한 훅을 배포하려면 다음 방법 중 하나를 택해야 한다.

- **Git 네이티브 `core.hooksPath`**: 저장소 안에 `.githooks/` 폴더를 만들어 커밋해 두고, 팀원이 `git config core.hooksPath .githooks`를 실행해 연결한다. 별도 도구 설치 없이 가장 단순하다.
- **Git 네이티브 Config-Based & Parallel Hooks (Git 2.54~2.55)**: **Git 2.54(2026.04)** 부터 `.gitconfig`·`.git/config`에 `[hook "<name>"]` 블록으로 여러 훅을 직접 선언하는 설정 기반 훅이 도입되었고, **Git 2.55(2026.06)** 에서는 `parallel = true`와 `[hook] jobs` 옵션으로 독립 훅의 **네이티브 병렬 실행**까지 지원한다.
- **`pre-commit` (`pre-commit.com`)**: Python 기반이지만 Node, Go, Rust, Docker 등 **모든 언어의 린터를 격리 환경에 자동 설치해 주는 다언어 표준 도구**다.
- **`Husky` + `lint-staged`**: Node.js/프론트엔드 생태계의 사실상 표준이다. `npm install`(`prepare` 스크립트)만으로 팀원 모두에게 자동으로 훅이 설치된다.
- **`Lefthook`**: Go 단일 바이너리로 제작되어 실행 속도가 압도적으로 빠르며, 저장소 내 YAML 선언 기반 병렬 훅 실행으로 대규모 모노레포에 강하다.

```ini
# Git 2.54~2.55 네이티브 Config-Based & Parallel Hooks 예시 (.gitconfig 또는 .git/config)
[hook]
    jobs = -1    # 사용 가능한 CPU 코어 수만큼 병렬 실행 (Git 2.55+)

[hook "linter"]
    event = pre-commit
    command = ruff check --fix
    parallel = true

[hook "typecheck"]
    event = pre-commit
    command = mypy src/
    parallel = true
```

*(다만 Git 네이티브 Config-Based Hooks는 언어별 런타임/린터 자동 설치나 저장소 단위 단일 설정 배포 기능까지는 포함하지 않으므로, 팀 온보딩 자동화가 중요하다면 아래 `pre-commit`이나 Husky·Lefthook과 함께 비교해 선택하는 것이 좋다.)*

---

## 3. 실무 심화: `pre-commit` 프레임워크로 코드·커밋 품질 동시에 지키기

실무 협업에서 풀 리퀘스트(PR) 리뷰 시간이 길어지는 가장 흔한 원인 중 하나는 **"세미콜론·들여쓰기·트레일링 공백·커밋 메시지 오타"** 같은 사소한 스타일 지적이다. `pre-commit` 단계에서 코드 포맷터·린터를 돌려 고칠 수 있는 것은 자동 수정하고 수정 불가한 에러는 커밋을 차단하며, `commit-msg` 단계에서 커밋 메시지 규칙까지 검증하면 PR 리뷰어는 오직 비즈니스 로직과 아키텍처에만 집중할 수 있다.

**[`pre-commit`](https://pre-commit.com/)** 프레임워크는 이 흐름을 `.pre-commit-config.yaml` 파일 하나로 완벽히 선언하고 관리하게 해 준다.

### 실전 `.pre-commit-config.yaml` 구성 예시 (`pre-commit` + `commit-msg`)

```yaml
# default_install_hook_types를 지정하면 pre-commit install만 쳐도 두 스테이지 모두 설치됨!
default_install_hook_types: [pre-commit, commit-msg]

repos:
  # 1. 공식 기본 품질 검사 (공백, 대용량 파일, YAML 문법, 머지 충돌 마커 잔존 확인)
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict

  # 2. 언어별 포맷터 · 린터 (예: Python용 Ruff — 초고속 린트 및 자동 포맷팅)
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  # 3. commit-msg 단계에서 Conventional Commits 규약 강제 (commitlint)
  - repo: https://github.com/alessandrojcm/commitlint-pre-commit-hook
    rev: v9.18.0
    hooks:
      - id: commitlint
        stages: [commit-msg]
        additional_dependencies: ['@commitlint/config-conventional']
```

### 실무자가 반드시 겪는 핵심 함정: `--hook-type commit-msg`

`pre-commit`을 처음 도입할 때 가장 많이 하는 실수가 바로 **`pre-commit install`만 실행하고 `commit-msg` 훅이 왜 안 도는지 헤매는 것**이다.

기본적으로 `pre-commit install`은 `.git/hooks/pre-commit`만 설치한다. 따라서 YAML 상단에 `default_install_hook_types: [pre-commit, commit-msg]`를 적어 두거나, 설치 시 `--hook-type` 옵션을 명시적으로 함께 넘겨야 한다.

```bash
# 1. pre-commit 패키지 설치 (Homebrew 또는 uv / pip)
$ brew install pre-commit

# 2. pre-commit과 commit-msg 훅을 모두 .git/hooks/에 연결
$ pre-commit install --hook-type pre-commit --hook-type commit-msg

# 3. (팁) 기존 저장소 전체 파일에 대해 한 번 훅 테스트 돌려 보기
$ pre-commit run --all-files

# 4. (팁) YAML에 적은 rev 버전들을 최신 안정 버전으로 일괄 업데이트
$ pre-commit autoupdate
```

이 설정이 끝나면 누군가 스타일이 엉망인 코드나 `WIP fix` 같은 커밋 메시지를 남기려 해도 **로컬 커밋 시점(`pre-commit` → `commit-msg`)에서 즉시 차단**된다. 참고로 `git commit` 중 자동 수정(`ruff --fix`, `trailing-whitespace`)이 파일을 고치면 안전을 위해 일단 커밋을 멈추므로, 수정된 파일을 다시 `git add`한 뒤 커밋을 재실행하면 깔끔하게 통과한다.

---

## 4. 직접 만들어 보는 10줄 `commit-msg` 훅 미니 실습

외부 프레임워크 없이 순수 Bash 스크립트만으로 Git이 훅을 어떻게 부르는지 확인해 보자. Git은 `commit-msg`를 실행할 때 첫 번째 인자(`$1`)로 `.git/COMMIT_EDITMSG` 경로를 넘겨준다.

```bash
# 1. 실행 가능한 훅 파일 생성 (.git/hooks/commit-msg)
$ cat << 'EOF' > .git/hooks/commit-msg
#!/usr/bin/env bash
MSG_FILE="$1"
FIRST_LINE=$(head -n 1 "$MSG_FILE")
PATTERN="^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9._-]+\))?!?: .+"

if ! echo "$FIRST_LINE" | grep -Eq "$PATTERN"; then
  echo "[Commit Policy Error] Conventional Commits 형식이 아닙니다: $FIRST_LINE" >&2
  echo "예시: feat(auth): add OAuth2 token refresh" >&2
  exit 1
fi
EOF

$ chmod +x .git/hooks/commit-msg
```

이제 잘못된 메시지로 커밋을 시도하면 exit code `1`에 의해 즉시 거절된다.

```bash
$ git commit -m "fix bug"
[Commit Policy Error] Conventional Commits 형식이 아닙니다: fix bug
예시: feat(auth): add OAuth2 token refresh

$ git commit -m "fix(auth): handle expired refresh token"
[main 4f8a1b2] fix(auth): handle expired refresh token
```

---

## 5. `--no-verify` 우회 한계와 서버 측 정책 강제 (8.4 현대화)

로컬 훅은 빠른 피드백이라는 엄청난 장점이 있지만, 보안·거버넌스 관점에서 **치명적인 맹점**이 하나 있다. 바로 개발자가 언제든 **`git commit --no-verify`(단축키 `-n`)** 또는 **`git push --no-verify`** 를 입력하면 로컬 훅 전체를 손쉽게 건너뛸 수 있다는 점이다.

따라서 Pro Git **8.4(An Example Git-Enforced Policy)** 가 강조하는 결론은 오늘날에도 유효하다. **"로컬 훅은 편의를 위한 1차 필터일 뿐, 진짜 조직 규칙은 반드시 중앙 서버에서 강제해야 한다."**

- **자체 호스팅 서버(`update` 훅)**: Pro Git 8.4는 서버 측 `update` 훅에서 ① 커밋 메시지 정규식 검사, ② 특정 핵심 디렉토리 수정 시 허용된 사용자 목록(ACL) 대조, ③ Non-Fast-Forward Push 거부를 구현하는 방법을 소개한다.
- **현대 클라우드 플랫폼(GitHub / GitLab)**: 오늘날 팀은 자체 쉘 스크립트를 서버에 심는 대신 **GitHub Rulesets(Branch Protection Rules)** + **`CODEOWNERS`**(경로별 시니어 승인 필수) + **CI 필수 상태 검사(Required Status Checks에서 `commitlint` 및 린터·테스트 실행)** 를 결합한다.

결국 가장 이상적인 엔지니어링 파이프라인은 **로컬(`pre-commit` · `Husky` · `Lefthook`)에서 수 초 만에 빠른 1차 방어**를 제공해 개발자의 시간을 아끼고, **서버(CI & Rulesets)에서 우회 불가능한 2차 최종 게이트**를 세우는 2중 방어 구조다.

---

## 마치며

이로써 **Pro Git 8장(Customizing Git)** 2부작(`#15`~`#16`)을 모두 마쳤다.

- **1부 (`#15`)**: `git config` 3계층과 `includeIf` 다중 계정 분리, iOS · Python 실무 바이너리 `.gitattributes` 템플릿, `clean`/`smudge` 필터와 Git LFS 원리
- **2부 (`#16`)**: 클라이언트·서버 Git Hooks 생명주기, `pre-commit` 프레임워크(`--hook-type commit-msg`)·Husky·Lefthook 활용법, 그리고 `--no-verify` 우회를 막는 2중 정책 방어선

Git을 단순히 남이 정해 준 대로 쓰는 단계를 넘어 저장소의 파일 규칙과 자동화 품질 관문까지 직접 설계할 수 있게 되었다면, 팀의 생산성과 코드베이스 안정성을 한 단계 끌어올릴 준비가 끝난 셈이다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 8.3 Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 8.4 An Example Git-Enforced Policy](https://git-scm.com/book/en/v2/Customizing-Git-An-Example-Git-Enforced-Policy)
- [pre-commit Official Documentation (pre-commit.com)](https://pre-commit.com/)
- [Git Hooks 제대로 쓰기: pre-commit & commit-msg로 품질 지키기 (Velog)](https://velog.io/@leeunduck/Git-Hooks-%EC%A0%9C%EB%8C%80%EB%A1%9C-%EC%93%B0%EA%B8%B0-pre-commit-commit-msg%EB%A1%9C-%ED%92%88%EC%A7%88-%EC%A7%80%ED%82%A4%EA%B8%B0)
- [GitHub Blog - Highlights from Git 2.54 (Config-Based Hooks)](https://github.blog/2026-04-20-highlights-from-git-2.54/)
- [GitHub Blog - Highlights from Git 2.55 (Parallel Hooks)](https://github.blog/2026-06-29-highlights-from-git-2.55/)
