# 📋 57TB 직원 등록 LIFF 앱 프로젝트 완전 가이드

## 🎯 프로젝트 개요

### 목적
- 57TB 미용실 직원들이 LINE을 통해 쉽게 개인정보를 등록할 수 있는 시스템
- LINE Bot 친구 추가 → LIFF 앱 → 정보 입력 → Notion 자동 저장

### 완성일
- **2025년 1월 27일** 완료

### 기술 스택
- **Frontend**: HTML/CSS/JavaScript (LIFF SDK)
- **Backend**: n8n (자동화 워크플로우)
- **Database**: Notion (57TB Employee Master)
- **Hosting**: Netlify (자동 배포)
- **Repository**: GitHub

---

## 🏗️ 시스템 아키텍처

```
직원 → LINE Bot → LIFF 앱 → n8n API → Notion DB
```

### 상세 플로우
1. **직원**: LINE에서 57TB Bot 친구 추가
2. **자동 인사말**: LIFF 앱 링크 포함 메시지 발송
3. **LIFF 앱**: 직원 정보 입력 폼 제공
4. **n8n API**: 입력 데이터 처리 및 검증
5. **Notion**: 57TB Employee Master DB에 정보 저장

---

## 📊 데이터 구조

### Notion: 57TB Employee Master
| 필드명 | 타입 | 설명 | 옵션값 |
|--------|------|------|--------|
| Name | Title | 직원 이름 | - |
| ID | Unique ID | 직원 ID | A57250-XX 형식 |
| Status | Select | 재직 상태 | Active, Resigned |
| Branch | Select | 근무 지점 | Asoke, SaiMai |
| Team | Select | 소속 팀 | HEAD OFFICE, อนาคต, Extra, Manager Team, Saimai, มั่งมีศรีสุข |
| Position | Select | 직급 | CLN, CEO, TL, HS, STP, RECEPTION, ADMIN, SV, CFO |
| Email | Email | 이메일 | - |
| assigned stylist | Text | 담당 스타일리스트 | - |
| LINE User ID | Text | LINE 사용자 ID | - |
| Notes | Rich Text | 비고 | - |
| Phone | Phone | 전화번호 | - |

---

## 🎨 LIFF 앱 기능 상세

### 단계별 워크플로우

#### 1단계: 직원 이름 선택
- **기능**: n8n API에서 미등록 직원 목록 조회
- **UI**: 검색창 + 직원 목록 (이름, 직원ID 표시)
- **API**: `/api/unregistered-employees`

#### 2단계: 지점 선택
- **옵션**: 
  - **Asoke** (สาขาอโศก)
  - **SaiMai** (สาขาสายไหม)

#### 3단계: 직급 선택
- **옵션**:
  - **STP** (นักเรียนฝึกงาน) - 인턴/학습생
  - **RECEPTION** (แผนกต้อนรับ) - 리셉션
  - **CLN** (แผนกทำความสะอาด) - 청소
  - **ADMIN** (ออนไลน์คอนซัลแทนต์) - 온라인 상담 직원
  - **HS** (นักออกแบบทรงผม) - 헤어 디자이너

#### 4단계: 팀 선택 (조건부)
- **자동 할당**:
  - CLN, RECEPTION, ADMIN → Manager Team
  - SaiMai 지점의 HS, STP → Saimai 팀
- **선택 필요**:
  - Asoke 지점의 HS, STP → อนาคต, มั่งมีศรีสุข, Extra 중 선택

#### 5단계: 정보 확인
- **표시**: 입력한 모든 정보 요약
- **액션**: 수정 또는 최종 제출

#### 6단계: 완료
- **기능**: n8n으로 데이터 전송 후 성공 메시지
- **자동**: 3초 후 LIFF 앱 종료

---

## 🔧 팀 할당 로직 상세

### 자동 할당 규칙
```javascript
if (position === 'CLN' || position === 'RECEPTION' || position === 'ADMIN') {
    team = 'Manager Team';
    // 팀 선택 단계 건너뛰기
} else if (branch === 'SaiMai' && (position === 'HS' || position === 'STP')) {
    team = 'Saimai';
    // 팀 선택 단계 건너뛰기
} else {
    // Asoke 지점의 HS, STP만 팀 선택 단계로 이동
    showTeamSelection();
}
```

