- 상태: 초안

# 데이터 모델

- 출처: [PRD.md](PRD.md) 4장, [DECISIONS.md](DECISIONS.md) D-006·D-007·D-012·D-013, [ARCHITECTURE.md](ARCHITECTURE.md) 4장
- 저장소는 SQLite 파일 하나(`backend/data/taskflow.db`), 테이블 하나(`cards`)다. (D-009)
- 카드 = Task. 아래 필드 여섯 개가 모든 문서에서 같은 뜻으로 쓰인다.

## 1. 필드

| 필드 | 뜻 | 누가 넣는지 | 필수 여부 | 변경 규칙 |
| --- | --- | --- | --- | --- |
| `id` | 카드를 가리키는 번호 (정수) | 저장소가 자동으로 매긴다 (D-007) | 필수 (자동) | 바꿀 수 없다 |
| `title` | 할 일의 이름 | 사용자 | 필수. 앞뒤 공백을 뺀 뒤 비면 안 된다 (V-1). 길이 제한 「결정 필요 (Q-08)」 | 수정으로 바꿀 수 있다 (R-004) |
| `description` | 할 일의 자세한 설명 | 사용자 | 선택. 없으면 빈 문자열 (잠정 D-012, Q-01). 길이 제한 「결정 필요 (Q-08)」 | 수정으로 바꿀 수 있다 (R-004) |
| `status` | 카드의 현재 진행 상태 | 처음에는 서버가 `todo` 로 넣고 (잠정 D-012), 그다음은 사용자가 고른다 (F-11) | 필수. 세 값 중 정확히 하나 (V-2, V-3) | 상태 변경 요청으로만 바뀐다. 수정 요청으로는 바뀌지 않는다 (D-005) |
| `created_at` | 카드를 만든 시각 | 서버 | 필수 (자동) | 바꿀 수 없다. 같은 상태 안 순서에 쓴다 (D-013) |
| `updated_at` | 카드를 마지막으로 바꾼 시각 | 서버 | 필수 (자동) | 수정·상태 변경 때 서버가 갱신한다 |

시각 형식은 ISO 8601 UTC 문자열이다 (예: `2026-10-19T10:00:00Z`).

## 2. status 짝 맞추기

| 화면 표시 | 저장 값 | 뜻 |
| --- | --- | --- |
| 할 일 | `todo` | 아직 시작하지 않았다. 새 카드의 처음 상태 (잠정) |
| 진행 중 | `in_progress` | 하고 있다 |
| 완료 | `done` | 끝났다 |

- 저장·API 에서는 오른쪽 코드만 쓰고, 화면에서만 왼쪽 이름을 보여 준다. 짝은 `domain/status.js` 한 곳에 둔다. (D-006)
- 상태 세 가지는 고정이라고 가정한다. 늘리거나 이름을 바꿀 수 있는지는 「결정 필요 (Q-12)」.
- 상태 코드 세 개는 저장소 CHECK 제약, API 검증, 화면 목록에서 같아야 한다.

## 3. 저장소 정의

```sql
CREATE TABLE IF NOT EXISTS cards (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    title       TEXT    NOT NULL CHECK (length(trim(title)) > 0),
    description TEXT    NOT NULL DEFAULT '',
    status      TEXT    NOT NULL CHECK (status IN ('todo', 'in_progress', 'done')),
    created_at  TEXT    NOT NULL,
    updated_at  TEXT    NOT NULL
);
```

## 4. 응답 예

```json
{
  "id": 1,
  "title": "보고서 초안 쓰기",
  "description": "",
  "status": "todo",
  "created_at": "2026-10-19T10:00:00Z",
  "updated_at": "2026-10-19T10:00:00Z"
}
```

## 5. 이 모델에 없는 것

마감일(Later, F-13) · 순서 칸 (Q-06) · 지운 표시 (Q-07) · 사용자 id (로그인 없음, F-08). 이 필드들은 PRD 에서 정해지기 전에는 추가하지 않는다.
