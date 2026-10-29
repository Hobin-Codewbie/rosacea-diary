# 주사 피부 다이어리 (rosacea-diary)

개인 사용자가 아이폰에서 쓰는 주사(로사시아) 피부 기록 웹앱. 미국 주사학회(NRS) Rosacea Diary를 기준으로, 매일 유발요인을 기록하고 갑자기 올라올 때를 따로 기록해 개인 유발요인을 찾는 게 목적.

- 배포: GitHub Pages, `main` 브랜치 루트 → https://hobin-codewbie.github.io/rosacea-diary/
- 사용: 아이폰 Safari → 홈 화면에 추가 (PWA, standalone)
- 서버·계정·빌드 없음. 정적 파일만. 푸시하면 1~2분 뒤 반영.

## 사용자와 대화 방식

- 요청자는 개발 담당, 실제 사용자는 가족. 한국어, 짧고 편하게.
- 바꾼 뒤에는 "뭘 바꿨는지 + 사용자 폰에서 반영 방법(앱 완전히 닫았다 열기)"만 짧게.

## 파일

| 파일 | 역할 |
| --- | --- |
| `index.html` | 앱 전체 (HTML + CSS + JS 한 파일). 프레임워크·라이브러리 없음 |
| `sw.js` | 서비스워커. 앱 파일 network-first(`cache:"no-cache"`), 오프라인이면 캐시. 날씨 API는 캐시 안 함 |
| `manifest.webmanifest`, `icon-*.png` | 홈 화면 앱 설정·아이콘 |

## 사용자 환경 (설계 전제)

- **2교대 근무**: 주간 07–19, 야간 19–07, 휴무. 앱 안 "근무 시간 바꾸기"로 수정 가능(localStorage `rosacea-shift-times`).
- **클린룸 근무, 방진복·후드·마스크 착용** → 착용 시간, 클린룸 건조함, 근무 중 땀·열감을 기록.
- 지역 **구미** (날씨: Open-Meteo, lat 36.1195 / lon 128.3446, `ensureWeather()`).
- 저녁에 몰아서 쓰기보다 **끼니 먹고 바로 짧게 입력 → 저녁에 피부 점수** 흐름.

## 화면 구조

탭 4개: **매일 다이어리 / 올라왔을 때 / 분석 / 기록** (+ 제목 옆 버전 버튼 → 업데이트 내역 화면)

매일 다이어리 섹션 순서:
0. 근무·수면 (주간/야간/휴무, 수면 시간, 방진복 착용 시간)
1. 오늘 피부 상태 (붉어짐 0–3, 가려움 0–3, 증상, 사진)
2. 노출된 날씨 (Sunny·Windy·Cloudy·Humid·Hot·Cold·Mild·Dry + 노출 시간, 자외선 차단제). 구미 날씨로 추천 항목 미리 체크
3. 식사·음료 — **끼니 탭 4개(한 번에 하나만 펼침)**. 근무별 이름: 주간=출근 전/근무 중 식사/퇴근 후/간식, 야간=출근 전/근무 중 야식/퇴근 후/근무 중 간식, 휴무=아침/점심/저녁/간식·야식. 끼니 안 순서: 시각 → 주의 음식·음료(**접혀 있음**) → 먹은 메뉴(**입력한 것만 표시**, 지난 메뉴는 datalist 자동완성)
4. 활동·감정·온도 / 5. 스킨케어·화장품(새 화장품: 브랜드·제품명·종류·반응) / 6. 치료·몸 상태 / 7. 메모

피부 점수 없이 **중간 저장** 가능.

## 사용자가 확정한 UX 원칙 (되돌리지 말 것)

- 끼니는 탭으로 하나씩. 한 화면에 끼니 여러 개 펼치지 않기.
- 주의 음식·음료 목록은 기본 접힘. 펼치기 전엔 선택한 항목 요약만.
- 먹은 메뉴 영역에 지난 메뉴 칩을 늘어놓지 않기.
- 모든 입력칸 글자 크기 16px 이상 (iOS 자동 확대 방지). `input,select,textarea{font-size:16px!important}` 유지.
- 버전과 바뀐 점을 맨 위에 표시.

## 핵심 로직 (고칠 때 주의)

