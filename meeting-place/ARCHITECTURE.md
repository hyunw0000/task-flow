- 상태: 초안

# 아키텍처: 미팅플레이스

- 작성일: 2026-10-20
- 출처: [PRD.md](PRD.md), [DECISIONS.md](DECISIONS.md), [DATA-MODEL.md](DATA-MODEL.md), [API-SPEC.md](API-SPEC.md)
- 이 문서의 (D-xxx)는 [DECISIONS.md](DECISIONS.md)의 결정 번호, R-xxx는 PRD 요구사항이다.

## 1. 전체 모양

```
┌────────────────────┐  HTTP(JSON) + SSE  ┌────────────────────┐   SQL   ┌────────────────┐
│ 화면               │ ─────────────────▶ │ API                │ ──────▶ │ 저장소         │
│ React · Vite       │ ◀───────────────── │ FastAPI            │ ◀────── │ SQLite 파일 1개 │
│ (휴대폰 브라우저)  │      /api/...      │                    │         └────────────────┘
└─────────┬──────────┘                    └─────────┬──────────┘
          │ 지도 SDK                                │ 서버끼리 HTTPS
          ▼                                         ▼
   카카오맵 JS SDK                   카카오 로컬 · Google Places · ODsay
```

- 화면은 외부 API(지도 SDK 제외)와 저장소에 직접 닿지 않는다. (D-003)
- 모임 상태의 원본은 저장소다. 화면이 들고 있는 상태는 서버가 보낸 사본이다.
- 개발 중에는 Vite 개발 서버가 `/api` 요청을 FastAPI로 넘긴다.

## 2. 화면 구조

```
meeting-place/frontend/
├── index.html
├── vite.config.js           # /api 요청을 API로 넘기는 설정
└── src/
    ├── main.jsx
    ├── App.jsx              # 경로(/, /m/:id)에 따라 화면을 고른다
    ├── api/
    │   └── client.js        # fetch와 SSE 연결을 모은 곳. 다른 곳은 fetch를 직접 쓰지 않는다
    ├── hooks/
    │   ├── useMeeting.js    # SSE로 모임 상태를 받아 들고 있다
    │   └── useShareLocation.js  # 위치 권한, 10초마다 전송, 켜고 끄기
    ├── lib/
    │   └── session.js       # 모임별 참가 토큰을 localStorage에 저장·읽기
    ├── pages/
    │   ├── CreateMeeting.jsx    # S-01
    │   ├── JoinMeeting.jsx      # S-02
    │   ├── MeetingRoom.jsx      # S-03
    │   └── Notice.jsx           # S-04
    └── components/
        ├── Map.jsx              # 카카오맵 래퍼
        ├── ShareBar.jsx         # 위치 공유 바
        ├── PlaceList.jsx        # 추천 탭
        ├── CandidateList.jsx    # 후보 탭
        └── PeopleList.jsx       # 사람 탭
```

- 화면 상태는 React 기본 기능(`useState`, 훅)만으로 관리한다. (D-002)
- 모임 상태는 `useMeeting` 한 곳이 SSE로 받아 하위 컴포넌트에 내려 준다. 변경 요청은 서버가 성공하면 SSE로 새 상태가 오므로 화면이 미리 바꾸지 않는다.
- 추천 목록은 SSE에 싣지 않고 추천 탭이 열려 있을 때 `GET /places`로 따로 받는다. 기준 지점이 바뀌면 다시 받는다.

## 3. API 구조

```
meeting-place/backend/
├── app/
│   ├── main.py          # FastAPI 앱, 시작할 때 저장소 준비, 만료 정리 작업 시작
│   ├── routes/
│   │   ├── meetings.py      # 모임·참가·상태·잠금·끝내기
│   │   ├── location.py      # 공유 켜기·끄기, 위치 보내기
│   │   ├── places.py        # 추천, 이동시간
│   │   ├── candidates.py    # 후보·투표·확정
│   │   └── events.py        # SSE
│   ├── schemas.py       # 요청·응답 모양과 입력 검증 (BR-01~09)
│   ├── auth.py          # 토큰 해시, 모임장·참가자 판단 (D-004)
│   ├── repository.py    # SQL은 여기에만 둔다
│   ├── hub.py           # 모임별 SSE 구독자 목록, 변경 알림
│   ├── services/
│   │   ├── pivot.py         # 기준 지점 계산 (BR-10·11·15·24)
│   │   ├── recommend.py     # 추천 점수 (BR-12·13·14·25)
│   │   ├── kakao.py         # 카카오 로컬 API
│   │   ├── google_places.py # 평점·영업 정보 (BR-26)
│   │   ├── transit.py       # ODsay (대안 TMAP) (D-016)
│   │   └── cache.py         # cache_entries 읽기·쓰기 (BR-21)
│   └── cleanup.py       # 만료된 모임 삭제 (BR-20)
├── data/                # SQLite 파일 (커밋하지 않는다)
├── .env.example         # 키 이름만 적는다
└── pyproject.toml
```

