# 📚 Documentation Index

GitHub Issues와 Notion 동기화 설정 및 사용 가이드입니다.

## 🚀 빠른 시작

개인 레포에서 테스트하는 단계별 가이드:

### Step 1: GitHub Projects V2 설정
**[→ 01-projects-setup.md](./01-projects-setup.md)**

GitHub Projects V2를 생성하고 필드를 설정합니다.
- Projects 생성
- 필드 추가 (Status, Priority, Story Points, Capacity 등)
- 테스트 이슈 생성 및 추가

예상 소요 시간: **10-15분**

---

### Step 2: Notion 데이터베이스 설정
**[→ 02-notion-setup.md](./02-notion-setup.md)**

Notion Integration을 만들고 데이터베이스를 준비합니다.
- Integration 생성 및 API Key 발급
- 데이터베이스 생성
- 필수 속성 추가
- Integration 연결

예상 소요 시간: **15-20분**

---

### Step 3: GitHub Secrets 설정
**[→ 03-github-secrets.md](./03-github-secrets.md)**

Repository Secrets에 API Key를 안전하게 저장합니다.
- NOTION_API_KEY 추가
- NOTION_DATABASE_ID 추가
- Workflow 권한 설정

예상 소요 시간: **5분**

---

### Step 4: 테스트 및 실행
**[→ 04-testing.md](./04-testing.md)**

동기화를 실행하고 결과를 확인합니다.
- 수동 워크플로우 실행
- 결과 확인
- 자동 동기화 테스트
- 문제 해결

예상 소요 시간: **10분**

---

### Step 5: 여러 레포 동기화 (선택사항)
**[→ 05-multiple-repos.md](./05-multiple-repos.md)**

여러 레포의 이슈를 한 곳에서 중앙 관리합니다.
- config.yml 설정
- PAT 설정 (Classic Token 권장)
- 중앙 수집 모드

예상 소요 시간: **15분** (Organization은 20분)

---

### Step 6: Classic Token 가이드 (PAT 필요 시)
**[→ 06-classic-token-guide.md](./06-classic-token-guide.md)**

여러 레포 및 Projects 연동을 위한 Classic Token 생성 가이드입니다.
- Classic Token 생성 방법
- 필요한 권한 (repo, read:project)
- Secret 추가 (PAT_GITHUB)
- 문제 해결

예상 소요 시간: **10분**

---

### Step 7: 커스텀 필드 추가 (고급)
**[→ 07-custom-fields-guide.md](./07-custom-fields-guide.md)**

GitHub Projects나 Notion에 커스텀 필드를 추가하는 방법입니다.
- 필드 타입별 코드 예시
- 복사-붙여넣기 템플릿
- 실전 예시 (Team, Due Date 등)
- 코드 수정 위치 안내

예상 소요 시간: **15-30분** (필드당 5-10분)

---

### Step 8: 실시간 동기화 방안 (향후 발전)
**[→ 08-realtime-sync-options.md](./08-realtime-sync-options.md)**

15분 스케줄을 넘어 실시간 동기화를 구현하는 방법들을 분석합니다.
- 5가지 옵션 비교 (Repository Dispatch, Org Webhook, GitHub App 등)
- 비용/성능/난이도 분석
- 단계별 구현 로드맵
- 즉시 구현 가능한 방법 (30분 소요)

**지연시간:** 15분 → 30초 (30배 개선!)

---

## 📋 전체 진행 체크리스트

### Prerequisites (사전 준비)
- [ ] GitHub 계정
- [ ] Notion 계정
- [ ] 테스트용 GitHub 레포지토리 (개인 레포)

### Setup (설정)
- [ ] GitHub Projects V2 생성 ✅ [가이드](./01-projects-setup.md)
- [ ] Notion Integration 생성 ✅ [가이드](./02-notion-setup.md)
- [ ] Notion 데이터베이스 생성 ✅ [가이드](./02-notion-setup.md)
- [ ] GitHub Secrets 추가 ✅ [가이드](./03-github-secrets.md)

### Testing (테스트)
- [ ] 수동 동기화 실행 ✅ [가이드](./04-testing.md)
- [ ] Notion에서 결과 확인 ✅ [가이드](./04-testing.md)
- [ ] 자동 동기화 확인 ✅ [가이드](./04-testing.md)

### Advanced (고급 기능) - 구현 예정
- [ ] 여러 레포 동기화
- [ ] Projects V2 필드 동기화
- [ ] Organization 레벨 프로젝트 연동

---

## 🎯 현재 기능 상태

### ✅ 구현 완료
- GitHub Issues → Notion 동기화
- 이슈 제목, 번호, 상태, 라벨, URL, 생성일, 담당자
- Markdown → Notion 블록 변환
  - 헤딩 (# ## ###)
  - 코드 블록 (```)
  - 리스트 (-, 1.)
  - 체크박스 (- [ ])
  - 인용구 (>)
  - 인라인 스타일 (**, `)
- 자동 동기화 (이벤트, 스케줄)
- 수동 동기화

### 🚧 구현 중
- Projects V2 연동 (다음 단계)

### 📅 예정
- 여러 레포 동기화
- Notion → GitHub 양방향 동기화
- 코멘트 동기화
- 마일스톤 지원

---

## 📖 추가 자료

### 공식 문서
- [GitHub Projects V2 Docs](https://docs.github.com/en/issues/planning-and-tracking-with-projects)
- [Notion API Docs](https://developers.notion.com/)
- [GitHub Actions Docs](https://docs.github.com/en/actions)

### API 참조
- [GitHub REST API - Issues](https://docs.github.com/en/rest/issues)
- [GitHub GraphQL API - Projects](https://docs.github.com/en/graphql/reference/objects#projectv2)
- [Notion API - Databases](https://developers.notion.com/reference/database)
- [Notion API - Blocks](https://developers.notion.com/reference/block)

---

## 🆘 도움말

### 문제가 발생했나요?

1. **[문제 해결 가이드](./04-testing.md#문제-해결)** 확인
2. **Actions 로그** 확인
3. **GitHub Issues**에 질문 남기기

### 일반적인 에러

| 에러 | 해결 방법 |
|------|-----------|
| 401 Unauthorized (Notion) | [가이드](./03-github-secrets.md) - NOTION_API_KEY 확인 |
| 404 Not Found (Notion) | [가이드](./02-notion-setup.md) - Database ID 확인 |
| 403 Forbidden (GitHub) | [가이드](./03-github-secrets.md) - Workflow permissions 확인 |
| GITHUB_REPOSITORY not set | Actions 탭에서 실행했는지 확인 |

---

## 💡 팁

### 처음 설정하시나요?
1. 순서대로 Step 1 → 2 → 3 → 4 진행
2. 각 단계의 체크리스트 확인
3. 문제가 생기면 해당 가이드의 "문제 해결" 섹션 참고

### 이미 설정을 완료했나요?
- [테스트 가이드](./04-testing.md)로 바로 이동
- 새 이슈를 만들어서 동기화 테스트

### Organization에 적용하고 싶나요?
- 개인 레포에서 먼저 테스트
- 테스트 성공 후 Organization 레포로 확장
- PAT (Personal Access Token) 필요할 수 있음

---

## 📝 피드백

이 문서에 대한 피드백이나 개선 사항이 있으시면:
- GitHub Issues에 등록
- Pull Request 환영합니다!

---

**마지막 업데이트:** 2024-01-17
**버전:** 1.0 (기본 동기화)
**다음 버전 예정:** 1.1 (Projects V2 연동)