- **기록일(record day)**: `recordDayOf(ts)` — 전날이 야간 근무면 다음 날 정오 전까지 전날 기록. 그 외엔 새벽 5시 전이면 전날. 앱을 열면 이 기록일로 열림. 올라왔을 때 기록의 `day`도 이 함수로 정함.
- **끼니 실제 시각**: `mealTs(day, shift, "HH:MM")` — 야간 기록일은 12시 전 시각을 다음 날로, 그 외는 5시 전을 다음 날로.
- **끼니 기본 시각**: `mealPlan(shift)`가 근무 시간에서 계산. 지금이 그 끼니 ±3시간이면 현재 시각.
- **직전 끼니 연결**: `lastMealBefore(ts)` → 올라왔을 때 탭에서 "직전 끼니 내용 같이 기록", `mealRef` 저장.
- **분석**: 의심 요인 = 요인 있던 날(또는 다음 날 중 높은 값) 평균 붉어짐 − 없던 날, 3회 이상만, 차이 0.8↑ "의심". 근무별 비교(주간/야간/야간 다음 날/휴무), 식사 후 올라오기까지(1h/1–3h/3–6h/6h+), 시간대.
- 분석은 `redness != null`인 날만 사용 (중간 저장된 날 제외).

## 데이터 (사용자 폰 안에만 있음 — 지우거나 형식 깨지 말 것)

IndexedDB `rosacea-diary` v1, store 4개, 키-값:

- `days` (키 `YYYY-MM-DD`): `{date, shift, sleep, suit, redness, itch, symptoms[], photos[photoId], weather{key:duration|null}, sunscreen, meals{breakfast|lunch|dinner|snack:{state:"ate"|"skip"|null, time:"HH:MM", items[], risk[], drinks[]}}, drinks[], foods[], foodsOther[] (구버전 호환), acts[], skin[], cosmetics[{brand,name,type,reaction}], body[], meds[], memo, triggers[] (분석용 평탄화), score(=redness), savedAt}`
- `flares` (키 `f<timestamp>`): `{id, ts, day, redness, itch, severity, areas[], triggers[], mealRef{meal,label,time,gapMin}|null, memo, savedAt}`
- `weather` (키 날짜): `{date, region, summary, tmax, tmin, humidity, uv, wind, suggested[], source, fetchedAt, final}`
- `photos` (키 `p…`): JPEG Blob (긴 변 1600px로 줄여 저장)

필드를 바꿀 땐 **옛 기록도 읽히게** 기본값 처리할 것 (`loadDay`의 병합, `|| []` 패턴). 백업 JSON(`app:"rosacea-diary", version:1`)과 CSV 내보내기도 같이 맞출 것.

## 바꿀 때마다 반드시

1. `index.html`의 `CHANGELOG` 맨 위에 새 항목 추가 (`["1.6","MM/DD",["바뀐 점", ...]]`). 이게 앱 상단 버전·"바뀐 점" 배너에 그대로 뜸. 바뀐 점은 사용자가 읽는 말로 짧게.
2. `sw.js`의 `CACHE` 이름 숫자 올리기 (`rosacea-diary-v8` …).
3. 폰 크기(390px)로 확인. Playwright가 있으면 `python3 -m http.server`로 띄워서 클릭 흐름 + 콘솔 에러 확인. 날씨 API는 route로 가짜 응답, 폰트는 abort.
4. 커밋 메시지는 한국어로 `vX.Y: 요약`. `main`에 푸시.

## 하지 않을 것

- 기록·사진을 외부 서버로 보내는 기능 (개인 건강 정보). 외부 통신은 Open-Meteo 날씨와 Google Fonts만.
- 프레임워크·빌드 도구 도입 (한 파일 유지가 배포·수정이 쉬움).
- `alert/confirm` 대신 화면 안 확인 UI 사용 (삭제 확인 등 기존 패턴 따름).

## 나중에 할 수 있는 것 (요청 시)

- 아침 피부 점수 따로 (전날 → 다음 날 아침 지연 반응 보기)
- 근무별 알림: 아이폰 단축어 자동화로 앱 열기 안내
- 구내식당 메뉴 묶음
- 기능이 안정되면 Xcode로 Swift 네이티브 앱 전환 (알림·위젯), 같은 데이터 구조 유지