- 요청 흐름: `routes` → `schemas`로 검증 → `repository`·`services` → 응답. 비즈니스 규칙은 `services`와 `routes`에 두고 SQL은 `repository`에만 둔다.
- 외부 API는 `services/` 아래 파일에서만 부른다. 다른 곳에서 `httpx` 등으로 직접 부르지 않는다. 호출은 비동기로 한다.
- 모든 외부 호출은 `cache.py`를 거친다. (D-012)

## 4. 저장소

- SQLite 파일 하나. 테이블은 [DATA-MODEL.md](DATA-MODEL.md)의 다섯 개다.
- 외래 키를 켜고, 모임을 지우면 하위 데이터가 따라 지워지게 한다.
- 동시 쓰기가 많지 않아 WAL 모드로 충분하다. 서버 프로세스는 하나로 시작한다. (SSE 구독자 목록이 메모리에 있기 때문이다)

## 5. 실시간 전달

```
참가자 A ─ PUT /me/location (10초마다) ─▶ API ─ 저장 ─▶ hub.notify(모임)
                                                         │
참가자 B·C ◀── SSE event: state (전체 상태) ◀────────────┘
```

1. 위치가 들어오면 `repository`에 덮어쓰고, 기준 지점 재계산 조건(BR-15)을 본다. 조건이 되면 백그라운드에서 계산한다.
2. `hub`가 그 모임의 구독자 모두에게 각자 기준의 `MeetingState`를 만들어 보낸다.
3. 짧은 시간에 알림이 몰리면 1초에 한 번으로 묶어서 보낸다.
4. 연결이 끊기면 화면이 "다시 연결 중"을 보이고, `EventSource`가 자동으로 다시 붙는다. 다시 붙으면 전체 상태를 다시 받는다.

한계: 서버 프로세스가 둘 이상이면 `hub`가 공유되지 않는다. 1차는 프로세스 하나다. 늘려야 하면 D-005를 다시 본다.

## 6. 기준 지점 계산 흐름

1. 쓸 수 있는 위치를 모은다. 2곳 미만이면 끝낸다. (BR-10)
2. 평균 좌표 근처에서 카카오 로컬 API로 지하철역·번화가 후보를 5~10곳 뽑는다.
3. 후보 × 참가자 이동시간을 `transit.py`로 구한다. 참가자 6명 × 후보 10곳이면 최대 60번이라 캐시와 병렬 호출로 줄인다.
4. 최대 이동시간이 가장 짧은 후보를 고르고 `meetings.pivot_*`에 쓴다. (BR-11)
5. 실패하면 임시 기준 지점을 쓰고 `approximate`를 true로 한다. (BR-24)

## 7. 보안과 개인정보

| 위험 | 대응 |
| --- | --- |
| API 키 유출 | 서버 `.env`에만 둔다. 카카오맵 JS 키만 브라우저에 두고 도메인 제한. (D-003) |
| 링크가 새서 낯선 사람이 들어옴 | 모임장이 내보내기·잠금을 쓴다. 모임 id는 무작위 10자다. (D-013) |
| 토큰 탈취 | 서버에는 해시만 저장한다. 모임이 끝나면 토큰도 지워진다. SSE 쿼리 토큰은 서버 로그에 남기지 않는다. |
| 위치 과다 수집 | 동의 전 수신 금지, 이력 저장 금지, 확정·종료·만료 때 삭제. (BR-05·09·20·23) |
| 호출 폭주 | 위치 갱신 5초 제한, 외부 API 한도 80%에서 멈춤. (BR-08·30) |
| 평문 통신 | 위치 API가 HTTPS를 요구하므로 공개할 때는 HTTPS만 쓴다. (Q-14) |

## 8. 하지 않는 것

- 로그인, 사용자 계정, 모임 재사용
- 다중 서버, 메시지 큐, 외부 DB
- 위치 이력과 이동 경로 저장

## 9. 열린 문제

| 항목 | 메모 |
| --- | --- |
| 배포 위치와 HTTPS | Q-14. 개발 중 휴대폰 시험 방법은 [DEV-ENVIRONMENT.md](DEV-ENVIRONMENT.md) 4장 |
| 위치정보법 | Q-03. 공개 전에 확인한다. |
| ODsay 응답 구조 | 실제 호출로 확인하기 전에는 `transit.py`의 반환 모양이 바뀔 수 있다. |
