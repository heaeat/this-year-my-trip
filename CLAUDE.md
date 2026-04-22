# 올해의 여행 — 프로젝트 참고 문서

## 개요
여행 예약/정산 관리 웹앱. **단일 HTML 파일** (`index.html`) 하나로 구성된 Vanilla JS + Firebase 앱.
Firebase Hosting으로 배포 중.

- **URL**: `this-year-my-trip.web.app` (또는 `this-year-my-trip.firebaseapp.com`)
- **파일**: `c:\works\this-year-my-trip\index.html` (CSS + JS 모두 인라인)
- **Firebase 설정**: `firebase.json`

---

## 용어 정의

대화 중 아래 명칭을 기준으로 사용.

| 명칭 | 의미 |
|------|------|
| **전체보기** | 사이드바에서 특정 여행을 선택하지 않은 상태 (`S.selectedTrip === null`). 가계부 탭에서는 카테고리 요약 카드(`rCatSummary`) 표시. |
| **특정여행** | 사이드바에서 여행을 하나 선택한 상태 (`S.selectedTrip`에 ID 존재). 탭바가 표시되며 가계부에 전체 목록 표시. |
| **탭** | 특정여행 선택 시 상단에 표시되는 `가계부 / 항공편 / 숙소 / 렌터카 / 가고싶은 곳 / 정산 / 메모` 메뉴. `S.activeTab`으로 관리. |
| **상태 필터** | 가계부 탭의 `전체 / 정산완료 / 정산필요 / 미예약` 버튼. `S.filterStatus`로 관리. |
| **예약 카테고리** | 예약에 붙는 카테고리: 항공/숙소/렌터카/교통/식비/관광/다이빙/usim. 상수 `CC`, `CI`로 정의. |
| **예약 카테고리 필터** | 특정여행 가계부 탭의 카테고리 드롭다운(`전체 카테고리 / 항공 / 숙소 / ...`). `S.filterCategory`로 관리. |
| **카테고리 요약 카드** | 전체보기 가계부에서 예약 카테고리별로 건수·금액·상태를 보여주는 카드 그리드. `rCatSummary(res)` 렌더링. |
| **장소 카테고리** | 가고싶은 곳 탭의 장소에 붙는 카테고리: 맛집/관광지/쇼핑/카페/디저트/물놀이/공항/숙소/기타. 상수 `PLACE_CAT`, `PLACE_COLOR`로 정의. 예약 카테고리와 별개. |

---

## Firebase 구성

```js
firebase.initializeApp({
  apiKey: "AIzaSyCINhPlJyeaSJZBVzClIftXcsZR5ZkyNPM",
  authDomain: "this-year-my-trip.firebaseapp.com",  // ← 반드시 firebaseapp.com (web.app으로 바꾸면 OAuth 오류)
  projectId: "this-year-my-trip",
  ...
});
```

**Firestore 컬렉션:**
- `trips` — 여행 문서 (최상위)
- `reservations` — 예약 문서 (최상위, `tripId` 필드로 여행과 연결)
- `places` — 가고싶은 곳 문서 (최상위, `tripId` 필드로 여행과 연결)

**함수:**
```js
tC() → db.collection('trips')
rC() → db.collection('reservations')
pC() → db.collection('places')
```

`places` 쿼리는 별도 try/catch로 감싸 실패해도 reservations 로드에 영향 없음.

---

## 앱 상태 (S)

```js
let S = {
  screen: 'loading' | 'login' | 'setup' | 'app' | 'migrating',
  user: null,                  // Firebase User 객체
  trips: [],                   // 여행 배열
  reservations: [],            // 예약 배열
  places: [],                  // 가고싶은 곳 배열
  selectedTrip: null,          // 선택된 여행 ID (null이면 전체보기)
  filterStatus: 'all',         // 가계부 탭 상태 필터 (가계부 탭 전용)
  filterCategory: 'all',       // 가계부 탭 카테고리 필터 (가계부 탭 전용)
  modal: null,                 // 열려있는 모달 정보 { type, ... }
  activeTab: 'budget',         // 현재 탭: budget|flights|hotels|cars|notes|settle|wishlist
  selectedYear: <현재연도>,
  notesEditing: false,
  settleShowAll: false,        // 정산 탭: 정산완료 항목도 포함 여부
  tripDropdown: false,         // 모바일 헤더 여행 선택 드롭다운
  statusPicker: null,          // 상태 변경 팝오버 { id, cur, top, left }
};
```

