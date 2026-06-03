# 📊 57TB LIFF 앱 데이터 흐름 및 API 명세서

## 📋 목차
1. [전체 데이터 흐름](#전체-데이터-흐름)
2. [API 상세 명세](#api-상세-명세)
3. [데이터 변환 로직](#데이터-변환-로직)
4. [에러 핸들링](#에러-핸들링)
5. [보안 및 인증](#보안-및-인증)
6. [성능 최적화](#성능-최적화)

---

## 🔄 전체 데이터 흐름

### 시스템 아키텍처 다이어그램
```
[직원] → [LINE Bot] → [LIFF App] → [n8n API] → [Notion DB]
   ↓         ↓           ↓           ↓           ↓
LINE ID   친구추가    정보입력    데이터처리   영구저장
```

### 상세 데이터 플로우

#### 1. 초기화 단계
```
직원 LINE 앱 실행
  ↓
57TB Bot 친구 추가
  ↓
자동 인사말 메시지 수신 (LIFF 링크 포함)
  ↓
LIFF 링크 클릭
  ↓
LIFF 앱 로드 시작
```

#### 2. 인증 및 데이터 로딩
```
LIFF SDK 초기화
  ↓
LINE 로그인 상태 확인
  ↓
사용자 프로필 정보 획득
  ↓
n8n API 호출 (미등록 직원 목록)
  ↓
직원 목록 렌더링
```

#### 3. 사용자 입력 단계
```
Step 1: 직원 이름 선택
  ↓
Step 2: 근무 지점 선택 (Asoke/SaiMai)
  ↓  
Step 3: 직급 선택 (STP/HS/RECEPTION/CLN/ADMIN)
  ↓
Step 4: 팀 선택 (조건부 - 자동할당 or 수동선택)
  ↓
Step 5: 정보 확인 및 최종 제출
```

#### 4. 데이터 저장 단계
```
LIFF 앱에서 최종 데이터 수집
  ↓
LINE User ID 포함하여 JSON 생성
  ↓
n8n API로 POST 요청
  ↓
n8n에서 Notion API 호출
  ↓
Notion 57TB Employee Master 업데이트
  ↓
성공 응답 반환
  ↓
LIFF 앱 종료
```

---

## 🌐 API 상세 명세

### Base Configuration
```javascript
const API_CONFIG = {
    BASE_URL: 'https://dylan-automation.app.n8n.cloud/webhook',
    TIMEOUT: 10000,
    RETRY_ATTEMPTS: 3,
    HEADERS: {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }
};
```

### 1. 미등록 직원 조회 API

#### Endpoint
```http
GET /api/unregistered-employees
```

#### Request Headers
```http
Content-Type: application/json
Accept: application/json
User-Agent: LIFF/2.0 57TB-Employee-App
```

#### Response Success (200)
```json
{
  "status": "success",
  "data": [
    {
      "id": "A57250-63",
      "name": "ZIN",
      "employeeId": "A57250-63",
      "status": "Active",
      "branch": null,
      "position": null,
      "team": null,
      "lineUserId": null
    },
    {
      "id": "A57250-62", 
      "name": "YU",
      "employeeId": "A57250-62",
      "status": "Active",
      "branch": null,
      "position": null,
      "team": null,
      "lineUserId": null
    }
  ],
  "count": 2,
  "timestamp": "2025-01-27T10:30:00Z"
}
```

#### Response Error (400/500)
```json
{
  "status": "error",
  "message": "Failed to fetch unregistered employees",
  "code": "FETCH_ERROR",
  "timestamp": "2025-01-27T10:30:00Z"
}
```

#### 클라이언트 데이터 변환
```javascript
// API 응답을 앱에서 사용할 형태로 변환
employees = data.map(emp => ({
    id: emp.employeeId || emp.id,
    name: emp.name,
    employeeId: emp.employeeId || emp.id
}));
```

### 2. 팀 옵션 조회 API

#### Endpoint
```http
GET /api/team-options?branch={branch}&position={position}
```

#### Request Parameters
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| branch | string | Yes | 근무 지점 | Asoke, SaiMai |
| position | string | Yes | 직급 | HS, STP, ADMIN, CLN, RECEPTION |

#### Request Example
```http
GET /api/team-options?branch=Asoke&position=HS
Content-Type: application/json
```

#### Response Success (200)
```json
{
  "status": "success",
  "data": [
    {
      "value": "อนาคต",
      "name": "อนาคต", 
      "desc": "ทีมอนาคต",
      "branch": "Asoke",
      "availablePositions": ["HS", "STP"]
    },
    {
      "value": "มั่งมีศรีสุข",
      "name": "มั่งมีศรีสุข",
      "desc": "ทีมมั่งมีศรีสุข", 
      "branch": "Asoke",
      "availablePositions": ["HS", "STP"]
    },
    {
      "value": "Extra",
      "name": "Extra",
      "desc": "ทีมเอ็กซ์ตร้า",
      "branch": "Asoke", 
      "availablePositions": ["HS", "STP"]
    }
  ],
  "count": 3,
  "query": {
    "branch": "Asoke",
    "position": "HS"
  },
  "timestamp": "2025-01-27T10:30:00Z"
}
```

#### Response Empty (200)
```json
{
  "status": "success", 
  "data": [],
  "count": 0,
  "query": {
    "branch": "Asoke",
    "position": "HS"
  },
  "message": "No teams available for this branch/position combination",
  "timestamp": "2025-01-27T10:30:00Z"
}
```

#### Fallback 처리 로직
```javascript
// API가 빈 배열 반환 시 클라이언트 fallback
if (!teams || teams.length === 0) {
    console.log('API returned empty array, using fallback data');
    
    const fallbackData = {
        'Asoke_HS': [
            { value: 'อนาคต', name: 'อนาคต', desc: 'ทีมอนาคต' },
            { value: 'มั่งมีศรีสุข', name: 'มั่งมีศรีสุข', desc: 'ทีมมั่งมีศรีสุข' },
            { value: 'Extra', name: 'Extra', desc: 'ทีมเอ็กซ์ตร้า' }
        ],
        'Asoke_STP': [
            { value: 'อนาคต', name: 'อนาคต', desc: 'ทีมอนาคต' },
            { value: 'มั่งมีศรีสุข', name: 'มั่งมีศรีสุข', desc: 'ทีมมั่งมีศรีสุข' },
            { value: 'Extra', name: 'Extra', desc: 'ทีมเอ็กซ์ตร้า' }
        ],
        'SaiMai_HS': [
            { value: 'Saimai', name: 'Saimai', desc: 'ทีมสายไหม' }
        ],
        'SaiMai_STP': [
            { value: 'Saimai', name: 'Saimai', desc: 'ทีมสายไหม' }
        ]
    };
    
    const key = `${branch}_${position}`;
    return fallbackData[key] || [];
}
```

### 3. 직원 정보 업데이트 API

#### Endpoint
```http
POST /api/update-employee
```

#### Request Headers
```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {liff-access-token}
```

#### Request Body
```json
{
  "employeeId": "A57250-63",
  "name": "ZIN", 
  "branch": "Asoke",
  "position": "HS",
  "team": "อนาคต",
  "lineUserId": "U1234567890abcdef1234567890abcdef",
  "timestamp": "2025-01-27T10:30:00Z",
  "source": "LIFF_APP"
}
```

#### Request Body Validation
```javascript
// 클라이언트 사이드 검증
function validateSubmitData(data) {
    const required = ['employeeId', 'name', 'branch', 'position', 'team', 'lineUserId'];
    
    for (const field of required) {
        if (!data[field] || typeof data[field] !== 'string') {
            throw new Error(`Missing or invalid field: ${field}`);
        }
    }
    
    // 형식 검증
    if (!/^A57250-\d+$/.test(data.employeeId)) {
        throw new Error('Invalid employee ID format');
    }
    
    if (!/^U[a-f0-9]{32}$/.test(data.lineUserId)) {
        throw new Error('Invalid LINE User ID format');
    }
    
    // 값 검증
    const validBranches = ['Asoke', 'SaiMai'];
    if (!validBranches.includes(data.branch)) {
        throw new Error('Invalid branch');
    }
    
    const validPositions = ['STP', 'HS', 'RECEPTION', 'CLN', 'ADMIN'];
    if (!validPositions.includes(data.position)) {
        throw new Error('Invalid position');
    }
    
    return true;
}
```

#### Response Success (200)
```json
{
  "status": "success",
  "message": "Employee data updated successfully", 
  "data": {
    "employeeId": "A57250-63",
    "name": "ZIN",
    "branch": "Asoke", 
    "position": "HS",
    "team": "อนาคต",
    "lineUserId": "U1234567890abcdef1234567890abcdef",
    "registrationStatus": "completed",
    "updatedAt": "2025-01-27T10:30:00Z"
  },
  "notionPageId": "12345678-1234-1234-1234-123456789012",
  "timestamp": "2025-01-27T10:30:00Z"
}
```

#### 백엔드 처리 — SQL ID(회사프로그램 ID) 자동 채움 [2026-06-03 추가]
n8n "57 Line User ID" 워크플로우(`JedxWlzNZ8JsL2GS`)가 `/api/update-employee` 수신 후 처리하는 흐름:
1. **Find Employee** — `name`으로 Employee Master 페이지 검색
2. **Lookup SQL ID (신규 노드)** — 회사 MSSQL `view_tb_employee`에서 직원의 `zid`(회사프로그램 ID) 조회 (READ-ONLY SELECT)
   - 매칭 키: `znickname` = name, `zbranch` = 지점코드(Asoke→`03`, SaiMai→`04`), `zstatus` = `'Y'`(재직)
   - 쿼리: `SELECT TOP 1 zid FROM view_tb_employee WHERE znickname=$1 AND zbranch=$2 AND zstatus='Y'`
   - credential: `g3rH2M8PYppNVh9I` (57TB Microsoft SQL account, DB `db_57total_thai`)
   - `alwaysOutputData: true` + `onError: continueRegularOutput` — 조회 실패해도 흐름 유지
3. **Update Employee** — Branch/Team/Position/LINE User ID + **SQL ID = 조회된 zid** 함께 저장
   - SQL ID 필드는 `ignoreIfEmpty: true`: 매칭 실패(동명이인 부재·철자 차이) 시 **SQL ID만 빈 채로 두고 나머지는 정상 저장**(graceful). 못 찾은 건 수동 보완.
4. **Line MSM** — 등록 완료 LINE 메시지 발송

> 배경: 이전엔 이 API가 Branch/Team/Position/LINE User ID만 채우고 SQL ID는 비워둬서 매번 수동 입력해야 했음(예: PAOPAO). 이 노드 추가로 신입이 LINE 등록하는 순간 회사프로그램 ID까지 자동으로 채워짐. 라이브 e2e 검증 완료(graceful 경로). 글로벌 메모리 `resigned-staff-line-protection.md` 인접 기록.

#### Response Error (400)
```json
{
  "status": "error",
  "message": "Invalid employee data",
  "code": "VALIDATION_ERROR",
  "errors": [
    {
      "field": "employeeId",
      "message": "Employee ID not found"
    },
    {
      "field": "lineUserId", 
      "message": "LINE User ID already registered"
    }
  ],
  "timestamp": "2025-01-27T10:30:00Z"
}
```

#### Response Error (500)
```json
{
  "status": "error",
  "message": "Internal server error",
  "code": "NOTION_API_ERROR",
  "details": "Failed to update Notion database",
  "timestamp": "2025-01-27T10:30:00Z"
}
```

---

## 🔄 데이터 변환 로직

### 1. LIFF Profile to LINE User ID
```javascript
async function getLINEUserProfile() {
    try {
        const profile = await liff.getProfile();
        return {
            userId: profile.userId,
            displayName: profile.displayName,
            pictureUrl: profile.pictureUrl,
            statusMessage: profile.statusMessage
        };
    } catch (error) {
        console.error('Failed to get LINE profile:', error);
        throw new Error('LINE authentication failed');
    }
}
```

### 2. 팀 자동 할당 로직
```javascript
function getAutoAssignedTeam(branch, position) {
    // 자동 할당 규칙
    const autoAssignRules = {
        // 관리직은 무조건 Manager Team
        'CLN': 'Manager Team',
        'RECEPTION': 'Manager Team', 
        'ADMIN': 'Manager Team',
        
        // SaiMai 지점의 HS/STP는 Saimai 팀
        'SaiMai_HS': 'Saimai',
        'SaiMai_STP': 'Saimai'
    };
    
    // 직급별 우선 확인
    if (autoAssignRules[position]) {
        return autoAssignRules[position];
    }
    
    // 지점+직급 조합 확인
    const key = `${branch}_${position}`;
    if (autoAssignRules[key]) {
        return autoAssignRules[key];
    }
    
    // 자동 할당 불가
    return null;
}
```

### 3. 단계별 진행 로직
```javascript
async function handleStepTransition(currentStep, selectedData) {
    switch(currentStep) {
        case 3: // Position 선택 후
            const autoTeam = getAutoAssignedTeam(selectedData.branch, selectedData.position);
            
            if (autoTeam) {
                selectedData.team = autoTeam;
                return 5; // 확인 단계로 바로 이동
            } else {
                return 4; // 팀 선택 단계로 이동
            }
            
        case 4: // Team 선택 후
            return 5; // 확인 단계로 이동
            
        case 5: // 확인 후
            await submitData(selectedData);
            return 6; // 성공 단계로 이동
            
        default:
            return currentStep + 1;
    }
}
```

### 4. 에러 응답 정규화
```javascript
function normalizeErrorResponse(error) {
    if (error.response) {
        // HTTP 에러 응답
        return {
            type: 'HTTP_ERROR',
            status: error.response.status,
            message: error.response.data?.message || 'Server error',
            code: error.response.data?.code || 'UNKNOWN_ERROR'
        };
    } else if (error.request) {
        // 네트워크 에러
        return {
            type: 'NETWORK_ERROR',
            message: 'Network connection failed',
            code: 'NETWORK_ERROR'
        };
    } else {
        // 기타 에러
        return {
            type: 'CLIENT_ERROR',
            message: error.message || 'Unknown error',
            code: 'CLIENT_ERROR'
        };
    }
}
```

---

## ⚠️ 에러 핸들링

### 1. API 호출 에러 처리
```javascript
async function apiCallWithRetry(url, options, maxRetries = 3) {
    let lastError;
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch(url, {
                ...options,
                timeout: API_CONFIG.TIMEOUT
            });
            
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            
            return await response.json();
            
        } catch (error) {
            lastError = error;
            
            console.warn(`API call attempt ${attempt} failed:`, error.message);
            
            // 마지막 시도가 아니면 대기 후 재시도
            if (attempt < maxRetries) {
                const delay = Math.pow(2, attempt - 1) * 1000; // 지수 백오프
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
    }
    
    throw lastError;
}
```

### 2. LIFF 초기화 에러
```javascript
async function initializeLIFF() {
    const maxRetries = 3;
    let attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            await liff.init({ 
                liffId: API_CONFIG.LIFF_ID,
                withLoginOnExternalBrowser: true
            });
            
            if (!liff.isLoggedIn()) {
                liff.login();
                return;
            }
            
            return true;
            
        } catch (error) {
            attempt++;
            console.error(`LIFF init attempt ${attempt} failed:`, error);
            
            if (attempt === maxRetries) {
                showError('앱을 시작할 수 없습니다. LINE 앱에서 다시 시도해주세요.');
                throw error;
            }
            
            await new Promise(resolve => setTimeout(resolve, 2000));
        }
    }
}
```

### 3. 데이터 검증 에러
```javascript
function validateUserInput(data) {
    const errors = [];
    
    // 필수 필드 검증
    if (!data.employeeId) errors.push('직원 ID가 선택되지 않았습니다.');
    if (!data.name) errors.push('이름이 입력되지 않았습니다.');
    if (!data.branch) errors.push('지점이 선택되지 않았습니다.');
    if (!data.position) errors.push('직급이 선택되지 않았습니다.');
    if (!data.team) errors.push('팀이 선택되지 않았습니다.');
    
    // 형식 검증
    if (data.employeeId && !/^A57250-\d+$/.test(data.employeeId)) {
        errors.push('잘못된 직원 ID 형식입니다.');
    }
    
    if (errors.length > 0) {
        throw new ValidationError(errors);
    }
    
    return true;
}

class ValidationError extends Error {
    constructor(errors) {
        super('Validation failed');
        this.name = 'ValidationError';
        this.errors = errors;
    }
}
```

### 4. 사용자 친화적 에러 메시지
```javascript
function getErrorMessage(error) {
    const errorMessages = {
        'NETWORK_ERROR': '인터넷 연결을 확인해주세요.',
        'VALIDATION_ERROR': '입력 정보를 다시 확인해주세요.',
        'LIFF_INIT_ERROR': 'LINE 앱에서 다시 실행해주세요.',
        'API_TIMEOUT': '서버 응답이 지연되고 있습니다. 잠시 후 다시 시도해주세요.',
        'DUPLICATE_REGISTRATION': '이미 등록된 직원입니다.',
        'EMPLOYEE_NOT_FOUND': '직원 정보를 찾을 수 없습니다.',
        'UNKNOWN_ERROR': '알 수 없는 오류가 발생했습니다.'
    };
    
    return errorMessages[error.code] || errorMessages['UNKNOWN_ERROR'];
}
```

---

## 🔒 보안 및 인증

### 1. LIFF 토큰 검증
```javascript
async function validateLIFFToken() {
    try {
        const accessToken = liff.getAccessToken();
        
        if (!accessToken) {
            throw new Error('No access token available');
        }
        
        // 토큰 유효성 검증 (선택사항)
        const response = await fetch('https://api.line.me/oauth2/v2.1/verify', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded'
            },
            body: `access_token=${accessToken}`
        });
        
        if (!response.ok) {
            throw new Error('Invalid access token');
        }
        
        return accessToken;
        
    } catch (error) {
        console.error('Token validation failed:', error);
        liff.login();
    }
}
```

### 2. 데이터 암호화 (클라이언트)
```javascript
// 민감한 데이터 로깅 시 마스킹
function maskSensitiveData(data) {
    const masked = { ...data };
    
    if (masked.lineUserId) {
        masked.lineUserId = masked.lineUserId.substring(0, 8) + '...';
    }
    
    if (masked.employeeId) {
        masked.employeeId = 'A57250-***';
    }
    
    return masked;
}

// 안전한 로깅
console.log('Submitting data:', maskSensitiveData(submitData));
```

### 3. CSRF 방지
```javascript
// 요청에 타임스탬프 및 nonce 추가
function addSecurityHeaders(data) {
    return {
        ...data,
        timestamp: new Date().toISOString(),
        nonce: generateNonce(),
        source: 'LIFF_APP',
        version: '1.0'
    };
}

function generateNonce() {
    return Math.random().toString(36).substring(2, 15) + 
           Math.random().toString(36).substring(2, 15);
}
```

### 4. Input Sanitization
```javascript
function sanitizeInput(value) {
    if (typeof value !== 'string') return value;
    
    return value
        .trim()
        .replace(/[<>\"']/g, '') // XSS 방지
        .substring(0, 100); // 길이 제한
}

function sanitizeFormData(data) {
    return {
        employeeId: sanitizeInput(data.employeeId),
        name: sanitizeInput(data.name),
        branch: sanitizeInput(data.branch),
        position: sanitizeInput(data.position),
        team: sanitizeInput(data.team),
        lineUserId: data.lineUserId // LINE User ID는 검증됨
    };
}
```

---

## ⚡ 성능 최적화

### 1. API 응답 캐싱
```javascript
const cache = new Map();
const CACHE_DURATION = 5 * 60 * 1000; // 5분

async function cachedApiCall(url, options = {}) {
    const cacheKey = url + JSON.stringify(options);
    const cached = cache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < CACHE_DURATION) {
        console.log('Using cached response for:', url);
        return cached.data;
    }
    
    const data = await apiCallWithRetry(url, options);
    
    cache.set(cacheKey, {
        data,
        timestamp: Date.now()
    });
    
    return data;
}
```

### 2. 지연 로딩
```javascript
// 팀 옵션은 필요할 때만 로드
let teamOptionsCache = null;

async function getTeamOptions(branch, position) {
    const cacheKey = `${branch}_${position}`;
    
    if (teamOptionsCache && teamOptionsCache[cacheKey]) {
        return teamOptionsCache[cacheKey];
    }
    
    const options = await loadTeamOptions(branch, position);
    
    if (!teamOptionsCache) teamOptionsCache = {};
    teamOptionsCache[cacheKey] = options;
    
    return options;
}
```

### 3. 요청 최적화
```javascript
// 병렬 요청 처리
async function preloadData() {
    const promises = [
        cachedApiCall(`${API_CONFIG.BASE_URL}/api/unregistered-employees`),
        // 다른 API 호출들...
    ];
    
    try {
        const [employees, ...others] = await Promise.allSettled(promises);
        
        if (employees.status === 'fulfilled') {
            renderEmployeeList(employees.value);
        }
        
        // 다른 결과 처리...
        
    } catch (error) {
        console.error('Preload failed:', error);
    }
}
```

### 4. DOM 최적화
```javascript
// 가상 스크롤링 (직원 목록이 많은 경우)
function renderEmployeeListVirtual(employees, searchTerm = '') {
    const filteredEmployees = employees.filter(emp => 
        emp.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
    
    const ITEMS_PER_PAGE = 20;
    const currentPage = Math.floor(scrollTop / ITEM_HEIGHT);
    const startIndex = currentPage * ITEMS_PER_PAGE;
    const endIndex = Math.min(startIndex + ITEMS_PER_PAGE, filteredEmployees.length);
    
    const visibleEmployees = filteredEmployees.slice(startIndex, endIndex);
    
    // DOM 업데이트
    const fragment = document.createDocumentFragment();
    visibleEmployees.forEach(emp => {
        fragment.appendChild(createEmployeeElement(emp));
    });
    
    employeeContainer.replaceChildren(fragment);
}
```

이 문서를 통해 57TB LIFF 앱의 모든 데이터 흐름과 API 상호작용을 완전히 이해하고 관리할 수 있습니다.





