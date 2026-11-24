+++
date = '2026-12-04T09:00:00+09:00'
draft = false
title = 'Git 설정 가이드 #15: config와 .gitattributes'
description = 'Pro Git 8장 1부(8.1~8.2)를 바탕으로 config·includeIf, Python·iOS용 .gitattributes, clean/smudge와 Git LFS 원리를 정리한다.'
tags = ['Git', 'ProGit', 'GitConfig', 'GitAttributes', 'GitLFS', 'iOS', 'Python', 'Engineering']
categories = ['Engineering']
ShowToc = true
TocOpen = true
+++

7장까지 Git의 내부 객체와 협업 전략, 고급 복구·디버깅 도구를 모두 살펴보았다. 이제 **Pro Git 8장(Customizing Git)** 을 2부작으로 나누어, Git을 팀의 개발 환경과 프로젝트 특성에 맞게 커스터마이징하는 방법을 파헤친다.

이번 1부(**8.1~8.2**)에서는 다중 계정 자동 전환까지 포함한 **`git config` 계층 구조**를 짚고, 특히 실무에서 매번 마주치는 **`.gitattributes` 설정을 iOS/Xcode 프로젝트와 Python/AI 프로젝트의 필수 바이너리 파일 목록까지 포함해 완벽하게 정리**한 뒤, `clean`/`smudge` 필터와 Git LFS의 숨겨진 원리까지 연결해 본다.

---

## 1. `git config` 3계층 구조와 현대 실무 설정 (8.1)

Git 설정은 적용 범위에 따라 3단계 파일로 나뉘며, **아래 계층이 위 계층의 값을 덮어쓴다(Override).**

1. **`--system`** (`/etc/gitconfig`): 시스템 전체 사용자·저장소 공통 기본값
2. **`--global`** (`~/.gitconfig` 또는 `~/.config/git/config`): 현재 로그인한 OS 사용자 기본값
3. **`--local`** (`.git/config`): 특정 저장소 1곳에만 적용되는 최우선 설정 (옵션 생략 시 기본값)

특정 설정이 대체 어디 파일에서 읽어 들여진 것인지 헷갈릴 때는 `--show-origin`을 붙이면 경로까지 바로 나온다.

```bash
$ git config --list --show-origin
file:/opt/homebrew/etc/gitconfig  credential.helper=osxkeychain
file:/Users/me/.gitconfig         user.name=Wooyoung Choi
file:.git/config                  remote.origin.url=https://...
```

{{< figure src="gitconfig-and-attributes-matrix.svg" alt="git config 3계층 우선순위, includeIf 다중 계정 자동 전환, 그리고 공통·iOS·Python 프로젝트별 .gitattributes 분류 다이어그램" caption="git config 3계층과 includeIf 계정 분리(상단) 및 실무 프로젝트별(공통·iOS·Python/AI) .gitattributes 핵심 분류(하단) (Pro Git 8.1~8.2 재구성)" width="820" height="540" >}}

### 회사 계정과 개인 계정을 자동으로 갈아끼우는 `includeIf`

실무에서 가장 많이 겪는 사고 중 하나가 회사 레포지토리에 개인 Gmail 주소로 커밋하거나, 반대로 개인 오픈소스에 회사 이메일로 커밋하는 일이다. Git `2.13+`부터는 **`includeIf "gitdir:..."`** 조건문을 쓰면 디렉토리 위치만으로 이름·이메일·서명 키가 자동 전환된다.

```ini
# ~/.gitconfig (기본 개인 계정)
[user]
    name = Wooyoung Choi
    email = dev.personal@gmail.com

# ~/work/ 폴더 하위의 모든 저장소에만 사내 프로필 자동 덮어쓰기
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
```

```ini
# ~/.gitconfig-work (사내 계정 전용)
[user]
    name = Wooyoung Choi
    email = wy.choi@company.com
    signingkey = ~/.ssh/id_ed25519_work.pub
```

이 외에 팀 생산성을 올려 주는 기본 설정으로는 커밋 메시지 가이드를 미리 띄우는 **`commit.template`**, 전역 무시 목록 **`core.excludesfile`**, 오타 명령을 자동 보정하는 **`help.autocorrect`**(`prompt` 권장), 그리고 서버 저장소 보호용 `receive.fsckObjects = true`(손상 객체 수신 거부) 및 `receive.denyNonFastForwards = true` 등이 있다.

---

