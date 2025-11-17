# ⚡ 실시간 동기화 구현 방안 조사

현재 15분 스케줄 동기화의 한계를 극복하기 위한 실시간 동기화 방법들을 분석합니다.

## 목차
- [현재 문제점](#현재-문제점)
- [실시간 동기화 옵션](#실시간-동기화-옵션)
- [옵션별 상세 분석](#옵션별-상세-분석)
- [권장 방안](#권장-방안)
- [구현 로드맵](#구현-로드맵)

---

## 현재 문제점

### 스케줄 기반 동기화의 한계

```yaml
schedule:
  - cron: '*/15 * * * *'  # 15분마다
```

**문제:**
- ⏰ 최대 15분 지연 (이슈 생성 → 동기화)
- 📊 불필요한 API 호출 (변경 없어도 실행)
- 🔋 GitHub Actions 사용량 낭비
- 🚫 진짜 실시간이 아님

**요구사항:**
- ✅ 이슈 생성/수정 즉시 동기화 (몇 초 이내)
- ✅ 모든 레포에서 작동
- ✅ Organization 확장 가능

---

## 실시간 동기화 옵션

| 옵션 | 지연시간 | 난이도 | 비용 | 확장성 | 추천 |
|------|----------|--------|------|--------|------|
| **1. Repository Dispatch** | ~30초 | ⭐⭐ 보통 | 무료 | ⭐⭐⭐ | ✅ |
| **2. Organization Webhook** | ~10초 | ⭐⭐⭐ 고급 | 무료 | ⭐⭐⭐⭐ | ✅✅ |
| **3. GitHub App** | ~5초 | ⭐⭐⭐⭐ 매우 고급 | 무료 | ⭐⭐⭐⭐⭐ | 💎 |
| **4. 외부 서버** | ~5초 | ⭐⭐⭐ 고급 | 유료 | ⭐⭐⭐⭐⭐ | 💰 |
| **5. Webhook Relay** | ~10초 | ⭐⭐ 보통 | 유료 | ⭐⭐⭐⭐ | 💰 |

---

## 옵션별 상세 분석

---

## 옵션 1: Repository Dispatch ⭐ (가장 현실적)

### 개념

```
다른 레포의 이슈 이벤트
  ↓
작은 workflow (각 레포)
  ↓
repository_dispatch API 호출
  ↓
중앙 레포(issue-sync)의 workflow 트리거
  ↓
동기화 실행
```

### 장점
- ✅ **무료** (GitHub Actions 범위 내)
- ✅ **설정 간단** (각 레포에 작은 workflow만 추가)
- ✅ **즉시 반응** (~30초)
- ✅ **확장 쉬움** (레포마다 복사-붙여넣기)

### 단점
- ⚠️ 각 레포에 workflow 파일 추가 필요
- ⚠️ PAT 필요 (repository_dispatch 권한)
- ⚠️ 레포가 많으면 관리 복잡

### 구현 방법

#### 1. 중앙 레포(issue-sync-test)에 트리거 추가

```yaml
# .github/workflows/action.yml
on:
  issues:  # 이 레포의 이슈
  repository_dispatch:  # ← 추가!
    types: [issue_updated]
  schedule:
  workflow_dispatch:
```

#### 2. 다른 레포에 트리거 workflow 추가

**각 레포:** `.github/workflows/trigger-sync.yml`

```yaml
name: Trigger Issue Sync

on:
  issues:
    types: [opened, edited, closed, reopened, labeled, unlabeled]

jobs:
  trigger-central-sync:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger central sync repo
        run: |
          curl -X POST \
            -H "Accept: application/vnd.github.v3+json" \
            -H "Authorization: token ${{ secrets.PAT_GITHUB }}" \
            https://api.github.com/repos/junhojang01/issue-sync-test/actions/workflows/action.yml/dispatches \
            -d '{"ref":"main","inputs":{}}'
```

#### 3. PAT 권한 확인

```
PAT 권한에 추가:
- actions: write (또는 workflow scope)
```

### 지연시간
```
이슈 생성 → 10초 → trigger workflow 실행 → 10초 → 중앙 sync
→ 총 약 20-30초
```

### 비용
- **무료** (Actions 무료 범위 내)
- Public 레포: 무제한
- Private 레포: 월 2000분 (개인 계정)

---

## 옵션 2: Organization Webhook 🏢 (가장 강력)

### 개념

```
Organization 레벨 Webhook 설정
  ↓
모든 레포의 이슈 이벤트 수신
  ↓
Webhook → GitHub Actions (workflow_dispatch)
  또는
Webhook → 외부 서버 → GitHub Actions
  ↓
동기화 실행
```

### 장점
- ✅ **한 번만 설정** (Organization 레벨)
- ✅ **모든 레포 자동 적용** (새 레포도 자동)
- ✅ **매우 빠름** (~10초)
- ✅ **관리 쉬움** (한 곳에서 관리)

### 단점
- ⚠️ **Organization 권한 필요** (Owner/Admin)
- ⚠️ **Webhook 수신 서버 필요** (또는 우회 방법)
- ⚠️ 개인 레포에서는 불가능

### 구현 방법 (2가지)

#### 방법 A: Webhook → 외부 서버 → Actions

```
Organization Webhook
  ↓
AWS Lambda / Cloud Run (무료 티어)
  ↓
repository_dispatch API 호출
  ↓
issue-sync-test workflow 실행
```

**필요:**
- 외부 서버 (Lambda, Cloud Run, Vercel 등)
- 간단한 코드 (webhook 받아서 API 호출)

#### 방법 B: GitHub App 사용 (옵션 3과 유사)

### 지연시간
```
이슈 생성 → 즉시 → Webhook → 5초 → 서버 → 5초 → Actions
→ 총 약 10-15초
```

### 비용
- Organization Webhook: **무료**
- 외부 서버: 
  - AWS Lambda: 월 100만 요청 **무료**
  - Vercel: Hobby 플랜 **무료**
  - Cloud Run: 월 200만 요청 **무료**

---

## 옵션 3: GitHub App 💎 (최고급)

### 개념

```
GitHub App 생성
  ↓
Organization에 설치
  ↓
모든 레포의 이벤트를 App으로 전달
  ↓
App 서버에서 처리 또는 Actions 트리거
  ↓
동기화 실행
```

### 장점
- ✅ **가장 강력함** (GitHub 공식 방법)
- ✅ **세밀한 권한 제어**
- ✅ **Organization 모든 레포 자동 적용**
- ✅ **매우 빠름** (~5초)
- ✅ **확장성 최고**

### 단점
- ⚠️ **구현 복잡** (서버 코드 필요)
- ⚠️ **서버 호스팅 필요**
- ⚠️ **유지보수 필요**

### 구현 방법

#### 1. GitHub App 생성

```
Settings → Developer settings → GitHub Apps → New GitHub App

Webhook:
  - Webhook URL: https://your-server.com/webhook
  - Events: Issues

Permissions:
  - Issues: Read & write
  - Contents: Read-only
  - Projects: Read-only
```

#### 2. 서버 구현 (Python Flask 예시)

```python
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def webhook():
    event = request.json
    
    if event['action'] in ['opened', 'edited', 'closed']:
        # repository_dispatch 트리거
        requests.post(
            'https://api.github.com/repos/junhojang01/issue-sync-test/dispatches',
            headers={'Authorization': f'Bearer {PAT}'},
            json={'event_type': 'issue_updated'}
        )
    
    return 'OK', 200
```

#### 3. 서버 배포

- Vercel (무료)
- Heroku (무료 티어)
- AWS Lambda (무료 티어)
- Cloud Run (무료 티어)

### 지연시간
```
이슈 생성 → 즉시 → App → 3초 → Actions
→ 총 약 5-10초
```

### 비용
- GitHub App: **무료**
- 서버: **무료 티어 충분** (트래픽 적음)

---

## 옵션 4: 외부 서버 + Webhook 🖥️

### 개념

```
각 레포 Webhook 설정
  ↓
외부 서버 (Lambda, Cloud Run)
  ↓
직접 Notion API 호출 또는 GitHub Actions 트리거
  ↓
동기화 완료
```

### 장점
- ✅ **완전한 제어**
- ✅ **매우 빠름** (~5초)
- ✅ **복잡한 로직 가능**
- ✅ **스케일링 쉬움**

### 단점
- ⚠️ **서버 관리 필요**
- ⚠️ **코드 복잡**
- ⚠️ **각 레포에 Webhook 설정**

### 구현 방법

#### 1. 서버리스 함수 (AWS Lambda)

```python
import json
import requests
import boto3

def lambda_handler(event, context):
    # GitHub Webhook 데이터
    webhook_data = json.loads(event['body'])
    
    issue = webhook_data['issue']
    repo = webhook_data['repository']['full_name']
    
    # 직접 Notion API 호출
    sync_to_notion(issue, repo)
    
    return {
        'statusCode': 200,
        'body': 'OK'
    }

def sync_to_notion(issue, repo):
    # sync_issues.py의 로직을 여기에 구현
    pass
```

#### 2. 각 레포에 Webhook 설정

```
Repository → Settings → Webhooks → Add webhook

Payload URL: https://your-lambda-url.amazonaws.com/webhook
Content type: application/json
Events: Issues
```

### 지연시간
```
이슈 생성 → 즉시 → Webhook → 2초 → Lambda → 3초 → Notion
→ 총 약 5초
```

### 비용
- AWS Lambda: 월 100만 요청 **무료**
- API Gateway: 월 100만 요청 **무료**
- 그 이상: 매우 저렴 (~$1/100만 요청)

---

## 옵션 5: Webhook Relay 서비스 🔄

### 개념

```
GitHub Webhook
  ↓
Webhook Relay (서드파티)
  ↓
GitHub Actions 트리거
  ↓
동기화 실행
```

### 서비스 예시
- **Smee.io** (무료, 간단)
- **Hookdeck** (무료 티어)
- **Webhook.site** (테스트용)

### 장점
- ✅ **서버 불필요**
- ✅ **설정 간단**
- ✅ **빠름** (~10초)

### 단점
- ⚠️ **서드파티 의존**
- ⚠️ **보안 우려** (webhook 데이터 노출)
- ⚠️ **무료 티어 제한**

### 구현 방법 (Smee.io)

#### 1. Smee 채널 생성

```
https://smee.io → Start a new channel
→ URL 복사: https://smee.io/abc123
```

#### 2. 각 레포 Webhook 설정

```
Payload URL: https://smee.io/abc123
Events: Issues
```

#### 3. Smee client 실행 (서버 또는 로컬)

```bash
npm install -g smee-client
smee -u https://smee.io/abc123 -t http://your-actions-trigger
```

### 지연시간
```
이슈 생성 → 즉시 → Smee → 5초 → Actions
→ 총 약 10초
```

### 비용
- Smee.io: **무료** (제한적)
- Hookdeck: 무료 티어 (월 10만 이벤트)

---

## 옵션별 비교 분석

---

## 📊 옵션 1: Repository Dispatch (권장 ⭐)

### 아키텍처

```
┌─────────────────┐
│  backend-api    │ Issue 생성
│                 │    ↓
│  workflow:      │ .github/workflows/trigger-sync.yml 실행
│  trigger-sync   │    ↓
└─────────────────┘ curl로 API 호출
         │
         ↓ repository_dispatch
         
┌─────────────────┐
│ issue-sync-test │ on: repository_dispatch 트리거
│                 │    ↓
│  workflow:      │ sync_issues.py 실행
│  action.yml     │    ↓
└─────────────────┘ 모든 레포 동기화
```

### 구현 상세

#### 중앙 레포 수정

**파일:** `.github/workflows/action.yml`

```yaml
on:
  issues:
    types: [opened, edited, closed, reopened, labeled, unlabeled]
  
  # ✨ 추가!
  repository_dispatch:
    types: [issue_updated, issue_created, issue_closed]
  
  schedule:
    - cron: '*/15 * * * *'  # 백업용으로 유지
  
  workflow_dispatch:
```

#### 각 레포에 추가

**파일:** `.github/workflows/trigger-sync.yml` (새로 생성)

```yaml
name: Trigger Central Issue Sync

on:
  issues:
    types: [opened, edited, closed, reopened, labeled, unlabeled, assigned]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger sync in central repo
        run: |
          curl -X POST \
            -H "Accept: application/vnd.github.v3+json" \
            -H "Authorization: token ${{ secrets.PAT_GITHUB }}" \
            https://api.github.com/repos/junhojang01/issue-sync-test/actions/workflows/action.yml/dispatches \
            -d '{"ref":"main"}'
```

#### 필요 권한

PAT에 추가:
```
Workflows: Read and write
(또는 Classic Token의 workflow scope)
```

### 장점 상세
- ✅ 코드 간단 (curl 한 줄)
- ✅ Actions 무료 범위 충분
- ✅ 안정적 (GitHub 인프라 사용)
- ✅ 디버깅 쉬움 (Actions 로그)

### 단점 상세
- ⚠️ 레포가 10개면 10개에 workflow 추가
- ⚠️ workflow 업데이트 시 모든 레포 수정

### 예상 흐름

```
1. deeplink-test에 이슈 생성
   ↓ (즉시)
2. trigger-sync.yml 실행 (10초)
   ↓
3. issue-sync-test의 action.yml 트리거 (5초)
   ↓
4. sync_issues.py 실행 (20초)
   ↓
5. Notion 동기화 완료

총 소요: 약 35초
```

---

## 📊 옵션 2: Organization Webhook (최고 ⭐⭐)

### 아키텍처

```
Organization 설정
  ↓
모든 레포의 Issue 이벤트
  ↓
Organization Webhook
  ↓
외부 서버 (Lambda/Vercel)
  ↓
repository_dispatch 또는 직접 동기화
  ↓
Notion 업데이트
```

### 구현 상세

#### 1. Organization Webhook 설정

```
Organization Settings → Webhooks → Add webhook

Payload URL: https://your-lambda-url.com/webhook
Content type: application/json
Secret: [생성]

Events:
  ☑ Issues
  ☑ Projects V2 (선택사항)

Active: ✓
```

#### 2. Lambda 함수 (Python)

```python
import json
import hmac
import hashlib
import requests
import os

def lambda_handler(event, context):
    # 1. Webhook 서명 검증
    signature = event['headers']['X-Hub-Signature-256']
    body = event['body']
    secret = os.environ['WEBHOOK_SECRET']
    
    expected = hmac.new(
        secret.encode(),
        body.encode(),
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(f'sha256={expected}', signature):
        return {'statusCode': 401, 'body': 'Invalid signature'}
    
    # 2. 이벤트 파싱
    webhook_data = json.loads(body)
    action = webhook_data.get('action')
    issue = webhook_data.get('issue')
    repo = webhook_data.get('repository', {}).get('full_name')
    
    if action in ['opened', 'edited', 'closed', 'reopened']:
        # 옵션 A: repository_dispatch 트리거
        trigger_sync_workflow(repo)
        
        # 옵션 B: 직접 Notion 동기화
        # sync_to_notion_directly(issue, repo)
    
    return {'statusCode': 200, 'body': 'OK'}

def trigger_sync_workflow(repo):
    """중앙 레포 workflow 트리거"""
    requests.post(
        'https://api.github.com/repos/junhojang01/issue-sync-test/actions/workflows/action.yml/dispatches',
        headers={
            'Authorization': f'Bearer {os.environ["PAT"]}',
            'Accept': 'application/vnd.github.v3+json'
        },
        json={'ref': 'main'}
    )
```

#### 3. Lambda 배포

```bash
# serverless framework 사용
serverless deploy

# 또는 AWS Console에서 수동 배포
```

### 장점 상세
- ✅ 한 번만 설정 (모든 레포 적용)
- ✅ 새 레포 자동 포함
- ✅ 매우 빠름
- ✅ 확장성 최고

### 단점 상세
- ⚠️ Organization Owner 권한 필요
- ⚠️ 서버 코드 작성/관리
- ⚠️ Webhook 보안 설정

### 지연시간
```
이슈 생성 → 즉시 → Webhook → 2초 → Lambda → 3초 → Actions → 20초
→ 총 약 25초

또는 직접 동기화:
이슈 생성 → 즉시 → Webhook → 2초 → Lambda → 3초 → Notion
→ 총 약 5초
```

### 비용
- Organization Webhook: **무료**
- AWS Lambda: **무료 티어로 충분**
  - 월 100만 요청 무료
  - 이슈 이벤트는 적음 (월 수백~수천 건)

---

## 📊 옵션 3B: GitHub App (완전 자동화)

### 더 나아가기

GitHub App을 만들면 **Marketplace에 배포** 가능:

```
GitHub App: "Notion Issue Sync"
  ↓
다른 사람들도 설치 가능
  ↓
Organization에 한 번 설치
  ↓
모든 레포 자동 동기화
```

**예시:** Slack, Jira 같은 통합

---

## 📊 옵션 4: 외부 서버 직접 운영

### 개념

```
외부 서버 (24/7 실행)
  ↓
GitHub Webhook 수신
  ↓
직접 Notion API 호출
  ↓
동기화 완료
```

### 장점
- ✅ 완전한 제어
- ✅ 가장 빠름 (~3초)
- ✅ 복잡한 로직 가능
- ✅ 실시간 처리

### 단점
- ⚠️ 서버 관리
- ⚠️ 비용 발생 (서버 호스팅)
- ⚠️ 모니터링/로깅 필요

### 비용
- Vercel/Netlify: **무료** (Serverless Functions)
- AWS EC2: $5-10/월 (t3.micro)
- DigitalOcean: $5/월
- Cloud Run: **무료 티어** (충분)

---

## 🎯 권장 방안

### 현재 단계: 개인 레포 테스트

#### 추천: **옵션 1 (Repository Dispatch)**

**이유:**
- ✅ 구현 간단 (workflow 파일만)
- ✅ 무료
- ✅ 충분히 빠름 (30초)
- ✅ 개인 레포에서 바로 사용 가능

**단점:**
- 레포 2개니까 2개에 workflow 추가 (감당 가능)

---

### 향후 단계: Organization 확장

#### 추천: **옵션 2 (Organization Webhook + Lambda)**

**이유:**
- ✅ 한 번만 설정
- ✅ 모든 레포 자동 적용
- ✅ 새 레포도 자동
- ✅ 거의 무료 (Lambda 무료 티어)
- ✅ 매우 빠름 (10초)

**구현:**
1. 간단한 Lambda 함수 (20줄)
2. Organization Webhook 설정
3. 끝!

---

### 장기: 제품화

#### 추천: **옵션 3 (GitHub App)**

**이유:**
- ✅ Marketplace 배포 가능
- ✅ 다른 팀도 사용
- ✅ 가장 전문적
- ✅ 확장성 최고

---

## 📋 단계별 로드맵

### Phase 1: 즉시 구현 (개인 레포)

```
옵션 1: Repository Dispatch
  ↓
예상 시간: 30분
비용: 무료
지연: ~30초
```

**구현:**
1. 중앙 레포에 `repository_dispatch` 추가 (5분)
2. 각 레포에 `trigger-sync.yml` 추가 (10분)
3. PAT 권한 추가 (`workflow` scope) (5분)
4. 테스트 (10분)

---

### Phase 2: Organization 확장 (3개월 후)

```
옵션 2: Organization Webhook + Lambda
  ↓
예상 시간: 2-3시간
비용: 무료 (Lambda)
지연: ~10초
```

**구현:**
1. AWS Lambda 계정 생성 (10분)
2. Lambda 함수 작성 (1시간)
3. Organization Webhook 설정 (10분)
4. 테스트 및 디버깅 (1시간)

---

### Phase 3: 제품화 (6개월 후)

```
옵션 3: GitHub App
  ↓
예상 시간: 1주일
비용: 무료 (서버 비용만)
지연: ~5초
```

**구현:**
1. GitHub App 생성 및 설정 (1일)
2. App 서버 구현 (2일)
3. 배포 및 테스트 (1일)
4. 문서화 (1일)
5. Marketplace 준비 (2일)

---

## 🔧 즉시 구현 가능: Repository Dispatch

### 코드 예시

#### 중앙 레포 수정 (5분)

```yaml
# .github/workflows/action.yml
on:
  issues:
    types: [opened, edited, closed, reopened, labeled, unlabeled]
  
  # ✨ 이것만 추가!
  repository_dispatch:
    types: [sync_issues]
  
  schedule:
    - cron: '*/15 * * * *'
  workflow_dispatch:
```

#### 다른 레포에 추가 (레포당 10분)

**파일 생성:** `.github/workflows/trigger-sync.yml`

```yaml
name: Trigger Issue Sync

on:
  issues:
    types: [opened, edited, closed, reopened, labeled, unlabeled]

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger central repo
        run: |
          curl -L \
            -X POST \
            -H "Accept: application/vnd.github+json" \
            -H "Authorization: Bearer ${{ secrets.PAT_GITHUB }}" \
            https://api.github.com/repos/${{ github.repository_owner }}/issue-sync-test/actions/workflows/action.yml/dispatches \
            -d '{"ref":"main"}'
```

#### PAT 권한 추가

Classic Token에:
```
✓ repo (이미 있음)
✓ read:project (이미 있음)
✓ workflow  ← 추가!
```

또는 Fine-grained Token:
```
Actions: Read and write ← 추가!
```

### 테스트

1. deeplink-test에 이슈 생성
2. 30초 대기
3. issue-sync-test의 Actions 탭 확인
4. "repository_dispatch" 트리거로 실행됨!
5. Notion 확인

---

## 💰 비용 비교

### 월 1000개 이슈 변경 기준

| 옵션 | 비용 | Actions 분 | 비고 |
|------|------|------------|------|
| **스케줄 (15분)** | 무료 | ~3000분 | 2880회 실행 |
| **Repository Dispatch** | 무료 | ~100분 | 1000회만 실행 |
| **Org Webhook + Lambda** | 무료 | ~100분 | Lambda 무료 티어 |
| **GitHub App + 서버** | $5/월 | ~100분 | 서버 비용 |

**Repository Dispatch가 가장 효율적!** 💚

---

## ⚡ 성능 비교

| 방법 | 평균 지연 | 최대 지연 | 신뢰성 |
|------|-----------|-----------|--------|
| 스케줄 (15분) | 7.5분 | 15분 | ⭐⭐⭐ |
| **Repository Dispatch** | 30초 | 1분 | ⭐⭐⭐⭐ |
| **Org Webhook + Lambda** | 10초 | 30초 | ⭐⭐⭐⭐⭐ |
| GitHub App | 5초 | 15초 | ⭐⭐⭐⭐⭐ |
| 외부 서버 | 3초 | 10초 | ⭐⭐⭐⭐⭐ |

---

## 🎯 최종 추천

### 지금 당장 (개인 레포 2-3개)

**→ Repository Dispatch** ✅

- 구현 시간: 30분
- 비용: 무료
- 효과: 15분 → 30초
- **50배 빠름!**

### 나중에 (Organization 10+ 레포)

**→ Organization Webhook + Lambda** ✅

- 구현 시간: 2-3시간
- 비용: 무료 (Lambda)
- 효과: 15분 → 10초
- **90배 빠름!**

### 미래 (제품화)

**→ GitHub App** 💎

- 구현 시간: 1주일
- 비용: 무료~$5/월
- 효과: Marketplace 배포 가능

---

## 🚀 다음 단계

### 즉시 실행 가능: Repository Dispatch 구현

**구현해드릴까요?**

1. 중앙 레포에 `repository_dispatch` 트리거 추가 (1분)
2. 각 레포용 `trigger-sync.yml` 템플릿 생성 (5분)
3. PAT 권한 추가 가이드 작성 (5분)
4. 테스트 (10분)

**총 20분이면 완료!**

---

## 📊 요약표

| 항목 | 현재 (스케줄) | Repository Dispatch | Org Webhook |
|------|--------------|---------------------|-------------|
| **지연시간** | 7.5분 평균 | 30초 | 10초 |
| **구현시간** | - | 30분 | 2-3시간 |
| **각 레포 설정** | 불필요 | workflow 추가 | 불필요 |
| **Organization 권한** | 불필요 | 불필요 | 필요 |
| **비용** | 무료 | 무료 | 무료 |
| **유지보수** | 쉬움 | 보통 | 보통 |
| **추천도** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 💡 결론

**현재 상황 (개인 레포 2개):**
- **Repository Dispatch 구현 추천**
- 30분이면 완성
- 15분 → 30초로 개선 (30배!)

**향후 확장 (Organization):**
- Organization Webhook + Lambda
- 한 번 설정으로 모든 레포 커버
- 거의 실시간 (10초)

---

**Repository Dispatch 바로 구현해드릴까요?** 🚀

