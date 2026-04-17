# 워크샵 스코어보드 — 프로젝트 컨텍스트

> Claude Code가 이 문서를 읽으면 프로젝트 전체를 파악할 수 있습니다.
> 작업 요청 전에 이 파일을 먼저 읽어주세요.

---

## 프로젝트 개요

모바일 전용 워크샵 실시간 점수 공유 웹앱.  
참가자들이 자신의 팀/개인 점수와 랭킹을 실시간으로 확인하고,  
진행자(MC)가 별도 관리자 페이지에서 점수를 조정하는 구조.

**배포 방식:** 정적 HTML → Cloudflare Pages  
**백엔드:** Firebase Realtime Database (실시간 동기화)  
**UI 특성:** 모바일 전용, 밝은 라이트 테마, 카드형 레이아웃

---

## 파일 구성

```
workshop-app/
├── index.html        ← 참가자 페이지
├── admin.html        ← 관리자(MC) 페이지
└── sample-data.json  ← Firebase 초기 데이터 구조
```

파일 3개가 전부입니다. 빌드 도구, 패키지 매니저, 프레임워크 없음.  
순수 HTML + CSS + Vanilla JS + Firebase SDK (CDN).

---

## Firebase 연결 정보

```js
const firebaseConfig = {
  apiKey:            "AIzaSyC7S-HzpRjOPqtbyfJAYO1y3ir0msbV4SA",
  authDomain:        "oromworkshop-56b21.firebaseapp.com",
  databaseURL:       "https://oromworkshop-56b21-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId:         "oromworkshop-56b21",
  storageBucket:     "oromworkshop-56b21.firebasestorage.app",
  messagingSenderId: "769454280422",
  appId:             "1:769454280422:web:7ec2295e6f58e157d67527"
};
```

Firebase SDK 버전: `10.12.0` (CDN 방식, ES module import)

```js
// 사용 패턴
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getDatabase, ref, onValue, update, push } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js";
```

---

## Firebase 데이터 구조

```json
{
  "participants": {
    "user_001": {
      "name": "김주희",
      "code": "3507",
      "team": "A팀",
      "score": 0
    }
  },
  "teams": {
    "A팀": {
      "teamName": "A팀",
      "score": 0
    }
  },
  "notices": {
    "notice_xxx": {
      "text": "공지 내용",
      "timestamp": 1700000000000
    }
  },
  "scoreLogs": {
    "log_xxx": {
      "target": "팀: A팀",
      "value": 10,
      "timestamp": 1700000000000
    }
  }
}
```

**현재 등록된 테스트 참가자:**

| uid | 이름 | code | 팀 |
|-----|------|------|----|
| user_001 | 김주희 | 3507 | A팀 |
| user_002 | 조준수 | 6282 | B팀 |
| user_003 | 홍길동 | 1234 | A팀 |
| user_004 | 이지연 | 5678 | B팀 |

---

## index.html — 참가자 페이지

### 화면 구성 (단일 페이지, 상태로 전환)

| 상태 | ID | 설명 |
|------|----|------|
| 로그인 화면 | `#loginScreen` | 이름 + 전화번호 뒤 4자리 입력 |
| 메인 앱 | `#appScreen` | 로그인 성공 후 표시 |

### 로그인 로직
- Firebase `participants`에서 `name` + `code` 일치 여부 확인
- 일치하면 `localStorage`에 저장 → 새로고침 시 자동 로그인
- **신규 생성 없음**, 등록된 참가자만 입장 가능

### 메인 앱 구성 (위→아래)
1. **sticky 헤더** — 팀명 뱃지 + 이름 + 팀점수/개인점수 pill
2. **공지사항 카드** — 최신 공지 1개, 실시간 업데이트
3. **팀 순위 카드** — 전체 팀 점수 내림차순
4. **개인 TOP 5 카드** — 상위 5명만 표시 (전체 순위 없음)
5. **로그아웃 버튼**

### 실시간 리스너 (Firebase `onValue`)
- `participants` → 개인 점수, 랭킹 갱신
- `teams` → 팀 점수 갱신
- `notices` → 공지 갱신

### 팀 색상 시스템
팀 순서(Object.keys 인덱스)로 색상 자동 배정:

```js
const TEAM_COLORS = ['A','B','C','D','E'];
// CSS 변수: --team-a ~ --team-e
// 클래스: .tc-A, .dot-A, .score-A (각각 배지/점/점수 색상)
```

---

## admin.html — 관리자 페이지

### 접근
- 비밀번호 입력: `orom0430`
- `sessionStorage`에 인증 상태 저장 (탭 닫으면 로그아웃)

### 탭 구성

| 탭 | ID | 기능 |
|----|----|------|
| 🏅 팀 점수 | `tab-team` | 팀 선택 후 +1/+5/+10/+50/-1/-5/-10/-50 또는 직접 입력 |
| 👤 개인 점수 | `tab-personal` | 참가자 선택 후 동일한 버튼 구성 |
| 📢 공지 | `tab-notice` | textarea 작성 후 전송 |
| 📋 로그 | `tab-log` | 최근 30개 점수 변경 이력 |
| 📊 현황 | `tab-scores` | 팀/개인 전체 점수 테이블 |

### 점수 변경 시 자동으로 하는 일
1. Firebase `teams/{key}` 또는 `participants/{uid}` 의 `score` 업데이트
2. `scoreLogs`에 `{ target, value, timestamp }` push

---

## CSS 디자인 시스템

```css
/* 컬러 팔레트 (라이트 테마) */
--bg:       #f4f6fb   /* 페이지 배경 */
--surface:  #ffffff   /* 카드 배경 */
--surface2: #f0f2f8   /* 인풋, pill 배경 */
--border:   #e2e6f0   /* 테두리 */
--accent:   #5b50f0   /* 주요 액션, 팀A */
--accent2:  #f05070   /* 강조, 팀B */
--accent3:  #16b864   /* 점수, 성공, 팀C */
--accent4:  #e08b00   /* 관리자 페이지 포인트 */
--text:     #1a1a2e   /* 본문 */
--text2:    #7a7a9a   /* 보조 텍스트 */
```

```css
/* 폰트 */
'Noto Sans KR'  /* 본문 */
'Space Mono'    /* 점수 숫자 */
```

주요 컴포넌트 클래스: `.section-card`, `.section-header`, `.score-pill`, `.tab-btn`, `.score-btn.plus/minus`, `.rank-item`, `.team-rank-item`

---

## 앞으로 할 작업 목록

### 1. 명단 데이터 업데이트
실제 워크샵 참가자로 `sample-data.json` 교체 후 Firebase에 재업로드.

작업 내용:
- `participants` 객체에 참가자 추가/수정/삭제
- `teams` 객체에 팀 추가/수정
- `uid` 키 형식 유지: `user_001`, `user_002`, ...
- `code`는 전화번호 뒤 4자리 (중복 없어야 함)

### 2. 작은 기능들 추가
예상 작업들 (확정 아님):
- 점수 애니메이션 강화
- 관리자 페이지에서 공지 삭제 기능
- 참가자 페이지에서 최근 점수 변동 알림 (예: "+10점!")
- 팀별 참가자 목록 보기
- 점수 초기화 버튼 (관리자)

### 3. 미니게임 기능 추가
예상 구조:
- 관리자가 게임 시작 → Firebase에 게임 상태 저장
- 참가자 화면에 게임 UI 표시 (실시간)
- 게임 결과 → 점수 자동 반영
- 별도 탭 또는 모달로 구현

---

## 수정 시 주의사항

1. **Firebase 실시간 특성** — `onValue` 리스너는 데이터 변경 시 자동 재실행됨. 렌더링 함수는 항상 전체를 다시 그리는 방식.
2. **모바일 전용** — `max-width`로 너비 제한하지 않음. 모든 요소가 전체 너비 사용.
3. **ES Module** — `<script type="module">` 사용. `window.XXX =` 패턴으로 전역 함수 노출.
4. **인증 없음** — Firebase 규칙이 `.read/.write: true`인 테스트 모드. 실제 운영 전 규칙 강화 필요.
5. **파일 3개가 전부** — 빌드 없음. 수정 후 Cloudflare Pages에 파일 직접 업로드하면 배포 완료.

---

## Cloudflare Pages 배포

- 프로젝트명: (배포 후 확인)
- 배포 방법: Cloudflare Dashboard → Pages → 직접 업로드 → `index.html`, `admin.html` 선택
- 자동 HTTPS 적용됨
