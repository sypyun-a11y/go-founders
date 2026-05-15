# AI 파운더스 1기 운영진 가이드

이 문서는 1기 60명(현재 50명 + 추가 10명 예정) 명단을 시트에 입력하고 점검하는 운영진용 가이드입니다.

## 1. 명단을 입력할 곳

- **스프레드시트 ID**: `1HeDIX4CYqlLJyF4NlvYBFwVFLYi-nMJRJVWtlIvJ7ms`
- **파일명**: `[B2C] AI 파운더스 1기 신청 데이터`
- **탭**: `1st_students` (이미 생성됨)
- **API base**: `https://spartaclub-ai-founders-pre-order.vercel.app/api/sheets`

## 2. `1st_students` 시트 컬럼 구성

헤더 행(A1부터)은 영문 소문자로 정확히 입력해야 합니다. 코드가 lowercase로 매칭해요.

| 컬럼 | 필수 | 형식 | 설명 |
|---|:-:|---|---|
| `name` | ✅ | 한글/영문 텍스트 | 학생 이름. 동명이인은 `phone`으로 구분 |
| `phone` | ✅ | `010-1234-5678` / `01012345678` (자유) | 로그인 매칭 키. 숫자만 추출 후 비교 |
| `team` | ✅ | 정수 (1~12) | 팀 번호. 좌석배치도/팀 리더보드 집계 |
| `pledge` | ⚠️ 선택 | 200자 이내 | 사전 리포트 "나에게 한마디". **빈값 OK** (학생이 카드에서 채움) |
| `desired_service` | ⚠️ 선택 | 200자 이내 | 사전 리포트 "만들고 싶은 서비스". 빈값 OK |

**⚠️ 주의**

- `name`은 `1st_scores` 시트의 `name`과 정확히 일치해야 점수가 매칭됨. 공백·표기 다르면 OT 기본점 5점으로 fallback.
- `phone` 비어있으면 그 학생은 로그인 자체 불가.
- `seat` 컬럼은 **불필요**. 좌석배치도는 `team` 번호로 자동 배치됨.

## 3. 입력 방법

### A. 수동 입력 (소량 / 추가 입력)

시트에 직접 행 추가. 1기 50명 매칭본은 다음 파일에 정리되어 있어요:

```
/Users/yj.baek/Desktop/ai-founders-landing/1st_students_seed.tsv
```

이 파일을 별도 시트(예: `seed_lookup`)에 통째로 paste한 뒤 `1st_students`에 phone 기준 VLOOKUP으로 `pledge`/`desired_service`를 채우는 방식을 권장.

### B. API 일괄 import

추가 10명을 한 번에 입력할 때 유용. CSV 텍스트를 POST.

```bash
curl -X POST https://spartaclub-ai-founders-pre-order.vercel.app/api/sheets \
  -H "Content-Type: application/json" \
  -d '{
    "type": "bulk_import_students",
    "admin_key": "<YOUR_ADMIN_KEY>",
    "csv": "name,phone,team,pledge,desired_service\n홍길동,01012345678,12,4주 뒤 출시,건강 다이어리\n..."
  }'
```

**전제 조건**

- Vercel 환경변수에 `ADMIN_KEY` 설정 필요 (없으면 모든 요청 401).
- CSV 첫 줄 = 헤더, 그 다음 행부터 데이터. `phone` 컬럼 필수.
- 같은 `phone` 행이 시트에 있으면 update, 없으면 append.
- CSV에 시트에 없는 컬럼이 있으면 시트 헤더 끝에 자동 추가됨.
- 따옴표로 감싼 필드(`"..."`) 지원, 이스케이프는 `""`로.

**응답 예시**

```json
{
  "ok": true,
  "added": 10,
  "updated": 0,
  "total": 10,
  "errors": []
}
```

## 4. 입력 완료도 확인

```
GET https://spartaclub-ai-founders-pre-order.vercel.app/api/sheets?action=student_stats
```

응답 예시:

```json
{
  "ok": true,
  "total": 60,
  "by_team": { "1": 5, "2": 5, "3": 5, "4": 5, "5": 5, "6": 5, "7": 5, "8": 4, "9": 4, "10": 4, "11": 3 },
  "missing": { "phone": 0, "pledge": 4, "team": 0, "desired_service": 4 }
}
```

**해석 가이드**

- `missing.phone > 0` → 그 학생은 **로그인 불가**. 즉시 채워야 함.
- `missing.team > 0` → 좌석배치도/팀 리더보드에서 빠짐.
- `missing.pledge > 0` → 학생이 카드에서 직접 채울 수 있으니 OK. 단, 사전 리포트 미제출자가 누구인지 운영 시트 대조해서 후속 안내 권장.
- `missing.desired_service > 0` → 동일. 학생이 카드에서 채움.

## 5. 학생이 직접 수정할 수 있는 항목

다음 두 컬럼은 학생이 본인 카드에서 편집 가능합니다. 학생이 저장하면 시트 값이 덮어써져요.

| 컬럼 | 수정 핸들러 |
|---|---|
| `pledge` | `POST {type: 'update_pledge', name, phone, pledge}` |
| `desired_service` | `POST {type: 'update_mission', name, phone, mission}` |

→ 운영진이 사전 리포트 응답을 미리 채워두면 학생이 카드에서 보고 그대로 두거나 수정. **운영진이 원본을 보존하고 싶다면** `pledge_original` 같은 컬럼을 별도로 추가하세요. 코드가 그 컬럼은 읽지도 쓰지도 않으니 영원히 보존됩니다.

## 6. `1st_scores` 시트 (점수 관리)

별도 탭. 운영진이 매주 수동 업데이트.

| 컬럼 | 형식 | 비고 |
|---|---|---|
| `name` | 텍스트 | `1st_students.name`과 정확히 일치해야 매칭 |
| `score` | 숫자 | 리더보드 1차 정렬 키 |
| `planet_count` | 정수 | 행성 도감 개수. 동점자 2차 정렬 키 |
| `notes` | 텍스트 | 운영진 메모 (API 응답엔 미노출) |

**비워둬도 OK**. 행이 없는 학생은 자동으로 `score=5`(OT 참석), `planet_count=0`. 첫 주차 끝나고 채우기 시작하면 됩니다.

## 7. 동명이인 검증

`student_login` / `update_mission` / `update_pledge` 모두 **이름 + 전화번호 둘 다 일치**해야만 통과. 동명이인이 있어도 phone으로 자동 구분되니 걱정 없어요.

## 8. 트러블슈팅

| 증상 | 원인 후보 |
|---|---|
| 학생이 "로그인 안 됨" | 시트의 `phone`이 비어있거나, 입력한 전화번호 자릿수가 다름 |
| 카드에 점수가 5점으로만 나옴 | `1st_scores`에 해당 학생 `name`이 없거나 표기 불일치 |
| 좌석배치도에 자기 자리가 안 보임 | `team` 컬럼이 비어있거나 비숫자 |
| 리더보드 "내 팀"이 강조 안 됨 | `_student.team` 누락 → `team` 값 확인 |
| `bulk_import_students` 401 | Vercel `ADMIN_KEY` 환경변수 미설정 또는 요청 body의 `admin_key` 불일치 |
