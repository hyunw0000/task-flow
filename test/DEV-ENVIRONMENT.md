# 개발 환경

- 확인일: 2026-10-19
- 확인한 곳: 이 컴퓨터, VS Code 안의 Claude Code 세션
- 출처: [ARCHITECTURE.md](ARCHITECTURE.md), [PRODUCT-DIRECTION.md](PRODUCT-DIRECTION.md) 의 기술 표준 경로 (화면 React · Vite, API FastAPI, 저장소 SQLite)

## 1. 운영체제

| 항목 | 값 |
| --- | --- |
| 이름 | Microsoft Windows 11 Pro |
| 버전 | 10.0.22631 (빌드 22631) |
| 아키텍처 | 64비트 (x86_64) |

## 2. 셸

| 셸 | 버전 | 비고 |
| --- | --- | --- |
| Windows PowerShell | 5.1.22621.2506 | 기본 셸 |
| Git Bash (MSYS2) | bash 5.3.15(2)-release | `MINGW64_NT-10.0-22631 3.6.10` |

## 3. 도구별 버전

각 명령의 `--version` 출력을 그대로 옮겼다. 명령을 찾지 못한 경우도 출력 그대로 적었다.

### git

```
$ git --version
git version 2.56.0.windows.2
```

- 위치: `/ucrt64/bin/git` (Git Bash 에서만 잡힌다. PowerShell 에서는 `git` 을 찾지 못했다.)

### node

```
$ node --version
/usr/bin/bash: line 1: node: command not found
```

- 설치는 되어 있다. 전체 경로로 실행하면 다음과 같다.

```
$ "/c/Program Files/nodejs/node.exe" --version
v24.22.0
```

- 시스템 PATH 에는 `C:\Program Files\nodejs\` 가 들어 있다. 지금 VS Code 세션의 PATH 에만 빠져 있다. VS Code 를 완전히 종료하고 다시 열면 잡힐 것으로 본다.

### npm

```
$ npm --version
/usr/bin/bash: line 1: npm: command not found
```

- node 와 같은 이유다. 전체 경로를 PATH 앞에 넣고 실행하면 다음과 같다.

```
$ npm --version
11.19.1
```

### uv

```
$ uv --version
uv 0.13.1 (62faed335 2026-10-13 x86_64-pc-windows-msvc)
```

- 위치: `C:\Users\EZ\.local\bin\uv.exe`

### python

```
$ python --version
Python
```

- 종료 코드: 49 (Git Bash), 9009 (PowerShell). 실제 Python 이 아니라 Microsoft Store 앱 실행 별칭(`C:\Users\EZ\AppData\Local\Microsoft\WindowsApps\python.exe`)이다.
- uv 가 관리하는 Python 은 하나 있다.

```
$ uv python list --only-installed
cpython-3.15.0-windows-x86_64-none    C:\Users\EZ\AppData\Roaming\uv\python\cpython-3.15-windows-x86_64-none\python.exe

$ "C:\Users\EZ\AppData\Roaming\uv\python\cpython-3.15-windows-x86_64-none\python.exe" --version
Python 3.15.0
```

- 이 Python 에 들어 있는 SQLite 버전은 3.53.1 이다. SQLite 는 따로 설치하지 않는다.

## 4. 새로 설치한 것

2026-10-19 에 설치했다. Python 3.14 와 uv 를 쓰기로 한 결정은 [DECISIONS.md](DECISIONS.md) D-016 이다.

### Python 3.14 (uv)

```
$ uv python install 3.14
Installed Python 3.14.8 in 3.88s
 + cpython-3.14.8-windows-x86_64-none (python3.14.exe)
```

### API: `backend/`

```
$ uv init --bare --python 3.14 --name task-flow-backend
$ uv python pin 3.14
$ uv add "fastapi[standard]"
```

설치 뒤 확인한 버전:

```
$ uv run python --version
Python 3.14.8

$ uv run fastapi --version
FastAPI CLI version: 0.0.32
FastAPI Cloud CLI version: 0.26.0
```

| 패키지 | 버전 |
| --- | --- |
| fastapi | 0.146.0 |
| uvicorn | 0.54.0 |
| sqlite (Python 내장) | 3.53.1 |

- 만든 파일: `backend/pyproject.toml`, `backend/uv.lock`, `backend/.python-version`, `backend/.venv/` (git 에 올리지 않는다)

### 화면: `frontend/`

```
$ npm create vite@latest frontend -- --template react --no-interactive
$ cd frontend && npm install
added 24 packages, and audited 25 packages in 9s
found 0 vulnerabilities

$ npx vite --version
vite/8.3.4 win32-x64 node-v24.22.0
```

| 패키지 | 버전 |
| --- | --- |
| vite | 8.3.4 |
| react · react-dom | 19.3.1 |
| @vitejs/plugin-react | 6.1.2 |
| oxlint | 1.88.0 |

- `oxlint` 는 Vite React 템플릿에 기본으로 들어 있는 코드 검사 도구다. 표준 경로에 없는 도구라 계속 둘지 정해야 한다.
- 만든 파일: `frontend/` 전체 (Vite 템플릿 그대로), `frontend/node_modules/` (git 에 올리지 않는다)

### 그 밖

- 프로젝트 맨 위에 `.gitignore` 를 만들었다: `backend/.venv/`, `backend/data/`, `__pycache__/`, `node_modules/`

## 5. 남은 일

| 할 일 | 상태 |
| --- | --- |
| node · npm 이 세션 PATH 에서 잡히게 하기 (VS Code 완전히 종료 후 다시 열기). 이번 설치는 PATH 앞에 `C:\Program Files\nodejs` 를 임시로 넣고 실행했다 | 대기 |
| Vite 템플릿의 `oxlint` 를 둘지 정하기 | 사용자 결정 대기 |
| ARCHITECTURE.md 2 · 3장 구조에 맞게 템플릿 파일 정리, `/api` 프록시 설정 (D-010) | 다음 단계 |