---

## 예약(Reservation) 데이터 구조

```js
{
  id: string,
  tripId: string,
  name: string,
  category: '항공'|'숙소'|'렌터카'|'교통'|'식비'|'관광'|'다이빙'|'usim',
  status: '미예약'|'정산필요'|'정산완료',
  total: number,
  currency: 'KRW'|'JPY'|'CNY'|'USD'|'VND'|'IDR'|'THB'|'EUR',
  customRate: number|null,     // 직접 입력 환율
  payer: string,               // 결제자 이름
  people: string[],            // 해당인원 이름 배열
  date: string,
  vendor: string,
  memo: string,
  flightDetails: {},           // 항공편 전용
  hotelDetails: {},            // 숙소 전용
  carDetails: {},              // 렌터카 전용
  createdBy: string,
}
```

**새 예약 기본 카테고리: `식비`**

---

## 가고싶은 곳(Place) 데이터 구조

```js
{
  id: string,
  tripId: string,
  name: string,
  lat: number,       // WGS84 위도 (항상 WGS84로 저장)
  lng: number,       // WGS84 경도
  address: string,
  category: '맛집'|'관광지'|'쇼핑'|'카페'|'디저트'|'물놀이'|'공항'|'숙소'|'기타',
  note: string,
  visited: boolean,
  createdBy: string,
}
```

```js
const PLACE_CAT = {맛집:'🍚', 관광지:'🎡', 쇼핑:'🛍️', 카페:'☕', 디저트:'🍰', 물놀이:'🏖️', 공항:'✈️', 숙소:'🏨', 기타:'📍'};
const PLACE_COLOR = {맛집:'#f59e0b', 관광지:'#8b5cf6', 쇼핑:'#ec4899', 카페:'#92400e', 디저트:'#f472b6', 물놀이:'#0ea5e9', 공항:'#64748b', 숙소:'#6366f1', 기타:'#9ca3af'};
```

---

## 주요 함수

| 함수 | 역할 |
|------|------|
| `render()` | 전체 앱 재렌더링 (try-catch 감싸짐) |
| `postRender()` | render() 후 호출 — wishlist 탭이면 지도 초기화 |
| `rApp()` | 앱 메인 HTML 생성 |
| `fil()` | 상태/카테고리 필터 적용 → 가계부 탭 전용으로만 사용 |
| `myName()` | 현재 유저 닉네임 (localStorage `nick_{uid}` → displayName) |
| `getKRW(r)` | 예약의 원화 환산 금액 |
| `addR(d)` / `updR(id,d)` / `delR(id)` | Firestore CRUD + 로컬 S 동기 업데이트 |
| `addTrip(d)` / `delTrip(id)` | 여행 CRUD |
| `calcSettlement(res)` | 정산 계산: balance map → greedy 최소 송금 |
| `showStatusPicker(e,id,cur,el)` | 상태 변경 팝오버 표시 |
| `pickStatus(status)` | 팝오버에서 상태 선택 후 저장 |
| `initWishMap(tripId)` | Leaflet 지도 초기화/재사용 |
| `parsePlaceCoord(val)` | 좌표/URL 파싱 → WGS84 반환 |
| `bd09mcToWgs84(mx,my)` | BD09MC(바이두 메르카토르) → WGS84 변환 |
| `bd09ToWgs84(bdLat,bdLng)` | BD09(바이두 위경도) → WGS84 변환 |
| `rCatSummary(res)` | 카테고리별 요약 카드 렌더링 (전체보기 가계부 전용) |