## 2. 실무 필수 `.gitattributes` 완벽 가이드: 개행 통일부터 iOS · Python 바이너리까지 (8.2)

개인 PC마다 설정하는 `git config core.autocrlf`와 달리, **`.gitattributes`는 저장소 루트에 함께 커밋되어 팀원 전원과 CI 서버에 100% 동일하게 강제 적용되는 경로별 규칙 파일**이다.

Git은 기본적으로 파일 앞부분 몇 KB만 훑어바이너리 여부를 추측한다. 만약 이 휴리스틱 추측이 빗나가면 바이너리 파일에 **줄바꿈 문자(`LF` ↔ `CRLF`) 변환을 걸어 파일을 영구 손상**시키거나, 거대한 의미 없는 텍스트 Diff를 뿜어내며 Merge 충돌을 일으킨다. 따라서 프로젝트 성격에 맞게 `.gitattributes`를 초기에 선언해 두는 것이 실무의 기본이다.

### ① 모든 프로젝트 공통: 개행 문자(EOL)와 Lockfile 규약

```gitattributes
# 모든 텍스트 파일의 저장소 내 줄바꿈을 자동으로 LF 정규화 (OS 혼합 팀 필수)
* text=auto eol=lf

# 쉘/빌드 스크립트는 Windows에서 체크아웃해도 반드시 LF 유지 (컨테이너 실행 오류 방지)
*.sh text eol=lf
Dockerfile text eol=lf
Makefile text eol=lf

# Windows 전용 배치 파일은 CRLF 고정
*.bat text eol=crlf
*.ps1 text eol=crlf

# 거대 Lockfile은 GitHub PR 화면에서 기본 접힘 처리 및 언어 통계 제외
package-lock.json linguist-generated=true
pnpm-lock.yaml linguist-generated=true
yarn.lock linguist-generated=true
```

*(참고: 기존 저장소에 `.gitattributes`를 뒤늦게 추가했다면 **`git add --renormalize .`** 을 한 번 실행해 이미 들어 있던 파일들의 개행까지 일괄 재정규화해야 한다.)*

---

### ② iOS / Xcode 프로젝트에서 반드시 지정해야 하는 바이너리 · 특수 파일

iOS 앱과 macOS 생태계는 IDE 설정 파일, 이미지 에셋 카탈로그, 서명 인증서, 정적/동적 프레임워크 등 **바이너리 보호와 병합 보호가 동시에 필요한 특수 파일**이 매우 많다.

| 파일 / 패턴 | 지정할 `.gitattributes` 속성 | 실무적 이유 |
| :--- | :--- | :--- |
| **`*.pbxproj`** | `binary merge=union` | Xcode 프로젝트 파일은 텍스트지만 24자리 UUID 라인이 뒤엉켜 일반 3-Way Merge 시 구문 파손이 잦다 *(가장 추천되는 해법은 XcodeGen/Tuist 도입)* |
| **`*.xcassets/**`**, `*.png`, `*.pdf`, `*.icns` | `binary` (`-text -diff`) | 에셋 카탈로그와 아이콘·이미지 리소스의 개행 손상 및 Diff 폭증 차단 |
| **`*.ttf`, `*.otf`, `*.woff2`** | `binary` | 커스텀 폰트 바이너리 변조 방지 |
| **`*.xcframework/**`**, `*.framework/**`, `*.a`, `*.dylib` | `binary` *(대용량은 `filter=lfs`)* | 서드파티 바이너리 프레임워크·라이브러리 보존 |
| **`*.ipa`, `*.mobileprovision`, `*.p12`, `*.cer`** | `binary` | 앱 바이너리, 프로비저닝 프로파일, 서명 인증서 무결성 보장 *(참고: 비밀키 `.p12`는 가능하면 저장소 외부 시크릿 매니저 보관 권장)* |
| **`*.storyboard`, `*.xib`** | `text eol=lf` *(또는 `binary`)* | XML 기반 UI 파일 (수동 병합이 사실상 불가능할 만큼 복잡하다면 `binary`로 지정해 실수 예방) |
| **`Podfile.lock`, `Package.resolved`** | `linguist-generated=true` | PR 코드 리뷰 시 의존성 잠금 파일이 화면을 덮는 현상 방지 |

#### 복붙 가능한 iOS / Swift 실전 `.gitattributes` 템플릿

