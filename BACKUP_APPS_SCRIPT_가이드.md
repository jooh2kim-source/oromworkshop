# 🔗 Google Sheets 자동 백업 연동 가이드

관리자 콘솔의 **💾 전체 점수 백업** 버튼을 눌렀을 때, CSV 다운로드와 함께 Google Sheets에도 자동으로 누적 기록되도록 연결하는 1회 설정 가이드.

---

## 1. Google Sheet 만들기

1. [Google Sheets](https://sheets.google.com)에서 새 스프레드시트 생성
2. 이름: 예) `2026 오롬 워크샵 점수 백업`
3. 시트 탭 2개 만들기 (자동 생성도 됨, 미리 만들면 더 깔끔):
   - `개인 점수`
   - `팀 점수`

## 2. Apps Script 추가

1. 시트 상단 메뉴: **확장 프로그램 → Apps Script**
2. 기본 코드 모두 지우고 아래 코드 붙여넣기:

```javascript
function doPost(e) {
  try {
    const ss = SpreadsheetApp.getActiveSpreadsheet();
    const data = JSON.parse(e.postData.contents);
    const ts  = data.timestamp || new Date().toISOString();

    // 시트 가져오기 (없으면 생성)
    let pSheet = ss.getSheetByName('개인 점수');
    if (!pSheet) {
      pSheet = ss.insertSheet('개인 점수');
      pSheet.appendRow(['시간','UID','이름','표시이름','피지컬47팀','바베큐팀','숙소','점수']);
    }
    let tSheet = ss.getSheetByName('팀 점수');
    if (!tSheet) {
      tSheet = ss.insertSheet('팀 점수');
      tSheet.appendRow(['시간','구분','팀키','팀이름','점수','팀장']);
    }

    // 한 번에 append (성능)
    const pRows = (data.participants || []).map(p =>
      [ts, p.uid, p.name, p.displayName||p.name, p.team||'', p.bbqTeam||'', p.room||'', p.score||0]
    );
    if (pRows.length) {
      pSheet.getRange(pSheet.getLastRow()+1, 1, pRows.length, 8).setValues(pRows);
    }

    const tRows = [];
    (data.teams || []).forEach(t => tRows.push([ts, '피지컬47', t.key, t.name, t.score||0, t.captainName||'']));
    (data.bbqTeams || []).forEach(t => tRows.push([ts, '바베큐', t.key, t.name, t.score||0, t.captainName||'']));
    if (tRows.length) {
      tSheet.getRange(tSheet.getLastRow()+1, 1, tRows.length, 6).setValues(tRows);
    }

    return ContentService
      .createTextOutput(JSON.stringify({ ok: true, count: pRows.length + tRows.length }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ ok: false, error: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

3. 좌측 상단 💾 저장

## 3. 웹 앱으로 배포

1. 우측 상단 **배포 → 새 배포**
2. ⚙️ 톱니바퀴 → **웹 앱**
3. 설정:
   - 설명: `점수 백업 수신`
   - 다음 사용자 인증: **나** (본인 계정)
   - 액세스 권한: **모든 사용자** (URL만 알면 누구나)
4. **배포** 클릭
5. 권한 승인 (Google 로그인 → 신뢰할 수 없는 앱 경고 → 고급 → 안전하지 않음으로 이동 → 허용)
6. 발급된 **웹 앱 URL** 복사 (`https://script.google.com/macros/s/.../exec` 형태)

## 4. 관리자 페이지에 URL 붙여넣기

1. https://oromtf.pages.dev → 📊 현황 탭 → 💾 전체 점수 백업 카드
2. 하단 **스프레드시트 URL** 입력란에 위에서 복사한 URL 붙여넣기
3. **저장** 클릭

이제 **💾 전체 점수 백업** 버튼 누르면:
- ✅ CSV 파일 즉시 다운로드 (안전망)
- ✅ Google Sheet에 새 행으로 누적 기록 (시간순 추적 가능)

---

## 트러블슈팅

| 증상 | 해결 |
|---|---|
| 스프레드시트에 기록 안 됨 | URL 복사 정확한지 확인 (`/exec` 끝나는지). 코드 수정 후엔 **새 배포** 필요 (기존 URL은 옛 코드 그대로) |
| 한글 깨짐 | Apps Script는 자동 UTF-8. 시트에서 깨지면 데이터→텍스트→나누기로 다시 정리 |
| 너무 많이 쌓임 | 시간 컬럼으로 필터/정렬해서 필요 구간만 보세요 |
| 권한 오류 | 배포 시 액세스 "모든 사용자" 확인. 본인 계정으로 실행 권한 |

## 데이터 컬럼 안내

**개인 점수 시트**:
| 시간 | UID | 이름 | 표시이름 | 피지컬47팀 | 바베큐팀 | 숙소 | 점수 |
|---|---|---|---|---|---|---|---|
| ISO timestamp | user_001 | 강현준 | 강현준 | 1조 | 2조 | 2호실 | 250 |

**팀 점수 시트**:
| 시간 | 구분 | 팀키 | 팀이름 | 점수 | 팀장 |
|---|---|---|---|---|---|
| ISO timestamp | 피지컬47 | 1조 | 1조 | 290 | 강현준 |
| ISO timestamp | 바베큐 | 1조 | 1조 | 180 | 김주희 |

---

매 백업마다 47개 개인 행 + 16개 팀 행 (피지컬47 7 + 바베큐 9) = 63행씩 쌓입니다.
