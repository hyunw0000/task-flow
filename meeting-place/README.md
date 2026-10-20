# 미팅플레이스

친구나 동료 2~6명이 서로 다른 곳에서 출발해 만날 장소를 정하는 웹 앱이다.

1. 모임장이 모임 이름과 상황(점심, 저녁, 2차)을 정해 모임을 만들고 링크를 채팅방에 공유한다.
2. 참가자는 링크를 열어 이름만 입력하면 된다. 회원가입이나 로그인은 없다.
3. 참가자가 동의하면 서로의 현재 위치가 지도에 실시간으로 보인다. 그래서 "지금 어디쯤이야?"라고 물을 필요가 없다.
4. 서버가 모두의 대중교통 이동시간을 계산해, 가장 먼 사람의 이동시간이 가장 짧은 지하철역이나 번화가를 기준 지점으로 잡는다.
5. 기준 지점 주변의 밥집, 술집, 카페를 평점, 거리, 이동시간 공평도 점수순으로 추천한다.
6. 마음에 드는 장소를 후보로 올리면 참가자가 1인 1표로 투표한다.
7. 모임장이 장소를 확정하면 위치 데이터가 즉시 지워진다. 모임은 24시간 뒤 자동으로 만료된다.
8. 비슷한 서비스는 중간 지점만 한 번 계산해 보여 주는 데 그친다. 이 앱은 실시간 위치와 장소 추천을 중심에 둔다.
9. 기술은 React와 Vite, FastAPI, SQLite를 쓰고, 지도와 장소 정보는 카카오, 평점은 Google, 이동시간은 ODsay 공식 API로 가져온다.
10. 지금은 기획과 설계 문서만 있고 코드는 아직 없다.

## 문서

| 문서 | 내용 |
| --- | --- |
| [PRD.md](PRD.md) | 목표, 범위, 요구사항, 열린 질문 |
| [SCOPE-BOUNDARY.md](SCOPE-BOUNDARY.md) | 만드는 것과 만들지 않는 것 |
| [USER-WORKFLOW.md](USER-WORKFLOW.md) | 모임 만들기부터 확정까지의 흐름 |
| [SCREEN-SPEC.md](SCREEN-SPEC.md) | 화면 목록과 각 화면의 내용 |
| [DATA-MODEL.md](DATA-MODEL.md) | 저장하는 데이터 구조 |
| [BUSINESS-RULES.md](BUSINESS-RULES.md) | 위치, 기준 지점, 추천, 투표 규칙 |
| [API-SPEC.md](API-SPEC.md) | 서버 API와 외부 API 연동 |
| [ARCHITECTURE.md](ARCHITECTURE.md) | 전체 구조와 실시간 전달 방식 |
| [DECISIONS.md](DECISIONS.md) | 결정 기록 |
| [DEV-ENVIRONMENT.md](DEV-ENVIRONMENT.md) | 개발 환경과 API 키 발급 체크리스트 |
| [AGENTS.md](AGENTS.md) | 사람과 AI 에이전트가 따르는 작업 규칙 |
| [WORKFLOW.md](WORKFLOW.md) | 진행 순서 |