```gitattributes
* text=auto eol=lf

# Xcode 프로젝트 및 의존성 잠금 파일
*.pbxproj binary merge=union
Podfile.lock linguist-generated=true
Package.resolved linguist-generated=true

# 이미지 · 에셋 · 폰트 · 미디어 바이너리
*.png binary
*.jpg binary
*.jpeg binary
*.webp binary
*.pdf binary
*.icns binary
*.ttf binary
*.otf binary
*.mp3 binary
*.mp4 binary

# 컴파일 산출물 · 라이브러리 · 인증서 · 프로비저닝
*.a binary
*.dylib binary
*.so binary
*.xcframework/** binary
*.ipa binary
*.mobileprovision binary
*.p12 binary
*.cer binary
```

---

### ③ Python / AI · 데이터 사이언스 프로젝트에서 반드시 지정해야 하는 바이너리 파일

Python 및 AI/ML 프로젝트는 가벼운 `.py` 스크립트 옆에 **모델 체크포인트, 직렬화 객체, 데이터셋, 오프라인 패키지 휠(`.whl`), 그리고 거대한 JSON 덩어리인 Jupyter Notebook(`.ipynb`)** 이 공존한다.

| 파일 / 패턴 | 지정할 `.gitattributes` 속성 | 실무적 이유 |
| :--- | :--- | :--- |
| **`*.pkl`, `*.pickle`, `*.joblib`** | `binary` | Python 직렬화 바이너리 파일은 바이트 하나만 달라져도 `UnpicklingError`로 즉시 깨짐 |
| **`*.pt`, `*.pth`, `*.onnx`, `*.safetensors`, `*.bin`** | `binary` *(+ `filter=lfs`)* | PyTorch · ONNX · LLM 모델 가중치 파일 보호 및 대용량 분리 |
| **`*.h5`, `*.hdf5`, `*.parquet`, `*.feather`, `*.npy`, `*.npz`** | `binary` *(+ `filter=lfs`)* | NumPy · Pandas · HDF5 학습 데이터셋 바이너리 보존 |
| **`*.whl`, `*.tar.gz`, `*.zip`, `*.so`, `*.pyd`** | `binary` | Python 배포 휠 패키지, Cython/Rust 네이티브 확장 모듈(`.so`/`.pyd`) |
| **`*.sqlite`, `*.sqlite3`, `*.db`** | `binary` | 로컬 테스트/피처용 내장 SQLite 바이너리 DB |
| **`*.ipynb`** | `filter=nbstripout` *(또는 `diff=ipynb`)* | 셀 실행 번호(`execution_count`)와 출력 이미지(base64) 때문에 발생하는 사소한 JSON 충돌 방지 |
| **`poetry.lock`, `uv.lock`, `Pipfile.lock`** | `linguist-generated=true` | PR 리뷰 노이즈 제거 및 GitHub 언어 통계 왜곡 차단 |

#### 복붙 가능한 Python / AI 실전 `.gitattributes` 템플릿

```gitattributes
* text=auto eol=lf
*.py text eol=lf

# Python / AI 직렬화 객체 · 데이터 · 확장 바이너리
*.pkl binary
*.pickle binary
*.joblib binary
*.npy binary
*.npz binary
*.parquet binary
*.h5 binary
*.hdf5 binary
*.sqlite binary
*.sqlite3 binary
*.db binary
*.whl binary
*.so binary
*.pyd binary

# 대용량 모델 가중치 (Git LFS 설치 후 git lfs track 연동 권장)
*.pt filter=lfs diff=lfs merge=lfs -text
*.pth filter=lfs diff=lfs merge=lfs -text
*.onnx filter=lfs diff=lfs merge=lfs -text
*.safetensors filter=lfs diff=lfs merge=lfs -text

# Jupyter Notebook 실행 출력 셀 자동 제거 (nbstripout --install 연동 시)
# *.ipynb filter=nbstripout

# Lock 파일 PR 접기
poetry.lock linguist-generated=true
uv.lock linguist-generated=true
Pipfile.lock linguist-generated=true
```

### 알아두면 쓸모 있는 추가 속성과 디버깅

- **`export-ignore`**: `git archive`로 배포용 소스 압축 파일을 만들 때 `.github/`, `tests/` 같은 개발 전용 폴더를 제외한다.
- **`merge=ours`**: `database.config`처럼 각 브랜치(운영/스테이징)의 설정 파일을 덮어쓰지 않고 항상 내 브랜치 값을 지킬 때 유용하다 *(단, `git config merge.ours.driver true` 사전 등록 필요)*.
- **속성 디버깅**: 특정 파일에 어떤 `.gitattributes` 규칙이 실제로 적용 중인지 헷갈릴 때는 아래 명령어로 즉시 검증할 수 있다.

