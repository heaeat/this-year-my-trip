# 올해의 여행 ✈️

> 돈을 그만 쓰자

여행 예약 및 정산을 관리하는 개인 웹앱입니다.

**→ [this-year-my-trip.web.app](https://this-year-my-trip.web.app)**

---

## 주요 기능

### 💰 가계부
- 항공, 숙소, 렌터카, 식비, 교통 등 카테고리별 예약/지출 관리
- 상태 관리: 미예약 → 정산필요 → 정산완료
- 다중 통화 지원 (KRW, JPY, CNY, USD, VND, IDR, THB, EUR) + 실시간 환율 자동 변환
- 카테고리별 요약 카드 → 클릭 시 상세 목록

### ✈️🏨🚗 항공 / 숙소 / 렌터카
- 예약 정보 연동 (항공편 정보, 체크인/아웃, 픽업 일정 등)
- 날짜 기준 자동 정렬

### 🧾 정산
- 결제자 / 해당 인원 기반 자동 정산 계산
- 최소 송금 횟수로 정산 금액 산출

### 📍 가고싶은 곳
- 지도(OpenStreetMap)에 방문하고 싶은 장소 핀으로 표시
- Google Places API 기반 장소 검색
- 카테고리별 아이콘 (맛집, 관광지, 카페, 쇼핑 등)
- 방문 완료 체크 기능
- 바이두 지도 URL/좌표 자동 변환 지원 (중국 여행)

### 📝 메모
- 여행별 자유 메모 (마크다운 지원)

### 👥 멤버 & 초대
- 초대 코드로 여행 공유
- 멤버별 정산 자동 계산

---

## 기술 스택

- **Frontend**: Vanilla JS, 단일 HTML 파일 (`index.html`)
- **Backend**: Firebase Firestore (데이터), Firebase Auth (Google 로그인)
- **Hosting**: Firebase Hosting
- **지도**: Leaflet.js + OpenStreetMap
- **장소 검색**: Google Places API
- **환율**: open.er-api.com

---

## 배포

```bash
firebase deploy --only hosting
```

`index.html` 하나에 CSS, JS가 모두 인라인으로 포함되어 있습니다.

---

## 주의사항

- Firebase `authDomain`은 반드시 `firebaseapp.com` 유지 (`web.app`으로 변경 시 OAuth 오류)
- Google Places API 키는 HTTP 리퍼러를 `this-year-my-trip.web.app/*`로 제한하여 사용