---

## 탭 구조와 필터 분리

```
rApp()
├── res = fil()              ← 필터 적용, 가계부 탭 전용
├── tripRes = reservations   ← 필터 미적용, 나머지 탭용
└── rTabContent(sel, res, tripRes)
    ├── 'budget'   → rTable(res, sel)      ← 필터 적용된 res 사용
    ├── 'flights'  → rTabFlights(tripRes)  ← 필터 무시
    ├── 'hotels'   → rTabHotels(tripRes)
    ├── 'cars'     → rTabCars(tripRes)
    ├── 'notes'    → rTabNotes(sel)
    ├── 'settle'   → rTabSettle(tripRes)
    └── 'wishlist' → rTabWishlist(sel)
```

**전체보기(`sel === null`)일 때**: 가계부 → `rCatSummary(res)` (카테고리 요약 카드)
**특정 여행 선택 시**: 가계부 → `rTable(res, sel)` (전체 목록)

**이 분리는 중요**: 가계부 탭의 상태/카테고리 필터가 다른 탭에 영향을 주지 않기 위함.

---

## 가계부 탭 — 카테고리 요약 카드 (`rCatSummary`)

전체보기에서만 표시. 카테고리별로:
- 건수, 총 KRW 금액
- 정산완료/정산필요/미예약 뱃지
- 클릭 시 `filterCategory` 설정 → 해당 카테고리 목록으로 이동
- 상태 필터(정산완료 등) 적용된 `res`를 받으므로 상태 필터와 함께 동작

---

## 가고싶은 곳 탭 (wishlist)

- Leaflet.js + OpenStreetMap 타일 (API 키 불필요)
- **장소 검색**: Google Places Autocomplete API (`_gAutoSvc`) 우선, 결과 없으면 Nominatim 폴백
  - city hint(`trip.wishlistCity`) 있으면 검색어에 붙여 우선 검색
  - Google API 미로드 시 Nominatim으로 fallback
- 장소 카드: 구글맵 링크(좌표 기반), 바이두맵 링크는 중국 여행에만 표시
- **중국 여행 판단**: `trip.name`에 한자(`[\u4e00-\u9fff]`) 포함 여부로 판단 (TODO: 개선 필요)
- 지도 인스턴스 `window._wishMap` — render()를 거쳐도 재사용 (`window._wishMapState`로 상태 보존)
- `postRender()`: wishlist 탭이고 modal 없을 때만 `setTimeout(() => initWishMap(), 0)` 호출

**지도 마커 디자인:**
- 배경 없이 이모지만 표시 (26px, drop-shadow)
- 이름 라벨: 마커 아래 검정색 텍스트 (흰색 text-shadow로 가독성 확보)
- 방문완료: opacity 0.4로 흐리게
- `PLACE_COLOR`로 카테고리별 색상 정의 (현재 마커엔 미사용, 향후 활용 가능)

**좌표 변환 흐름:**
- 바이두맵 URL 붙여넣기(`@x,y,z` 형식) → BD09MC로 자동 인식 → WGS84 변환 → 필드 업데이트
- 좌표 숫자 값 > 1000 → BD09MC 자동 감지 → WGS84 변환
- 바이두 위경도 직접 입력 → "바이두 좌표" 체크박스 체크 → BD09→WGS84 변환
- 일반 위경도 → WGS84 그대로 저장
- `onpaste` + setTimeout으로 모바일 붙여넣기 대응

**Google Places API:**
- 키: `AIzaSyCebO67d42-wfI7KqMwa7SYOQyAJ4x4DUE`
- 리퍼러 제한: `this-year-my-trip.web.app/*`
- `window.initGooglePlaces` → `_gAutoSvc`, `_gPlacesSvc` 초기화
- hover 미리보기 / 클릭 선택 시 `getDetails`로 lat/lng 지연 로드

---

