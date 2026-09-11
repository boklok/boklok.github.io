---
layout: post
title: "Okta PKCE 검증 랩 실습 가이드 - OAuth 2.0 보안 인증 구현"
date: 2026-09-10
categories: [Okta, OAuth, Security, OIDC]
tags: [okta, pkce, oauth2.0, oidc, security-engineer, authentication]
---

# Okta PKCE 검증 랩 실습 가이드

**RFC 7636 PKCE(Proof Key for Code Exchange)** 를 Okta에서 직접 검증하고, OAuth 2.0 Authorization Code Flow의 보안 메커니즘을 이해하는 실습 가이드입니다.

##  개요

이 가이드는 다음을 다룹니다:
- Okta Integrator Free Plan을 사용한 OIDC 앱 구성
- 3계층 액세스 제어(App Assignment, Sign-On Policy, Authorization Server Policy) 설정
- PKCE 필수 정책 적용 및 검증
- 실제 Authorization Code 흐름 테스트

---

##  사전 준비

### 필수 항목
1. **Okta 계정**: Okta Integrator Free Plan (무료, 만료 없음)
   - [Okta Integrator Free Plan 가입](https://developer.okta.com/signup)
2. **Node.js**: v18+ (로컬 테스트 서버용)
3. **브라우저**: Chrome, Edge 등 (Okta 로그인용)
4. **테스트 사용자**: Okta에서 최소 1개 생성

### 선택 사항
- Postman (API 테스트용)
- curl 또는 PowerShell (요청 테스트용)

---

## 단계 1: Okta 테넌트 기본 설정

### 1-1. 테스트 사용자 생성

```
Admin Console > Directory > People > Add Person
```

**입력값:**
- First name: `Test`
- Last name: `User`
- Email: `test+lab@gmail.com` (또는 본인 이메일 + 태그)
- Username: 자동 입력됨
- Password: 본인이 설정 (예: `Test123!@#`)
- Password 설정 방식: Set by admin
```
☑ User must change password on first login (선택 사항)
```

**저장 후:**
- 사용자가 생성됨
- Active 상태 확인

---

## 단계 2: OIDC 앱 생성 및 기본 설정

### 2-1. 새 OIDC 앱 생성

```
Admin Console > Applications and Resources > Applications > Create App Integration
```

**선택:**
- **Sign in method**: OIDC - OpenID Connect
- **Application type**: Native Application (또는 Single-Page App - SPA)
- **Create** 클릭

### 2-2. 앱 이름 및 기본 정보

**입력:**
```
App name: oidc pkce lab
Logo: (선택 사항)
```

**클릭:** Save

### 2-3. General 탭 - 클라이언트 설정

앱이 생성되면 **General** 탭에서:

**Client Credentials 섹션:**
- **Client ID**: 복사해서 저장 (예: `0oa17deepqlJFDpcG698`)
- **Client authentication**: `None` (선택됨 - 변경 금지)

**Proof Key for Code Exchange (PKCE):**
```
☑ Require PKCE as additional verification
```

**저장:** Save

### 2-4. Sign-in redirect URIs 설정

**Edit** 클릭 → **GENERAL** 섹션

**Sign-in redirect URIs:**
다음 3개 추가:
1. `http://localhost:8080/callback`
2. `http://localhost:5000/callback/okta-oidc`
3. `com.okta.integrator-5935493:/callback` (모바일용)

**Save** 클릭

---

## 단계 3: 3계층 액세스 제어 설정

### 3-1. 사용자 그룹 생성

```
Admin Console > Directory > Groups > Create Group
```

**입력:**
```
Name: dept-engineering
Description: Engineering department users for PKCE lab
```

**Save** → 테스트 사용자를 이 그룹에 할당

### 3-2. Sign-On Policy 설정

**oidc pkce lab 앱의 Sign On 탭:**

```
User authentication > Authentication policy
```

**새 정책 생성 (권장):**
```
Lab - Password Only
```

이 정책은 MFA 없이 비밀번호만으로 로그인을 허용합니다 (PKCE 메커니즘 테스트 목적).

**저장:** 이 정책을 앱에 적용

### 3-3. Authorization Server 액세스 정책 설정

```
Security > API > Authorization Servers > default > Access Policies
```

**새 정책 생성:**
```
Name: OIDC PKCE Lab Policy
Clients: All Clients
```

**규칙 추가:**
```
Rule Name: Allow Authorization Code for assigned users

IF:
  - Grant type is: ✓ Authorization Code
  - Scope requested: openid profile email offline_access

THEN:
  - Access is: Allow
```

**Save**

---

## 단계 4: PKCE 검증 테스트

### 4-1. 로컬 테스트 서버 생성 (Node.js)

**파일: `pkce-test.js`**

```javascript
const crypto = require('crypto');
const http = require('http');
const url = require('url');

const OKTA_DOMAIN = 'integrator-5935493.okta.com'; // 본인 도메인으로 변경
const CLIENT_ID = '0oa17deepqlJFDpcG698'; // 본인 Client ID로 변경
const REDIRECT_URI = 'http://localhost:8080/callback';

function generatePKCE() {
  const codeVerifier = crypto.randomBytes(32).toString('hex');
  const codeChallenge = crypto
    .createHash('sha256')
    .update(codeVerifier)
    .digest('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
  
  return { codeVerifier, codeChallenge };
}

const server = http.createServer((req, res) => {
  const parsedUrl = url.parse(req.url, true);
  
  if (parsedUrl.pathname === '/callback') {
    const code = parsedUrl.query.code;
    const error = parsedUrl.query.error;
    
    if (error) {
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
      res.end(`<h2> 에러 발생</h2><p>Error: ${error}</p><p>Description: ${parsedUrl.query.error_description || 'N/A'}</p>`);
    } else if (code) {
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
      res.end(`<h2> 성공!</h2><p>Authorization Code: <code>${code}</code></p><p>이 코드를 저장하세요 (다음 단계에서 사용)</p>`);
    } else {
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
      res.end('<h2>콜백 수신됨</h2>');
    }
  } else {
    res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
    res.end(`<h2>PKCE 테스트 서버</h2><p>http://localhost:8080/callback 대기 중...</p>`);
  }
});

