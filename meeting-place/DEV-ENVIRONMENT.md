- 상태: 초안

# 개발 환경

- 작성일: 2026-10-20
- 출처: [../DEV-ENVIRONMENT.md](../DEV-ENVIRONMENT.md) (2026-10-19에 이 컴퓨터에서 확인한 도구 버전), [ARCHITECTURE.md](ARCHITECTURE.md), [AGENTS.md](AGENTS.md)
- 이 문서는 새로 확인한 것과 아직 확인하지 않은 것을 구분한다. 도구 버전은 루트 문서의 값을 그대로 쓰고 여기서 다시 확인하지 않았다.

## 1. 도구 (루트 문서에서 확인한 값)

| 도구 | 버전 | 비고 |
| --- | --- | --- |
| OS | Windows 11 Pro 10.0.22631 | |
| Node.js | v24.22.0 | 세션 PATH에 없을 수 있다. 전체 경로나 VS Code 재시작으로 해결 |
| npm | 11.19.1 | |
| Vite | 8.3.x | 루트 `frontend/`에서 확인 |
| React | 19.2.x | |
| Python | 3.14 (uv로 설치) | D-015 |
| uv | 0.13.1 | |
| FastAPI | 0.146.0 | |
| SQLite | 3.53.1 (Python 내장) | 따로 설치하지 않는다. |

## 2. 폴더

`meeting-place/backend/`와 `meeting-place/frontend/`는 아직 없다. 구현 단계에서 만든다. (D-001) 구조는 [ARCHITECTURE.md](ARCHITECTURE.md) 2·3장이다.

## 3. API 키 발급 체크리스트

키는 서버의 `.env`에만 넣고 커밋하지 않는다. 저장소에는 키 이름만 적은 `.env.example`을 둔다.

| 완료 | 서비스 | 필요한 키 | 어디에 두나 | 확인할 것 |
| --- | --- | --- | --- | --- |
| ☐ | 카카오 개발자 | 앱 만들기, REST API 키 | `.env` `KAKAO_REST_API_KEY` | 로컬 API 호출 한도 |
| ☐ | 카카오 개발자 | JavaScript 키 | 프런트엔드 `.env` `VITE_KAKAO_JS_KEY` | **플랫폼 > Web에 사이트 도메인 등록** (`http://localhost:5173` 포함). 도메인 제한이 핵심 보호다. |
| ☐ | Google Cloud | Places API (New) 사용 설정, API 키 | `.env` `GOOGLE_PLACES_API_KEY` | 결제 계정 연결 필요 여부, 무료 한도, 키에 API 제한 걸기 |
| ☐ | ODsay | 앱 등록, API 키 | `.env` `ODSAY_API_KEY` | 허용 도메인·IP 등록 방식, 일일 호출 한도 (A-04) |
| ☐ | TMAP (대안) | 앱 키 | `.env` `TMAP_APP_KEY` | ODsay가 막히면 쓴다. (D-016) |

`.env.example`에 들어갈 내용:

```
KAKAO_REST_API_KEY=
GOOGLE_PLACES_API_KEY=
ODSAY_API_KEY=
TMAP_APP_KEY=
```

- `.gitignore`에 `.env`, `backend/data/`가 들어 있어야 한다. 구현 전에 확인한다.
- 키를 채팅이나 이슈에 붙여 넣지 않는다.

## 4. 휴대폰에서 시험하기

- 위치 API(`navigator.geolocation`)는 HTTPS 또는 `localhost`에서만 동작한다. 컴퓨터의 `localhost`에서는 되지만 같은 와이파이의 휴대폰이 `http://컴퓨터IP:5173`으로 열면 위치 권한이 막힌다.
- 휴대폰 시험에는 HTTPS 터널이나 임시 배포가 필요하다. 어떤 방법을 쓸지는 Q-14에서 정한다.
- 카카오맵 JS 키의 허용 도메인에 시험에 쓰는 주소도 추가해야 한다.
- 컴퓨터만으로 여러 참가자를 흉내 내려면 브라우저 시크릿 창이나 서로 다른 브라우저를 쓴다. 참가 토큰이 브라우저마다 따로이기 때문이다. 위치는 개발자 도구의 위치 재정의로 바꿀 수 있다.

## 5. 실행 (구현 후 확인할 것)

아직 코드가 없어 확인하지 못한 명령이다. 구현하면서 실제로 돌려 보고 이 칸을 고친다.

| 할 일 | 명령(예정) | 확인 |
| --- | --- | --- |
| 백엔드 의존성 설치 | `cd meeting-place/backend` 후 `uv sync` | 미확인 |
| 백엔드 실행 | `uv run fastapi dev app/main.py` | 미확인 |
| 프런트엔드 의존성 설치 | `cd meeting-place/frontend` 후 `npm install` | 미확인 |
| 프런트엔드 실행 | `npm run dev` | 미확인 |
| 린트 | `npm run lint` | 미확인 |

## 6. 코드 작업 규칙 요약

- 브랜치는 `feature/<주제>`, 커밋은 `<종류>: <한국어 요약>`이다. 자세한 것은 [AGENTS.md](AGENTS.md).
- 구현 전에 계획을 적고 사용자가 확인한 뒤 코드를 쓴다.
