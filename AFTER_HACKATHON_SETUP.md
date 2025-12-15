# After 해커톤 Google Apps Script 설정 가이드

## 1. Google Sheets 설정

1. 기존 해커톤 Google Sheets 파일을 엽니다
2. 하단에 새 시트 탭을 추가하고 이름을 **"후속프로그램"**으로 설정합니다
3. 도구 → 스크립트 편집기를 엽니다

## 2. Apps Script 코드 복사

아래 전체 코드를 복사하여 Apps Script 편집기에 붙여넣습니다:

```javascript
// After 해커톤 신청 처리
function doPost(e) {
  try {
    var spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
    var data = JSON.parse(e.postData.contents);

    // 시트 선택 (기존 해커톤과 구분하기 위해)
    var sheetName = '후속프로그램';
    var sheet = spreadsheet.getSheetByName(sheetName);

    // 시트가 없으면 생성
    if (!sheet) {
      sheet = spreadsheet.insertSheet(sheetName);
      // 헤더 추가
      sheet.appendRow([
        '제출시간',
        '이름',
        '이메일',
        '연락처',
        '소속',
        '해커톤 서비스 설명',
        '완성 목표',
        '팀 구성'
      ]);
    }

    // KST 시간 생성
    var now = new Date();
    var kstTimestamp = Utilities.formatDate(now, 'Asia/Seoul', 'yyyy-MM-dd HH:mm:ss');

    // 데이터 추가
    sheet.appendRow([
      kstTimestamp,
      data.name || '',
      data.email || '',
      data.phone || '',
      data.affiliation || '',
      data.service || '',
      data.goal || '',
      data.teamSize || ''
    ]);

    return ContentService.createTextOutput(JSON.stringify({
      status: 'success',
      message: '신청이 완료되었습니다.'
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    Logger.log('Error: ' + error.toString());
    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

## 3. 배포하기

1. Apps Script 편집기 상단의 **배포** → **새 배포** 클릭
2. 톱니바퀴 아이콘(⚙️) 클릭 → **웹 앱** 선택
3. 설정:
   - **실행 계정**: 나
   - **액세스 권한**: **모든 사용자**
4. **배포** 버튼 클릭
5. 권한 승인 (Google 계정 로그인 및 권한 허용)
6. 배포된 **웹 앱 URL** 복사

## 4. after-hackathon.html 파일 수정

1. `after-hackathon.html` 파일을 엽니다
2. **line 1057** 찾기:
   ```javascript
   const GOOGLE_SCRIPT_URL = 'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE';
   ```
3. `'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE'`를 복사한 웹 앱 URL로 교체:
   ```javascript
   const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycby.../exec';
   ```

## 5. 테스트

1. 로컬에서 `after-hackathon.html` 열기
2. 폼 작성 후 제출
3. Google Sheets "후속프로그램" 탭에서 데이터 확인

## 데이터 구조

Google Sheets "후속프로그램" 탭 컬럼:

| A | B | C | D | E | F | G | H |
|---|---|---|---|---|---|---|---|
| 제출시간 | 이름 | 이메일 | 연락처 | 소속 | 해커톤 서비스 설명 | 완성 목표 | 팀 구성 |

## 문제 해결

### 데이터가 저장되지 않는 경우
- "후속프로그램" 시트 이름이 정확한지 확인하세요
- Apps Script 배포 시 "모든 사용자" 액세스 권한을 선택했는지 확인하세요

### CORS 에러가 발생하는 경우
- `mode: 'no-cors'` 설정이 이미 되어있으므로 정상입니다
- 실제 데이터는 정상적으로 저장됩니다