### 팀별 직급 매트릭스
| 지점 | 직급 | 할당 방식 | 팀 옵션 |
|------|------|-----------|---------|
| Asoke | HS, STP | 선택 | อนาคต, มั่งมีศรีสุข, Extra |
| Asoke | CLN, RECEPTION, ADMIN | 자동 | Manager Team |
| SaiMai | HS, STP | 자동 | Saimai |
| SaiMai | CLN, RECEPTION, ADMIN | 자동 | Manager Team |

---

## 🌐 API 연동 정보

### n8n Webhook URLs
```
BASE_URL: https://dylan-automation.app.n8n.cloud/webhook

ENDPOINTS:
- /api/unregistered-employees (GET) - 미등록 직원 목록
- /api/team-options (GET) - 팀 옵션 (branch, position 파라미터)
- /api/update-employee (POST) - 직원 정보 업데이트
```

### API 요청/응답 예시

#### 미등록 직원 조회
```http
GET /api/unregistered-employees
Response: [
  {
    "id": "A57250-63",
    "name": "ZIN",
    "employeeId": "A57250-63"
  }
]
```

#### 직원 정보 업데이트
```http
POST /api/update-employee
Body: {
  "employeeId": "A57250-63",
  "name": "ZIN",
  "branch": "Asoke",
  "position": "HS",
  "team": "อนาคต",
  "lineUserId": "U1234567890abcdef"
}
```

---

## 🎛️ LINE 설정 정보

### LIFF 앱 설정
- **LIFF ID**: `2007748465-jvdwq0nG`
- **LIFF URL**: `https://liff.line.me/2007748465-jvdwq0nG`
- **Endpoint URL**: `https://bejewelled-moxie-ba664a.netlify.app`
- **Size**: Compact
- **Scopes**: openid, profile

### LINE Bot 정보
- **Provider**: LINE Official Account Manager
- **인사말 메시지**: LIFF 링크 포함
- **Rich Menu**: 필요 시 설정 가능

---

## 🚀 배포 및 호스팅

### GitHub Repository
- **URL**: `https://github.com/jiraprapa-ja/57tb-employee-liff-app`
- **브랜치**: master
- **파일**: index.html (단일 파일)

### Netlify 배포
- **사이트 URL**: `https://bejewelled-moxie-ba664a.netlify.app`
- **자동 배포**: GitHub push 시 자동 빌드/배포
- **도메인**: Netlify 기본 도메인 사용

### 배포 프로세스
1. 로컬에서 index.html 수정
2. GitHub 저장소에 파일 업로드
3. Netlify 자동 배포 (1-2분)
4. LIFF 앱에 즉시 반영

---

## 🛠️ 개발 및 수정 가이드

### 로컬 개발 환경
```bash
# Git 초기화 (최초 1회)
git init
git remote add origin https://github.com/jiraprapa-ja/57tb-employee-liff-app.git

# 변경사항 커밋 및 푸시
git add index.html
git commit -m "설명"
git push origin master
```

### 주요 수정 포인트

#### 1. Position 옵션 추가/수정
**위치**: HTML 섹션 (라인 333-358)
```html
<div class="option-button" data-position="NEW_POSITION">
    <span class="icon">🔧</span>
    <div class="title">NEW_POSITION</div>
    <div class="desc">새로운 직급 설명</div>
</div>
```

#### 2. 팀 할당 로직 수정
**위치**: JavaScript 섹션 (라인 765-775)
```javascript
// 새 직급에 대한 로직 추가
if (selectedData.position === 'NEW_POSITION') {
    selectedData.team = '할당할 팀명';
    showStep(5);
}
```

#### 3. Fallback 데이터 수정
**위치**: JavaScript 섹션 (라인 584-602)
```javascript
// API 실패 시 사용할 기본 팀 옵션
if (position === 'NEW_POSITION') {
    teams = [{ value: '팀명', name: '팀명', desc: '설명' }];
}
```

### 브랜치/지점 추가
1. **HTML**: 브랜치 선택 옵션 추가 (라인 312-322)
2. **JavaScript**: 로직에 새 브랜치 조건 추가
3. **Notion**: Team 필드에 새 팀 옵션 추가

---

## 🧪 테스트 체크리스트

