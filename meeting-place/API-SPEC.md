- 상태: 초안

# API 명세

- 출처: [PRD.md](PRD.md), [DECISIONS.md](DECISIONS.md) D-004·D-005·D-014, [ARCHITECTURE.md](ARCHITECTURE.md) 3장, [BUSINESS-RULES.md](BUSINESS-RULES.md), [DATA-MODEL.md](DATA-MODEL.md)
- 모든 경로는 `/api` 로 시작하고 본문은 JSON이다. 좌표는 `lat`, `lng` 실수다.
- 인증: 모임 만들기·참가가 돌려준 `token`을 `Authorization: Bearer <token>` 으로 보낸다. 예외는 미리보기(2번)와 SSE(5번, 쿼리 `?token=`)다. (D-004, BR-22)
- 오류 본문은 `{ "detail": ... }` 이다. 상태 코드: 422 입력 오류, 401 토큰 없음·틀림, 403 권한 없음, 404 없음, 409 상태 충돌, 429 호출 과다, 502 외부 API 실패. (D-014)

## 1. 한눈에 보기

| # | 행동 | 요청 | 성공 | 권한 | 요구사항 |
| --- | --- | --- | --- | --- | --- |
| 1 | 모임 만들기 | `POST /api/meetings` | 201 | 없음 | R-001 |
| 2 | 모임 미리보기 | `GET /api/meetings/{id}/preview` | 200 | 없음 | R-002 |
| 3 | 참여 | `POST /api/meetings/{id}/participants` | 201 | 없음 | R-002 |
| 4 | 모임 상태 받기 | `GET /api/meetings/{id}` | 200 | 참가자 | R-003 |
| 5 | 실시간 구독 | `GET /api/meetings/{id}/events?token=` | 200 (SSE) | 참가자 | R-003 |
| 6 | 공유 켜기·끄기 | `PUT /api/meetings/{id}/me/sharing` | 200 | 참가자 | R-002, R-004 |
| 7 | 내 위치 보내기 | `PUT /api/meetings/{id}/me/location` | 204 | 참가자 | R-003 |
| 8 | 기준 지점 고정·해제 | `PUT /api/meetings/{id}/pivot` | 200 | 모임장 | R-005 |
| 9 | 기준 지점 이동시간 | `GET /api/meetings/{id}/pivot/travel` | 200 | 참가자 | R-008 |
| 10 | 추천 목록 | `GET /api/meetings/{id}/places` | 200 | 참가자 | R-006, R-007 |
| 11 | 장소까지 이동시간 | `GET /api/meetings/{id}/places/{kakao_place_id}/travel` | 200 | 참가자 | R-008 |
| 12 | 후보 올리기 | `POST /api/meetings/{id}/candidates` | 201 | 참가자 | R-009 |
| 13 | 후보 삭제 | `DELETE /api/meetings/{id}/candidates/{cid}` | 204 | 올린 사람·모임장 | R-009 |
| 14 | 투표 | `PUT /api/meetings/{id}/vote` | 200 | 참가자 | R-009 |
| 15 | 투표 취소 | `DELETE /api/meetings/{id}/vote` | 204 | 참가자 | R-009 |
| 16 | 확정 | `POST /api/meetings/{id}/decision` | 200 | 모임장 | R-009 |
| 17 | 참가자 내보내기 | `DELETE /api/meetings/{id}/participants/{pid}` | 204 | 모임장 | D-013 |
| 18 | 잠금·해제 | `PUT /api/meetings/{id}/lock` | 200 | 모임장 | D-013 |
| 19 | 모임 끝내기 | `DELETE /api/meetings/{id}` | 204 | 모임장 | D-009 |
| 20 | 상태 확인 | `GET /api/health` | 200 | 없음 | 운영 |

링크 복사(R-010)와 지도 표시는 화면에서 하므로 API가 없다. 번호 5는 아래 3장에서 따로 설명한다.

## 2. 공통 모양

모임 상태 `MeetingState` (4번의 응답이자 SSE `state` 이벤트의 내용):

