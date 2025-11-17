# 🌐 서버리스 Webhook 솔루션 (외부 서버 최소화)

Organization의 많은 레포를 실시간 동기화하되, 관리 부담을 최소화하는 방법입니다.

## 목차
- [핵심 질문](#핵심-질문)
- [Webhook이 필요한 이유](#webhook이-필요한-이유)
- [서버리스 옵션](#서버리스-옵션)
- [권장 솔루션](#권장-솔루션)
- [완전 구현 가이드](#완전-구현-가이드)

---

## 핵심 질문

### Q: Webhook을 외부 서버 없이 받을 수 있나?

**A: 불가능합니다.** ❌

**이유:**
```
GitHub Webhook → HTTP 엔드포인트 필요
GitHub Actions → HTTP 엔드포인트 제공 안 함

따라서:
Webhook을 받으려면 반드시 HTTP 엔드포인트 필요
= 어떤 형태든 "서버" 필요
```

### Q: 그럼 어떻게 해야 하나?

**A: "서버리스" 사용!** ✅

**서버리스 = 서버 없음?**
- ❌ 서버가 아예 없는 건 아님
- ✅ 관리할 서버가 없음
- ✅ 자동으로 스케일링
- ✅ 사용한 만큼만 과금 (거의 무료)
- ✅ 코드만 배포하면 끝

**예시:**
- AWS Lambda
- Vercel Functions
- Cloudflare Workers
- Netlify Functions

→ **사실상 "서버 관리 부담 없음"** 💚

---

## Webhook이 필요한 이유

### Organization의 상황

```
레포가 50개라면?

❌ 각 레포에 workflow 추가:
   - 50개 파일 생성
   - 50개 레포 관리
   - workflow 업데이트 시 50번 수정

✅ Organization Webhook:
   - 1번만 설정
   - 모든 레포 자동 적용
   - 새 레포도 자동
```

**→ Webhook이 필수!**

---

## 서버리스 옵션

---

## 옵션 1: Cloudflare Workers ⭐⭐⭐ (최고 추천!)

### 특징
- ✅ **완전 무료** (월 10만 요청)
- ✅ **초간단 배포** (CLI 한 줄)
- ✅ **글로벌 CDN** (초고속)
- ✅ **관리 불필요**
- ✅ **무제한 확장**

### 코드 (20줄!)

```javascript
// worker.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  if (request.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 })
  }
  
  const webhook = await request.json()
  
  // GitHub Issue 이벤트만 처리
  if (webhook.issue && webhook.action) {
    // 중앙 레포 workflow 트리거
    await fetch(
      'https://api.github.com/repos/junhojang01/issue-sync-test/actions/workflows/action.yml/dispatches',
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${PAT}`,
          'Accept': 'application/vnd.github+json'
        },
        body: JSON.stringify({ ref: 'main' })
      }
    )
  }
  
  return new Response('OK', { status: 200 })
}
```

### 배포 (5분!)

```bash
# 1. Cloudflare 가입 (무료)
# 2. Wrangler 설치
npm install -g wrangler

# 3. 로그인
wrangler login

# 4. 프로젝트 생성
wrangler init notion-sync-webhook

# 5. 코드 복사 (위 코드)
# 6. 배포!
wrangler publish

# 완료! URL 획득:
# https://notion-sync-webhook.your-username.workers.dev
```

### Organization Webhook 설정

```
Organization Settings → Webhooks → Add webhook

Payload URL: https://notion-sync-webhook.your-username.workers.dev
Content type: application/json
Events: ☑ Issues