### 기능 테스트
- [ ] 모든 Position 옵션 표시
- [ ] 자동 팀 할당 (CLN, RECEPTION, ADMIN → Manager Team)
- [ ] SaiMai 지점 자동 할당 (HS, STP → Saimai)
- [ ] Asoke 지점 팀 선택 (HS, STP → 3개 옵션)
- [ ] 최종 데이터 전송 및 Notion 저장
- [ ] LIFF 앱 정상 종료

### 브라우저 테스트
- [ ] Chrome (데스크톱)
- [ ] Safari (iOS)
- [ ] Chrome (Android)
- [ ] LINE 내장 브라우저

### 에러 처리 테스트
- [ ] 네트워크 오류 시 fallback 데이터 사용
- [ ] API 빈 응답 시 기본 팀 옵션 제공
- [ ] LIFF 초기화 실패 시 에러 메시지

---

## 🔍 문제 해결 가이드

### 자주 발생하는 문제

#### 1. LIFF 앱이 로드되지 않음
**원인**: Endpoint URL 문제
**해결**: LINE Developers Console에서 LIFF 설정 확인
- Endpoint URL: `https://bejewelled-moxie-ba664a.netlify.app` (정확히)
- Size: Compact
- `/collector.html` 등 잘못된 경로 제거

#### 2. 팀 선택 옵션이 나오지 않음
**원인**: n8n API 빈 응답
**해결**: Fallback 데이터 확인
- Console에서 "API returned empty array, using fallback data" 메시지 확인
- JavaScript fallback 로직 점검

#### 3. 데이터가 Notion에 저장되지 않음
**원인**: n8n API 연동 문제
**해결**: 
- n8n 워크플로우 상태 확인
- API 엔드포인트 URL 확인
- POST 데이터 형식 검증

#### 4. 특정 직급/팀 조합 오류
**원인**: 로직 누락
**해결**: 팀 할당 로직 및 fallback 데이터에 해당 케이스 추가

### 디버깅 도구
```javascript
// Console 로그 활성화
console.log('Current data:', selectedData);
console.log('API response:', response);
console.log('Team options:', teams);
```

---

## 📈 향후 확장 계획

### 가능한 개선사항
1. **다국어 지원**: 태국어/영어 전환
2. **추가 필드**: 전화번호, 주소 등
3. **파일 업로드**: 프로필 사진, 서류
4. **실시간 검증**: 중복 체크, 유효성 검사
5. **알림 기능**: 등록 완료 후 관리자 알림
6. **통계 대시보드**: 등록 현황 모니터링

### 확장 시 고려사항
- **성능**: 직원 수 증가에 따른 최적화
- **보안**: 개인정보 보호 강화
- **사용성**: UI/UX 개선
- **유지보수**: 코드 모듈화 및 문서화

---

## 📞 연락처 및 지원

### 개발자 정보
- **이름**: Claude (AI Assistant)
- **프로젝트 관리자**: Dylan
- **이메일**: jiraprapa.ja@gmail.com
- **완료일**: 2025년 1월 27일

### 지원 범위
- 버그 수정 및 기능 개선
- 새로운 직급/팀 추가
- UI/UX 개선
- n8n 워크플로우 연동

---

## 📋 변경 이력

### v1.0 (2025-01-27)
- ✅ 초기 LIFF 앱 개발 완료
- ✅ 5단계 워크플로우 구현
- ✅ n8n API 연동
- ✅ Notion DB 저장
- ✅ 기본 Position 옵션 (STP, RECEPTION, CLN)
- ✅ 팀 자동 할당 로직

### v1.1 (2025-01-27)
- ✅ ADMIN, HS Position 옵션 추가
- ✅ 개선된 팀 할당 로직
- ✅ API 빈 응답 처리 fallback 추가
- ✅ Asoke 지점 HS 팀 선택 기능

---

## 🎯 결론

57TB 직원 등록 LIFF 앱이 성공적으로 완료되어 모든 직원이 쉽게 LINE을 통해 등록할 수 있게 되었습니다. 

**핵심 성과**:
- 📱 간편한 LINE 기반 등록 시스템
- 🏢 지점/직급별 맞춤 팀 할당
- 🔄 완전 자동화된 데이터 처리
- 🛠️ 유지보수 용이한 구조

이 문서를 바탕으로 향후 변경사항이나 확장 작업을 효율적으로 진행할 수 있습니다.