```json
{
  "id": "k3f9a2x7qp",
  "name": "금요일 저녁",
  "situation": "dinner",
  "status": "open",
  "locked": false,
  "me": { "id": 1, "is_host": true, "sharing": true },
  "participants": [
    { "id": 1, "name": "지수", "is_host": true, "sharing": true,
      "lat": 37.5, "lng": 127.0, "located_at": "2026-10-20T06:00:00Z", "stale": false }
  ],
  "pivot": { "name": "강남역", "lat": 37.497, "lng": 127.027, "pinned": false, "approximate": false },
  "candidates": [
    { "id": 3, "name": "식당 이름", "category": "음식점", "address": "서울 ...",
      "lat": 37.49, "lng": 127.02, "rating": 4.3, "rating_count": 120,
      "place_url": "https://place.map.kakao.com/...", "proposed_by": 2, "votes": 2 }
  ],
  "my_vote": 3,
  "decided_candidate_id": null,
  "expires_at": "2026-10-21T06:00:00Z"
}
```

- 공유를 끈 참가자의 `lat`·`lng`·`located_at`은 `null` 이다. (BR-09)
- `stale`은 2분이 넘으면 true다. (BR-06)
- `pivot`은 계산 전이면 `null` 이다.

## 3. 요청별 설명

### 1. 모임 만들기 `POST /api/meetings`

- 요청: `{ "name": "금요일 저녁", "situation": "dinner", "host_name": "지수" }`
- 성공 201: `{ "meeting_id": "k3f9a2x7qp", "participant_id": 1, "token": "…" }`
- 오류: 422 (BR-01, BR-03, BR-04)

### 2. 모임 미리보기 `GET /api/meetings/{id}/preview`

- 성공 200: `{ "name": "…", "situation": "dinner", "status": "open", "locked": false, "participant_count": 3, "full": false }`
- 위치 같은 개인 정보는 담지 않는다. 없는 모임·만료는 404.

### 3. 참여 `POST /api/meetings/{id}/participants`

- 요청: `{ "name": "민수" }`
- 성공 201: `{ "participant_id": 2, "token": "…" }`
- 오류: 422 (BR-03), 403 가득 참·잠김·확정·끝남 (BR-02, BR-29), 404

### 4. 모임 상태 `GET /api/meetings/{id}`

- 성공 200: `MeetingState`
- 오류: 401, 403 (내보내진 사람), 404

### 5. 실시간 구독 `GET /api/meetings/{id}/events?token=`

- 응답: `text/event-stream`. 연결 직후 `event: state` 로 `MeetingState` 전체를 한 번 보내고, 모임에 변화가 생길 때마다 같은 이벤트를 다시 보낸다. 20초마다 `: keep-alive` 주석 줄을 보낸다.
- 모임이 끝나거나 만료되면 `event: closed` 를 보내고 닫는다. 내보내진 참가자에게도 `closed` 를 보낸다.
- 브라우저의 `EventSource`가 끊기면 자동으로 다시 붙는다. 다시 붙으면 `state` 가 처음부터 온다.
- 이벤트에 들어가는 `me`, `my_vote` 는 구독한 사람 기준이다.

### 6. 공유 켜기·끄기 `PUT /api/meetings/{id}/me/sharing`

- 요청: `{ "enabled": true }`
- 성공 200: `{ "sharing": true }`
- 끄면(`false`) 서버의 마지막 위치를 지운다. (BR-09) 확정된 모임에서는 409.

### 7. 내 위치 보내기 `PUT /api/meetings/{id}/me/location`

- 요청: `{ "lat": 37.5, "lng": 127.0, "accuracy_m": 25 }`
- 성공 204.
- 오류: 409 (공유가 꺼져 있음, BR-05), 422 (한국 밖·오차 큼, BR-07), 429 (5초 안에 또 보냄, BR-08)

### 8. 기준 지점 고정·해제 `PUT /api/meetings/{id}/pivot`

- 요청: `{ "pinned": true }` 고정하면 지금 기준 지점을 유지한다. `{ "pinned": false }` 는 해제하고 즉시 다시 계산한다. (BR-15)
- 성공 200: `{ "pivot": … }` (`MeetingState.pivot` 모양)
- 오류: 403 모임장 아님, 409 기준 지점 없음

### 9. 기준 지점 이동시간 `GET /api/meetings/{id}/pivot/travel`

- 성공 200: `{ "items": [ { "participant_id": 1, "minutes": 32, "distance_m": 9800 } ] }`
- 기준 지점이 없으면 404. 이동시간 API 실패 시 502.

### 10. 추천 목록 `GET /api/meetings/{id}/places`

- 쿼리: `situation=lunch|dinner|second` (생략하면 모임의 상황), `categories=food,bar,cafe` (생략하면 BR-12의 기본값)
- 성공 200:

```json
{
  "pivot": { "name": "강남역", "lat": 37.497, "lng": 127.027 },
  "items": [
    { "kakao_place_id": "123", "name": "식당 이름", "category": "food",
      "address": "서울 ...", "lat": 37.49, "lng": 127.02,
      "rating": 4.3, "rating_count": 120, "open_now": true,
      "distance_m": 320, "farthest_participant_m": 5400,
      "score": 0.81, "place_url": "https://place.map.kakao.com/123" }
  ]
}
```

- 정렬은 BR-14다. 기준 지점이 없으면 409. 외부 API 실패는 502, 호출 한도는 429. (BR-30)

### 11. 장소까지 이동시간 `GET /api/meetings/{id}/places/{kakao_place_id}/travel`

- 쿼리: `lat`, `lng` (그 장소 좌표)
- 성공 200: 9번과 같은 모양. (BR-25)

### 12. 후보 올리기 `POST /api/meetings/{id}/candidates`

- 요청: `{ "kakao_place_id": "123" }`
- 서버가 최근 추천 결과(캐시)에서 그 장소 정보를 찾아 복사한다. 클라이언트가 보낸 장소 정보는 믿지 않는다.
- 성공 201: 만들어진 후보 한 개. (모양은 `MeetingState.candidates` 항목)
- 오류: 409 이미 올림·후보 10개 초과·확정됨 (BR-17), 404 추천에 없는 장소

### 13. 후보 삭제 `DELETE /api/meetings/{id}/candidates/{cid}`

- 성공 204. 오류: 403 (BR-27), 404, 409 확정된 후보

### 14. 투표 `PUT /api/meetings/{id}/vote`

- 요청: `{ "candidate_id": 3 }` 다른 후보로 보내면 표가 옮겨 간다.
- 성공 200: `{ "candidate_id": 3 }`
- 오류: 404 없는 후보, 409 확정된 모임 (BR-16)

### 15. 투표 취소 `DELETE /api/meetings/{id}/vote`

- 성공 204. 투표한 적이 없어도 204다.

### 16. 확정 `POST /api/meetings/{id}/decision`

- 요청: `{ "candidate_id": 3 }`
- 성공 200: `{ "status": "decided", "decided_candidate_id": 3 }`
- 부수 효과: 모든 참가자의 위치를 지운다. (BR-18)
- 오류: 403 모임장 아님, 404 없는 후보, 409 이미 확정

### 17. 참가자 내보내기 `DELETE /api/meetings/{id}/participants/{pid}`

- 성공 204. 오류: 403, 404, 409 모임장 자신 (BR-28)
- 내보낸 사람의 위치를 지우고 기준 지점을 다시 계산한다.

### 18. 잠금·해제 `PUT /api/meetings/{id}/lock`

- 요청: `{ "locked": true }` 성공 200: `{ "locked": true }`

### 19. 모임 끝내기 `DELETE /api/meetings/{id}`

- 성공 204. 모임과 하위 데이터를 모두 지우고, 구독 중인 화면에 `closed` 를 보낸다.

### 20. 상태 확인 `GET /api/health`

- 성공 200: `{ "status": "ok" }`

## 4. 외부 API 연동 메모

서버만 부른다. 호출은 `services/` 아래 파일에 모은다. ([ARCHITECTURE.md](ARCHITECTURE.md) 3장)

| 용도 | 서비스 | 어디서 쓰나 | 메모 |
| --- | --- | --- | --- |
| 장소 검색 | 카카오 로컬 API (카테고리 FD6·CE7, 키워드 "술집") | 10번 | 평점·영업시간 없음. 술집은 카테고리 코드가 없어 키워드로 찾는다. |
| 평점·영업 정보 | Google Places API (New) | 10번 | 이름·좌표로 매칭 (BR-26). 필드를 필요한 것만 요청해 비용을 줄인다. |
| 대중교통 이동시간 | ODsay (대안 TMAP) | 기준 지점 계산, 9번, 11번 | 한도에 주의한다. (BR-30) |
| 지도 표시 | 카카오맵 JS SDK | 화면 | 서버를 거치지 않는다. 도메인 제한. |

## 5. 아직 정하지 않은 것

- 위치를 보내는 방식과 주기: 잠정 10초 (Q-10).
- 직접 입력한 출발지: 지원하지 않는다. (Q-13)