Active: ✓
```

### 비용
- **완전 무료** (월 10만 요청)
- 이슈 이벤트는 월 1만 건도 안 될 것
- 초과해도 매우 저렴 ($0.50/100만 요청)

### 지연시간
```
이슈 생성 → 즉시 → Webhook → 1초 → Worker → 2초 → Actions → 20초
→ 총 약 23초
```

**거의 실시간!** ⚡

---

## 옵션 2: Vercel Functions ⭐⭐ (간단함)

### 특징
- ✅ **무료** (Hobby 플랜)
- ✅ **쉬운 배포** (git push만)
- ✅ **JavaScript/Python 지원**
- ✅ **자동 HTTPS**

### 코드

```javascript
// api/webhook.js
export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).send('Method not allowed')
  }
  
  const { action, issue, repository } = req.body
  
  if (action && issue) {
    // 중앙 레포 트리거
    await fetch(
      'https://api.github.com/repos/junhojang01/issue-sync-test/actions/workflows/action.yml/dispatches',
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${process.env.PAT}`,
          'Accept': 'application/vnd.github+json'
        },
        body: JSON.stringify({ ref: 'main' })
      }
    )
  }
  
  res.status(200).send('OK')
}
```

### 배포

```bash
# 1. Vercel 가입 (무료)
# 2. 프로젝트 연결
vercel

# 3. 환경 변수 설정
vercel env add PAT

# 완료! URL:
# https://your-project.vercel.app/api/webhook
```

### 비용
- Hobby 플랜: **무료**
- 제한: 월 100GB 대역폭 (충분함)

---

## 옵션 3: AWS Lambda ⭐⭐⭐ (강력함)

### 특징
- ✅ **무료 티어 넉넉** (월 100만 요청)
- ✅ **안정적** (AWS 인프라)
- ✅ **Python 지원**
- ✅ **기업용 확장 가능**

### 코드

```python
# lambda_function.py
import json
import os
import requests

def lambda_handler(event, context):
    # API Gateway에서 전달
    body = json.loads(event['body'])
    
    action = body.get('action')
    issue = body.get('issue')
    
    if action in ['opened', 'edited', 'closed', 'reopened']:
        # 중앙 레포 workflow 트리거
        response = requests.post(
            'https://api.github.com/repos/junhojang01/issue-sync-test/actions/workflows/action.yml/dispatches',
            headers={
                'Authorization': f'Bearer {os.environ["PAT"]}',
                'Accept': 'application/vnd.github+json'
            },
            json={'ref': 'main'}
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps('OK')
    }
```

### 배포

#### GUI 방법 (간단)
1. AWS Console → Lambda
2. Create function
3. 코드 복사-붙여넣기
4. API Gateway 추가
5. 완료!

#### CLI 방법
```bash
# SAM 사용
sam init
sam build
sam deploy
```

### 비용
- **무료 티어:** 월 100만 요청
- **초과 시:** $0.20/100만 요청 (매우 저렴)

---

## 옵션 4: Netlify Functions ⭐

### 특징
- ✅ 무료
- ✅ Git 연동 배포
- ✅ 간단함

### 단점
- ⚠️ 실행 시간 제한 (10초)
- ⚠️ 콜드 스타트 느림

---

## 옵션 5: Google Cloud Run ⭐⭐

### 특징
- ✅ **무료 티어** (월 200만 요청)
- ✅ Docker 지원
- ✅ 자동 스케일링

### 복잡도
- ⚠️ Docker 지식 필요
- ⚠️ 설정 복잡

---

## 🎯 권장 솔루션: Cloudflare Workers

### 왜 Cloudflare Workers인가?

| 항목 | Cloudflare | Vercel | AWS Lambda |
|------|------------|--------|------------|
| **무료 요청** | 10만/일 | 10만/월 | 100만/월 |
| **배포 난이도** | ⭐⭐ | ⭐ | ⭐⭐⭐ |
| **속도** | 매우 빠름 | 빠름 | 보통 |
| **관리** | 없음 | 없음 | 약간 |
| **글로벌** | ✅ | ✅ | ❌ |

**Cloudflare가 최고!** 💚

---

## 완전 구현 가이드

### 🚀 Cloudflare Workers로 실시간 동기화 (30분)

#### 준비물
- Cloudflare 계정 (무료)
- PAT (이미 있음)
- Organization Owner 권한

#### Step 1: Cloudflare 계정 생성 (5분)

1. https://cloudflare.com 접속
2. Sign up (무료)
3. 이메일 인증

#### Step 2: Worker 코드 작성 (10분)

##### 방법 A: Dashboard에서 (GUI)

1. Workers → Create a Service
2. 이름: `notion-sync-webhook`
3. Quick Edit 클릭
4. 코드 붙여넣기:

```javascript
// 완전한 코드
const PAT = 'YOUR_PAT_HERE';  // 환경 변수로 설정 (아래 참고)
const CENTRAL_REPO = 'junhojang01/issue-sync-test';

addEventListener('fetch', event => {
  event.respondWith(handleWebhook(event.request))
})

async function handleWebhook(request) {
  // CORS 처리
  if (request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'POST',
        'Access-Control-Allow-Headers': 'Content-Type'
      }
    })
  }
  
  if (request.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 })
  }
  
  try {
    const webhook = await request.json()
    
    // GitHub Issue 이벤트 확인
    const action = webhook.action
    const issue = webhook.issue
    
    if (!action || !issue) {
      return new Response('Not an issue event', { status: 200 })
    }
    
    // 동기화가 필요한 action만 처리
    const syncActions = ['opened', 'edited', 'closed', 'reopened', 'labeled', 'unlabeled']
    if (!syncActions.includes(action)) {
      return new Response('Action ignored', { status: 200 })
    }
    
    // 중앙 레포의 workflow 트리거
    const response = await fetch(
      `https://api.github.com/repos/${CENTRAL_REPO}/actions/workflows/action.yml/dispatches`,
      {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${PAT}`,
          'Accept': 'application/vnd.github+json',
          'X-GitHub-Api-Version': '2022-11-28'
        },
        body: JSON.stringify({ ref: 'main' })
      }
    )
    
    if (response.ok) {
      console.log(`✓ Triggered sync for ${webhook.repository?.full_name} #${issue.number}`)
      return new Response('Sync triggered', { status: 200 })
    } else {
      const error = await response.text()
      console.error(`✗ Failed to trigger: ${error}`)
      return new Response(`Failed: ${error}`, { status: 500 })
    }
    
  } catch (error) {
    console.error('Error:', error)
    return new Response(`Error: ${error.message}`, { status: 500 })
  }
}
```

5. Save and Deploy

##### 방법 B: CLI로 (권장)

```bash
# 1. Wrangler 설치
npm install -g wrangler