server.listen(8080, () => {
  const { codeVerifier, codeChallenge } = generatePKCE();
  
  console.log(' 로컬 서버 시작: http://localhost:8080');
  console.log('\n📋 PKCE 값:');
  console.log(`  Code Verifier: ${codeVerifier}`);
  console.log(`  Code Challenge: ${codeChallenge}`);
  
  const authUrl = `https://${OKTA_DOMAIN}/oauth2/default/v1/authorize?` +
    `client_id=${CLIENT_ID}&` +
    `response_type=code&` +
    `scope=openid%20profile%20email%20offline_access&` +
    `redirect_uri=${encodeURIComponent(REDIRECT_URI)}&` +
    `state=abc123&` +
    `nonce=xyz789&` +
    `code_challenge=${codeChallenge}&` +
    `code_challenge_method=S256`;
  
  console.log('\n🔗 아래 URL을 브라우저에서 열어주세요:');
  console.log(authUrl);
  console.log('\n 팁: 로컬 서버가 실행 중이면 자동으로 콜백을 처리합니다.');
});
```

### 4-2. 서버 실행

```bash
node pkce-test.js
```

**출력:**
```
 로컬 서버 시작: http://localhost:8080

📋 PKCE 값:
  Code Verifier: 7fe63c6ba96e0cccf3a83b3d191400330f81dff89c62947bb4a0f2b34a7494d1
  Code Challenge: lKqGMR-dD7pzAECV2hmCdRGaKxMHLb_91WvDPWI1mU0