## 필터 바 (가계부 탭)

두 줄 레이아웃:
- 1행: 전체/미예약/정산필요/정산완료
- 2행: 전체/✈️항공/🏨숙소/... (CC에 정의된 모든 카테고리 하드코딩, 데이터 유무 무관)

`filter-sep` div (`width:100%`)로 줄바꿈 강제.

---

## 정산 알고리즘 (`calcSettlement`)

1. 각 예약에서: 결제자(payer)에게 전체 금액 크레딧, 해당인원(people) 각자에게 1/N 데빗
2. 잔액이 음수(빚짐) 배열과 양수(받을 돈) 배열로 분리
3. Greedy: 가장 많이 빚진 사람 → 가장 많이 받을 사람 순으로 매칭

정산 탭 기본값: `정산필요` 상태만 계산, 체크박스로 `정산완료`도 포함 가능.

---

## 카테고리 상수

```js
// 예약 카테고리
const CC = {"항공":"flight","숙소":"hotel","렌터카":"car","교통":"transport","식비":"dining","관광":"sightseeing","다이빙":"diving","usim":"usim"};
const CI = {"항공":"✈️","숙소":"🏨","렌터카":"🚗","교통":"🚌","식비":"🍚","관광":"🎡","다이빙":"🤿","usim":"📱"};

// 가고싶은 곳 카테고리 (예약과 별개)
const PLACE_CAT = {맛집:'🍚', 관광지:'🎡', 쇼핑:'🛍️', 카페:'☕', 디저트:'🍰', 물놀이:'🏖️', 공항:'✈️', 숙소:'🏨', 기타:'📍'};
```

식비/맛집 아이콘: 🍜(면) → 🍚(흰쌀밥) 로 변경됨.

---

## 로그인 화면 (`rLogin`)

- 아이콘: 🧳, 타이틀: "올해의 여행", 멘트: "친구들과의 여행 예약을 한눈에 관리하세요"
- 앱 이름/멘트 변경 논의 중 (TODO.md 참고)
- 인앱 브라우저 감지 시 외부 브라우저 유도 UI 표시

---

## 로그인 (Firebase Auth)

- 항상 `signInWithPopup` 우선
- 팝업이 완전히 차단된 경우만 `signInWithRedirect` 폴백
- 인앱 브라우저(카카오톡, 인스타 등) 감지 시 외부 브라우저로 유도

**중요한 설정:**
- `firebase.json`의 `Cross-Origin-Opener-Policy: same-origin-allow-popups` 헤더 필수 (없으면 팝업 통신 차단됨)
- `authDomain`은 반드시 `firebaseapp.com` 유지 (`web.app`으로 변경 시 "액세스 차단됨" OAuth 오류 발생)
- `signInWithRedirect`만 사용하면 iOS Safari ITP로 인해 redirect loop 발생

---

## 모바일 대응

- 768px 이하에서 사이드바 숨기고 헤더에 여행 선택 드롭다운 버튼(`trip-sel-btn`) 표시
- FAB 버튼 (하단 우측 고정, `position:fixed`) — `overflow-x:hidden`을 `html/body`에 적용하면 iOS Safari에서 fixed 깨짐 → `.main`에만 적용
- 상태 배지 탭 시 즉시 변경 대신 팝오버 메뉴 표시 (실수 방지)
- wishlist 탭: 모바일은 지도(250px) 위, 목록 아래 세로 배열
  - **CSS 순서 주의**: 모바일 미디어쿼리(`@media max-width:768px`) 보다 데스크탑 스타일이 나중에 선언되면 덮어씀
  - 해결: 데스크탑 wishlist 스타일을 `@media(min-width:769px)`로 감싸야 함

---

## 환율

`open.er-api.com/v6/latest/KRW` API로 실시간 로드. 실패 시 하드코딩 폴백값 사용.
KRW 기준 (타 통화 → KRW 변환: `amount / rate`).

---

## 배포

```bash
firebase deploy --only hosting
```