```bash
$ git check-attr --all -- assets/model.safetensors
assets/model.safetensors: text: unset
assets/model.safetensors: diff: lfs
assets/model.safetensors: merge: lfs
assets/model.safetensors: filter: lfs
```

---

## 3. `clean` / `smudge` 필터와 Git LFS의 숨겨진 내부 원리 (8.2)

Pro Git 8.2의 하이라이트 중 하나는 `.gitattributes`를 통해 **체크아웃(`smudge`)과 스테이징(`clean`) 시점에 파일 내용을 원하는 스크립트로 통과시켜 자동 변환하는 필터 파이프라인**이다.

{{< figure src="clean-smudge-and-lfs.svg" alt="Git Attributes의 smudge/clean 필터 양방향 파이프라인 및 Git LFS 포인터 치환 구조 다이어그램" caption="체크아웃(smudge)과 스테이징(clean) 시점의 콘텐츠 변환 파이프라인(상단) 및 filter=lfs 기반 Git LFS 포인터 작동 원리(하단) (Pro Git 8.2 재구성)" width="820" height="480" >}}

1. **`clean` 필터 (`Working Tree → Git DB`)**: 개발자가 `git add`를 실행할 때 작동한다. 위 Python 예시의 `nbstripout`처럼, 디스크에 있는 `.ipynb` 파일에서 출력 이미지와 실행 횟수 JSON 필드를 깔끔하게 비운 결과만 `.git/objects`에 커밋한다.
2. **`smudge` 필터 (`Git DB → Working Tree`)**: 반대로 `git checkout`으로 파일을 디스크에 꺼낼 때 작동한다. 암호화된 설정 템플릿을 로컬 환경 변수 값으로 복호화해 채워 넣는 식의 작업에 활용할 수 있다.

놀랍게도 현대 AI·게임·모바일 개발에서 거대한 바이너리를 다루는 **Git LFS(Large File Storage) 역시 별도의 마법이 아니라 바로 이 `.gitattributes`의 `clean` / `smudge` 필터 위에서 돌아간다.**

`git lfs track "*.safetensors"`를 실행하면 `.gitattributes`에 `filter=lfs diff=lfs merge=lfs -text` 한 줄이 기록된다. 이후 개발자가 2.4GB짜리 모델 파일을 `git add` 하면 **`clean` 필터가 작동해 실제 2.4GB 바이너리 대신 고작 130바이트짜리 SHA-256 텍스트 포인터(`oid sha256:...`)만 Git 저장소에 커밋**하고, 반대로 다른 팀원이 `git checkout` 하면 **`smudge` 필터가 그 포인터를 읽어 별도 LFS HTTP 서버에서 실제 2.4GB 바이너리를 내려받아 워킹 트리에 복원**해 준다.

---

## 마치며

이번 **8장 1부(`#15`)** 에서는 `git config`의 계층과 `includeIf` 다중 계정 분리, OS 혼합 및 **iOS · Python 실무 환경에서 반드시 보호해야 할 `.gitattributes` 바이너리·노트북·Lockfile 규칙**, 그리고 `clean`/`smudge` 필터와 Git LFS의 연결 고리까지 정리했다.

이어지는 [8장 2부(#16)](/posts/pro-git-internals-ch8-part2/)에서는 특정 Git 이벤트마다 스크립트를 실행하는 **Git Hooks**의 생명주기와 함께, 다언어 실무 표준인 **`pre-commit` 프레임워크(`pre-commit.com`)** · Husky · Lefthook 활용법, 그리고 로컬 `--no-verify` 우회를 막는 서버 측 정책 강제 전략(Pro Git 8.3~8.4)을 깊게 파헤친다.

---

## 참고 자료

- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 8.1 Git Configuration](https://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration)
- [Scott Chacon & Ben Straub, *Pro Git* 2nd Edition - 8.2 Git Attributes](https://git-scm.com/book/en/v2/Customizing-Git-Git-Attributes)
- [Git Documentation - gitattributes](https://git-scm.com/docs/gitattributes)
- [Git LFS (Large File Storage) Specification](https://git-lfs.com/)
