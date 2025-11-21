# Google Sheets 연동 설정 가이드

해커톤 참가 신청 폼을 Google Sheets와 연동하는 방법입니다.

## 1단계: Google Sheets 생성

1. [Google Sheets](https://sheets.google.com) 접속
2. 새 스프레드시트 만들기
3. 이름: "경기도 AI 해커톤 신청자 명단"
4. 첫 번째 행에 다음 헤더 입력:

| A | B | C | D | E | F | G | H | I | J | K | L | M |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 제출시간 | 이름 | 이메일 | 연락처 | 소속 | 트랙 | 지원동기 | 운영체제 | Claude설치 | Claude연동 | 고유ID | 출석여부 | 출석시간 |

**참가자 환경 정보:**
- **운영체제 (H열):** Windows 또는 Mac
- **Claude설치 (I열):** Claude Code 설치 여부 (예/아니오)
- **Claude연동 (J열):** Claude Code 계정 연동 여부 (예/아니오)

**출석 체크 관련 컬럼:**
- **고유ID (K열):** QR 코드에 인코딩된 참가자 고유 식별자
- **출석여부 (L열):** 체크인 완료 시 'O' 표시
- **출석시간 (M열):** 체크인한 시간 자동 기록

## 2단계: Google Apps Script 생성

1. Google Sheets에서 **확장 프로그램** → **Apps Script** 클릭
2. 기본 코드를 삭제하고 아래 코드를 붙여넣기:

```javascript
// 참가 신청 처리 함수
function doPost(e) {
  try {
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    const data = JSON.parse(e.postData.contents);
    const trackName = data.division === 'general' ? '일반 트랙' : '개발 트랙';

    // 고유ID 생성 (타임스탬프 + 이메일 해시)
    const timestamp = new Date().getTime();
    const emailHash = Utilities.computeDigest(
      Utilities.DigestAlgorithm.MD5,
      data.email
    ).map(byte => (byte & 0xFF).toString(16).padStart(2, '0')).join('').substring(0, 8);
    const uniqueId = `HK2025-${timestamp}-${emailHash}`;

    // 데이터 저장 (고유ID 포함)
    sheet.appendRow([
      data.timestamp || new Date().toISOString(),
      data.name || '',
      data.email || '',
      data.phone || '',
      data.affiliation || '',
      trackName,
      data.motivation || '',
      data.os || '',              // H열: 운영체제
      data.claudeInstalled || '', // I열: Claude Code 설치 여부
      data.claudeConnected || '', // J열: Claude Code 계정 연동 여부
      uniqueId,                   // K열: 고유ID
      '',                         // L열: 출석여부 (비어있음)
      ''                          // M열: 출석시간 (비어있음)
    ]);

    // QR 코드 URL 생성 (Google Charts API)
    const qrCodeUrl = `https://chart.googleapis.com/chart?cht=qr&chs=300x300&chl=${encodeURIComponent(uniqueId)}`;

    // 이메일 발송 (HTML 포맷, QR 코드 포함)
    if (data.email) {
      const emailSubject = '경기도 AI 바이브코딩 해커톤 2025 신청 완료';
      const htmlBody = `
        <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
          <h2 style="color: #6DB544;">안녕하세요 ${data.name}님,</h2>
          <p>경기도 AI 바이브코딩 해커톤 2025 신청이 완료되었습니다.</p>

          <div style="background: #f5f5f5; padding: 20px; border-radius: 10px; margin: 20px 0;">
            <h3 style="margin-top: 0;">📋 신청 정보</h3>
            <ul style="line-height: 1.8;">
              <li><strong>이름:</strong> ${data.name}</li>
              <li><strong>이메일:</strong> ${data.email}</li>
              <li><strong>연락처:</strong> ${data.phone}</li>
              <li><strong>소속:</strong> ${data.affiliation}</li>
              <li><strong>선택 트랙:</strong> ${trackName}</li>
              <li><strong>운영체제:</strong> ${data.os}</li>
              <li><strong>Claude Code 설치:</strong> ${data.claudeInstalled}</li>
              <li><strong>Claude Code 계정 연동:</strong> ${data.claudeConnected}</li>
            </ul>
          </div>

          <div style="background: #f5f5f5; padding: 20px; border-radius: 10px; margin: 20px 0;">
            <h3 style="margin-top: 0;">📅 행사 정보</h3>
            <ul style="line-height: 1.8;">
              <li><strong>일시:</strong> 2025년 11월 29일 (토) 09:00-16:00</li>
              <li><strong>장소:</strong> 경기도의회 대회의실</li>
              <li><strong>준비물:</strong> 개인 노트북</li>
            </ul>
          </div>

          <div style="background: #fff3cd; padding: 20px; border-radius: 10px; margin: 20px 0; border-left: 4px solid #6DB544;">
            <h3 style="margin-top: 0;">💡 참고사항</h3>
            <ul style="line-height: 1.8;">
              <li>Claude Code 유료 계정이 임시 제공됩니다</li>
              <li>${trackName === '일반 트랙' ? '09:35-12:00 교육 진행 후 개발이 시작됩니다' : '바로 개발을 시작합니다'}</li>
              <li>점심 식사는 별도로 제공되지 않습니다</li>
            </ul>
          </div>

          <div style="background: #e8f5e9; padding: 20px; border-radius: 10px; margin: 20px 0; text-align: center;">
            <h3 style="color: #6DB544; margin-top: 0;">🎫 입장용 QR 코드</h3>
            <p style="color: #666;">행사 당일 이 QR 코드를 제시해주세요</p>
            <img src="${qrCodeUrl}" alt="입장 QR 코드" style="width: 250px; height: 250px; margin: 10px 0;">
            <p style="font-size: 12px; color: #999;">QR 코드가 보이지 않으면 이메일을 다시 확인해주세요</p>
          </div>

          <p style="color: #666; margin-top: 30px;">자세한 일정 및 안내사항은 행사 전 다시 연락드리겠습니다.</p>
          <p style="color: #666;">문의사항이 있으시면 <a href="mailto:partner@seeso.kr">partner@seeso.kr</a>로 연락주세요.</p>

          <hr style="border: none; border-top: 1px solid #ddd; margin: 30px 0;">
          <p style="color: #999; font-size: 12px; text-align: center;">
            경기도 AI 바이브코딩 해커톤 2025<br>
            주최: 경기도의회 | 문의: partner@seeso.kr
          </p>
        </div>
      `;

      MailApp.sendEmail({
        to: data.email,
        subject: emailSubject,
        htmlBody: htmlBody
      });
    }

    return ContentService.createTextOutput(JSON.stringify({
      status: 'success',
      message: '신청이 완료되었습니다.'
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}

// 출석 체크 API 함수
function doGet(e) {
  try {
    const action = e.parameter.action;
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();

    // 출석 체크인
    if (action === 'checkIn') {
      const uniqueId = e.parameter.id;

      if (!uniqueId) {
        return ContentService.createTextOutput(JSON.stringify({
          status: 'error',
          message: '잘못된 QR 코드입니다.'
        })).setMimeType(ContentService.MimeType.JSON);
      }

      // 고유ID로 참가자 찾기 (K열에서 검색)
      const dataRange = sheet.getDataRange();
      const values = dataRange.getValues();

      for (let i = 1; i < values.length; i++) {  // 0은 헤더 행
        if (values[i][10] === uniqueId) {  // K열 (고유ID)
          // 이미 체크인되었는지 확인 (L열)
          if (values[i][11]) {  // 출석여부가 이미 있으면
            return ContentService.createTextOutput(JSON.stringify({
              status: 'duplicate',
              message: `${values[i][1]}님은 이미 체크인하셨습니다.`,
              name: values[i][1],
              track: values[i][5],
              checkInTime: values[i][12]
            })).setMimeType(ContentService.MimeType.JSON);
          }

          // 체크인 처리
          const now = new Date();
          const timeString = Utilities.formatDate(now, 'Asia/Seoul', 'HH:mm:ss');

          sheet.getRange(i + 1, 12).setValue('O');  // L열: 출석여부
          sheet.getRange(i + 1, 13).setValue(timeString);  // M열: 출석시간

          return ContentService.createTextOutput(JSON.stringify({
            status: 'success',
            message: `${values[i][1]}님 체크인 완료!`,
            name: values[i][1],
            track: values[i][5],
            checkInTime: timeString
          })).setMimeType(ContentService.MimeType.JSON);
        }
      }

      // 참가자를 찾지 못한 경우
      return ContentService.createTextOutput(JSON.stringify({
        status: 'error',
        message: '등록되지 않은 참가자입니다.'
      })).setMimeType(ContentService.MimeType.JSON);
    }

    // 최근 체크인 목록 조회
    if (action === 'getRecent') {
      const dataRange = sheet.getDataRange();
      const values = dataRange.getValues();
      const recentCheckIns = [];

      for (let i = values.length - 1; i >= 1 && recentCheckIns.length < 5; i--) {
        if (values[i][11]) {  // 출석여부가 있으면 (L열)
          recentCheckIns.push({
            name: values[i][1],
            track: values[i][5],
            checkInTime: values[i][12]
          });
        }
      }

      return ContentService.createTextOutput(JSON.stringify({
        status: 'success',
        recent: recentCheckIns
      })).setMimeType(ContentService.MimeType.JSON);
    }

    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: '잘못된 요청입니다.'
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      status: 'error',
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

3. **저장** 버튼 클릭 (💾 아이콘)
4. 프로젝트 이름: "해커톤 신청 처리"

## 3단계: 배포하기

1. Apps Script 편집기에서 **배포** → **새 배포** 클릭
2. 설정:
   - **유형 선택**: ⚙️ 아이콘 → "웹 앱" 선택
   - **설명**: "해커톤 신청 폼 v1"
   - **실행 사용자**: "나"
   - **액세스 권한**: "**모든 사용자**" (중요!)
3. **배포** 버튼 클릭
4. 권한 승인:
   - "권한 검토" 클릭
   - Google 계정 선택
   - "고급" → "프로젝트명(안전하지 않음)으로 이동" 클릭
   - "허용" 클릭
5. **웹 앱 URL** 복사 (예: `https://script.google.com/macros/s/AKfycby.../exec`)

## 4단계: HTML 파일에 URL 입력

1. `index.html` 파일 열기
2. 약 2230번 라인에서 다음 코드 찾기:
```javascript
const GOOGLE_SCRIPT_URL = 'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE';
```

3. URL 교체:
```javascript
const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycby.../exec';
```

## 5단계: 테스트

1. 웹사이트에서 참가 신청 폼 작성 및 제출
2. Google Sheets 확인 → 데이터가 자동으로 추가되었는지 확인

## 문제 해결

### 데이터가 저장되지 않는 경우

1. **Apps Script 로그 확인**:
   - Apps Script 편집기 → 실행 로그 확인

2. **권한 재설정**:
   - 배포 → 배포 관리 → 편집 → 권한 재확인

3. **브라우저 콘솔 확인**:
   - F12 → Console 탭에서 에러 메시지 확인

### CORS 에러가 발생하는 경우

- `mode: 'no-cors'` 옵션이 설정되어 있어 정상입니다
- Google Apps Script는 no-cors 모드에서만 작동합니다

## 이메일 자동 발송 기능

위 코드에는 **이메일 자동 발송 기능이 이미 포함**되어 있습니다.

신청자가 폼을 제출하면 다음 내용의 이메일이 자동으로 발송됩니다:
- 신청 완료 확인
- 신청 정보 요약 (이름, 트랙, 소속 등)
- 행사 일시 및 장소
- 준비사항 및 참고사항

### 이메일 내용 커스터마이징

이메일 내용을 수정하려면 Apps Script 코드에서 `emailBody` 변수의 내용을 수정하세요 (46-80번 라인).

## 데이터 관리

### Google Sheets에서 확인

- 실시간으로 신청자 목록 확인
- 필터, 정렬 기능 사용 가능
- CSV 또는 Excel로 내보내기 가능

### 통계 확인

간단한 수식으로 통계 확인:
- 총 신청자: `=COUNTA(B:B)-1`
- 일반 트랙: `=COUNTIF(F:F,"일반 트랙")`
- 개발 트랙: `=COUNTIF(F:F,"개발 트랙")`
- Windows 사용자: `=COUNTIF(H:H,"Windows")`
- Mac 사용자: `=COUNTIF(H:H,"Mac")`
- Claude 미설치자: `=COUNTIF(I:I,"아니오")`
- Claude 미연동자: `=COUNTIF(J:J,"아니오")`
- 총 출석자: `=COUNTIF(L:L,"O")`

## 보안 팁

1. Google Sheets는 본인만 볼 수 있도록 설정
2. Apps Script URL은 공개되어도 괜찮음 (읽기 전용 불가능)
3. 정기적으로 데이터 백업

---

## 출석 체크 시스템 사용하기

### 1단계: attendance.html 설정

1. 프로젝트의 `attendance.html` 파일 열기
2. 43번 라인 찾기:
```javascript
const SCRIPT_URL = 'YOUR_GOOGLE_APPS_SCRIPT_URL_HERE';
```

3. Google Apps Script URL로 교체:
```javascript
const SCRIPT_URL = 'https://script.google.com/macros/s/AKfycby.../exec';
```

4. (선택사항) 보안 토큰 변경 (44번 라인):
```javascript
const VALID_TOKEN = 'HACKATHON2025SECRET';  // 원하는 값으로 변경
```

### 2단계: 담당자에게 링크 전달

배포 후 다음 링크를 담당자들에게만 공유:
```
https://gyunggido-climate-data.vercel.app/attendance.html?token=HACKATHON2025SECRET
```

**⚠️ 주의:** 토큰을 변경했다면 URL의 토큰도 함께 변경하세요!

### 3단계: 현장에서 사용하기

**담당자:**
1. 비밀 링크 접속
2. 카메라 권한 허용
3. 참가자의 QR 코드 스캔
4. 자동으로 출석 체크 완료!

**참가자:**
1. 이메일에서 받은 QR 코드를 핸드폰 화면에 띄움
2. 담당자에게 QR 코드 제시
3. 체크인 완료!

### 출석 현황 확인하기

Google Sheets에서 실시간으로 확인:
- **L열 (출석여부):** 'O' 표시가 있으면 체크인 완료
- **M열 (출석시간):** 체크인한 시간 기록
- **필터 사용:** L열에 'O'만 필터링하면 출석자만 볼 수 있음

### 참가자 환경 정보 확인

- **H열 (운영체제):** Windows/Mac 통계로 현장 세팅 준비
- **I열 (Claude설치):** 미설치자를 위한 현장 지원 준비
- **J열 (Claude연동):** 미연동자를 위한 계정 연동 안내 준비

### 문제 해결

**QR 코드가 인식되지 않을 때:**
- 조명이 충분한지 확인
- QR 코드를 카메라 정중앙에 위치
- 핸드폰 화면 밝기 최대로 설정

**"이미 체크인하셨습니다" 메시지:**
- 정상 작동입니다 (중복 체크인 방지)
- Google Sheets의 J열에서 체크인 시간 확인 가능

**카메라가 작동하지 않을 때:**
- 브라우저 설정에서 카메라 권한 확인
- HTTPS 연결 확인 (Vercel은 자동으로 HTTPS 제공)

---

**완료!** 이제 참가 신청부터 QR 출석 체크까지 자동화됩니다. 🎉