# 2. 로그인
wrangler login

# 3. 프로젝트 생성
mkdir notion-sync-webhook
cd notion-sync-webhook
wrangler init

# 4. src/index.js에 위 코드 복사

# 5. wrangler.toml 수정
name = "notion-sync-webhook"
main = "src/index.js"
compatibility_date = "2024-01-01"

# 6. 환경 변수 설정 (PAT)
wrangler secret put PAT

# 7. 배포!
wrangler publish

# 완료! URL 표시됨:
# https://notion-sync-webhook.your-username.workers.dev
```

#### Step 3: Organization Webhook 설정 (5분)

```
GitHub → Organization Settings → Webhooks → Add webhook

Payload URL: https://notion-sync-webhook.your-username.workers.dev
Content type: application/json
Secret: (선택사항, 보안 강화)
SSL verification: Enable

Which events:
  ☑ Issues
  
Active: ✓
```

#### Step 4: 중앙 레포 수정 (5분)

`.github/workflows/action.yml`에 추가:

```yaml
on:
  issues:
  
  # ✨ 추가
  repository_dispatch:
    types: [webhook_trigger]
  
  schedule:
  workflow_dispatch:
```

#### Step 5: 테스트 (5분)

1. Organization의 아무 레포에서 이슈 생성
2. 20-30초 대기
3. issue-sync-test의 Actions 확인
4. Notion에서 동기화 확인

**성공!** 🎉

---

## 🎨 고급: 직접 Notion 동기화 (더 빠름!)

### Webhook에서 바로 Notion으로

Actions를 거치지 않고 Worker에서 직접 Notion API 호출:

```javascript
// worker.js (고급 버전)
const PAT_GITHUB = 'xxx';
const NOTION_API_KEY = 'secret_xxx';
const NOTION_DATABASE_ID = 'xxx';

async function handleWebhook(request) {
  const webhook = await request.json()
  
  if (webhook.action && webhook.issue) {
    // 직접 Notion API 호출
    await syncToNotion(webhook.issue, webhook.repository)
  }
  
  return new Response('OK', { status: 200 })
}

async function syncToNotion(issue, repository) {
  // 1. 이미 존재하는지 검색
  const existing = await searchNotionPage(issue.number, repository.full_name)
  
  // 2. 생성 또는 업데이트
  if (existing) {
    await updateNotionPage(existing.id, issue, repository)
  } else {
    await createNotionPage(issue, repository)
  }
}