🔗 아래 URL을 브라우저에서 열어주세요:
https://integrator-5935493.okta.com/oauth2/default/v1/authorize?...
```

### 4-3. Authorization Code 획득

1. **터미널의 Authorization URL 복사**
2. **브라우저에서 새 탭 열기**
3. **주소창에 URL 붙여넣기** → Enter
4. **Okta 로그인 페이지** 나타남
5. **테스트 사용자 정보로 로그인**
   - Email: `test+lab@gmail.com`
   - Password: (설정한 비밀번호)
6. **콜백 페이지** 나타남 → **Authorization Code 표시됨**
   - Code를 복사해서 저장

---

## 단계 5: PKCE 없이 요청 테스트 (400 에러 확인)

PKCE 필수 정책을 검증하기 위해 `code_challenge` 없이 요청을 보냅니다:

```bash
curl "https://integrator-5935493.okta.com/oauth2/default/v1/authorize?
client_id=0oa17deepqlJFDpcG698&
response_type=code&
scope=openid%20profile%20email%20offline_access&
redirect_uri=http://localhost:8080/callback&
state=abc123"
```

**예상 결과:**
```
HTTP 400 Invalid Request
error: invalid_request
error_description: PKCE is required
```

**Okta의 동작:**
- `/authorize` 엔드포인트에서 즉시 400 에러 반환
- 로그인 화면 이전에 검증  (ADFS/Entra ID는 토큰 교환 단계에서 거부)

---

## 단계 6: Token 교환 (Code → Access Token)

**Authorization Code를 획득한 후:**

### Token 엔드포인트 요청

```bash
curl -X POST "https://integrator-5935493.okta.com/oauth2/default/v1/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "client_id=0oa17deepqlJFDpcG698" \
  -d "code=aBc123..." \
  -d "code_verifier=7fe63c6ba96e0cccf3a83b3d191400330f81dff89c62947bb4a0f2b34a7494d1" \
  -d "redirect_uri=http://localhost:8080/callback"
```

**응답 (성공):**
```json
{
  "access_token": "eyJhbGc...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "OART...",
  "id_token": "eyJhbGc...",
  "scope": "openid profile email offline_access"
}
```

---

## 📚 주요 학습 포인트

### PKCE의 역할

| 항목 | 설명 |
|------|------|
| **코드 검증자** | 클라이언트가 생성한 난수 (code_verifier) |
| **코드 챌린지** | 검증자의 SHA256 해시 (code_challenge) |
| **메서드** | S256 (SHA256) 또는 plain |
| **보안 이점** | Authorization Code 탈취 공격 방지 |

### Okta의 PKCE 검증 시점

1. **Authorization Request** (`/authorize`)
   - `code_challenge` 포함 여부 확인
   - 없으면 즉시 400 에러
   
2. **Token Request** (`/token`)
   - `code_verifier`와 저장된 `code_challenge` 비교
   - 불일치하면 토큰 거부

### 3계층 액세스 제어 구조

```
┌─────────────────────────────┐
│  1️ App Assignment          │
│  (dept-engineering 그룹)     │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│  2️ Sign-On Policy          │
│  (Lab - Password Only)       │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│  3️ Authorization Server     │
│  (OIDC PKCE Lab Policy)     │
└─────────────────────────────┘
```

---

## 🐛 문제 해결

### 404 에러: Page Not Found

**원인:** Okta 도메인이 잘못됨

**해결:**
```
**실패** https://integrator-5935493-admin.okta.com/oauth2/v1/authorize
**성공** https://integrator-5935493.okta.com/oauth2/default/v1/authorize
```

- `-admin` 제거
- `/oauth2/v1/` → `/oauth2/default/v1/`

### Redirect URI 불일치

**확인:**
```
Apps > oidc pkce lab > General > Sign-in redirect URIs
```

다음이 포함되어 있는지 확인:
- `http://localhost:8080/callback`

### 비밀번호 재설정

```
Directory > People > test+lab@gmail.com > Set Password
```

---

##  보안 권장사항

1. **PKCE 필수화** - 모든 OAuth 클라이언트에 적용
2. **State 파라미터** - CSRF 공격 방지
3. **HTTPS 사용** - Redirect URI는 HTTPS (localhost 제외)
4. **Scope 제한** - 필요한 권한만 요청
5. **Token 만료** - 짧은 access token, 긴 refresh token

---

##  참고 자료

- [RFC 7636 - PKCE](https://tools.ietf.org/html/rfc7636)
- [Okta OAuth 2.0 문서](https://developer.okta.com/docs/guides/implement-oauth-for-okta/)
- [OIDC Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [Okta Authorization Server](https://developer.okta.com/docs/guides/customize-authz-server/)

---

##  피드백

이 가이드에서 개선할 점이 있으면 댓글로 알려주세요!

**작성일:** 2026-09-10  
**업데이트:** Okta Integrator Free Plan, PKCE S256 검증