async function createNotionPage(issue, repository) {
  await fetch('https://api.notion.com/v1/pages', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${NOTION_API_KEY}`,
      'Content-Type': 'application/json',
      'Notion-Version': '2022-06-28'
    },
    body: JSON.stringify({
      parent: { database_id: NOTION_DATABASE_ID },
      properties: {
        'Title': { title: [{ text: { content: issue.title } }] },
        'Issue Number': { number: issue.number },
        'Repository': { rich_text: [{ text: { content: repository.full_name } }] },
        // ... 더 추가
      }
    })
  })
}
```

**장점:**
- ✅ **매우 빠름** (5-10초)
- ✅ Actions 사용량 절약
- ✅ 더 간단한 흐름

**단점:**
- ⚠️ Worker 코드가 복잡해짐
- ⚠️ Python 코드(sync_issues.py)를 JavaScript로 재작성 필요

---

## 💰 비용 분석 (Organization 100개 레포)

### 월 10,000개 이슈 변경 가정

| 항목 | 스케줄 | Dispatch (각 레포) | Cloudflare |
|------|--------|-------------------|------------|
| **Webhook 호출** | - | - | 10,000건 |
| **Actions 실행** | 17,280회 | 10,000회 | 10,000회 |
| **Actions 분** | ~52,000분 | ~3,000분 | ~3,000분 |
| **Cloudflare** | - | - | **무료** |
| **총 비용** | 무료* | 무료* | **무료** |

*Private 레포 Actions 무료 한도: 월 2,000분 (개인), 50,000분 (팀)

**Cloudflare로 Actions 사용량 94% 절감!** 🎉

---

## 🔒 보안

### Webhook Secret 검증

```javascript
// Cloudflare Worker
async function verifySignature(request, body) {
  const signature = request.headers.get('X-Hub-Signature-256')
  const secret = WEBHOOK_SECRET
  
  const encoder = new TextEncoder()
  const key = await crypto.subtle.importKey(
    'raw',
    encoder.encode(secret),
    { name: 'HMAC', hash: 'SHA-256' },
    false,
    ['sign']
  )
  
  const signatureBytes = await crypto.subtle.sign(
    'HMAC',
    key,
    encoder.encode(body)
  )
  
  const expectedSignature = 'sha256=' + 
    Array.from(new Uint8Array(signatureBytes))
      .map(b => b.toString(16).padStart(2, '0'))
      .join('')
  
  return signature === expectedSignature
}
```

**항상 Webhook Secret 검증하세요!** 🔐

---

## 🎯 완전 무료 구성

```
┌─────────────────────────────────────────┐
│ Organization (50개 레포)                │
│  - backend-api                          │
│  - frontend-web                         │
│  - mobile-app                           │
│  - ... (47개 더)                        │
└──────────────┬──────────────────────────┘
               │ Issue 이벤트
               ↓
┌─────────────────────────────────────────┐
│ Organization Webhook (무료)             │
└──────────────┬──────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────┐
│ Cloudflare Worker (무료)                │
│  - 20줄 코드                            │
│  - 자동 스케일링                        │
│  - 글로벌 CDN                           │
└──────────────┬──────────────────────────┘
               │ repository_dispatch
               ↓
┌─────────────────────────────────────────┐
│ issue-sync-test (중앙 레포)             │
│  - GitHub Actions (무료)                │
│  - sync_issues.py                       │
└──────────────┬──────────────────────────┘
               │
               ↓
┌─────────────────────────────────────────┐
│ Notion Database                         │
│  - 모든 레포 이슈                       │
│  - Projects 정보                        │
│  - 실시간 동기화 (20초)                 │
└─────────────────────────────────────────┘
```

**총 비용: $0/월** 💚  
**관리 시간: 월 0시간** 💚  
**지연 시간: ~20초** ⚡

---

## 📋 구현 체크리스트

### Cloudflare Workers 방식

- [ ] Cloudflare 계정 생성 (5분)
- [ ] Worker 코드 작성 (10분)
- [ ] 환경 변수 설정 (PAT)
- [ ] 배포 (wrangler publish)
- [ ] Worker URL 확인
- [ ] Organization Webhook 설정
  - [ ] Payload URL: Worker URL
  - [ ] Events: Issues
  - [ ] Active 체크
- [ ] 중앙 레포에 repository_dispatch 추가
- [ ] 테스트 (이슈 생성)
- [ ] Notion 확인

**총 소요: 30분**

---

## 🐛 문제 해결

### Worker가 트리거되지 않음

**확인:**
1. Organization Webhook에서 "Recent Deliveries" 확인
2. Worker URL이 정확한지
3. Worker Logs 확인

### Actions가 트리거되지 않음

**확인:**
1. Worker 코드의 repository_dispatch URL 확인
2. PAT 권한 (`workflow` scope)
3. 중앙 레포의 on: repository_dispatch 설정

### "Bad credentials" 에러

**확인:**
1. Worker 환경 변수의 PAT 확인
2. PAT 만료되지 않았는지
3. workflow scope 있는지

---

## 🎉 결론

### 외부 "서버" 필요? 

**기술적으로: YES** ✅  
**실질적으로: NO** ✅

**이유:**
- Cloudflare Workers = 서버리스
- 코드만 배포
- 관리 불필요
- 완전 무료
- 사실상 "서버 없음"과 동일!

### Organization 50개 레포 → 1번만 설정

```
❌ Repository Dispatch:
   - 50개 workflow 파일 관리

✅ Cloudflare + Org Webhook:
   - Worker 1개 (20줄)
   - Webhook 1번 설정
   - 끝!
```

---

## 💡 추천

### 지금 당장 (개인 레포 2개)
- 15분 스케줄 유지 (충분함)
- 또는 Repository Dispatch (간단)

### Organization 확장 시 (즉시!)
- **Cloudflare Workers + Org Webhook** ⭐⭐⭐
- 30분이면 구현
- 완전 무료
- 관리 불필요
- 실시간 (~20초)

---

**Cloudflare Workers 구현해드릴까요?** 🚀

1. Worker 코드 제공
2. 배포 가이드
3. Organization Webhook 설정 가이드
4. 테스트

30분이면 완성입니다!

