# 🧪 CodeMender CI/CD DevSecOps Hands-on Lab (GitHub & Google Cloud WIF)

이 실습 저장소는 **Google Cloud WIF (Workload Identity Federation)** 와 **CodeMender (Gemini 기반 자율 보안 에이전트)** 를 활용하여, 비밀 키(Service Account Key) 없이 GitHub Actions CI/CD 파이프라인에서 보안 취약점을 자동으로 탐지, 검증, 패치 및 Pull Request까지 생성하는 전체 DevSecOps 워크플로우를 직접 구축하고 실습할 수 있도록 설계된 핸즈온 랩입니다.

---

## 🎯 학습 목표
1. **Keyless Cloud 인증 (WIF)**: 영구적인 JSON 서비스 계정 키 없이 OIDC 토큰 교환을 통해 안전하게 GCP에 인증하는 메커니즘을 이해하고 구성합니다.
2. **Shift-Left Local Security**: 개발자 로컬 환경에서 Git Pre-commit Hook과 Semgrep을 통해 취약점을 조기에 차단하는 Inner Loop를 체험합니다.
3. **CI/CD Security Pipeline (GitHub Actions)**: 코드가 Push될 때마다 자동으로 실행되는 파이프라인 지시서(`.github/workflows/*.yml`)를 작성합니다.
4. **CodeMender (`cm`) 자율 보안 에이전트**:
   - `cm find`: 정적 분석을 통해 취약점 탐지
   - `cm verify`: 모의 공격 페이로드 실행을 통한 Exploit 검증 (오탐 제거)
   - `cm fix`: Gemini LLM 기반 자동 보안 패치 생성 및 테스트 통과 확인
   - `Autonomous PR`: 수정된 코드를 브랜치로 푸시하고 자동으로 PR 제출
5. **Security Gate**: 취약점이 발견되면 사람이 승인하기 전까지 프로덕션 배포를 차단(Block)하는 엔터프라이즈 게이트를 검증합니다.

---

## 📂 프로젝트 폴더 구조 및 가이드

각 폴더마다 상세한 설명과 가이드가 담긴 `README.md`가 포함되어 있습니다:

```text
├── README.md                      # [현재 파일] 전체 실습 가이드 및 아키텍처
├── package.json                   # 취약점이 포함된 Node.js 샘플 애플리케이션
├── src/                           # 🚨 보안 취약점 대상 애플리케이션 소스코드
│   └── README.md                  # 발견 대상 취약점(RCE, SSRF 등) 설명서
├── lab/                           # 🛠️ 실습 도구 및 템플릿
│   ├── README.md                  # lab 디렉토리 역할 안내
│   ├── bin/                       # CodeMender CLI 바이너리 (cm-linux)
│   ├── hooks/                     # 로컬 Git pre-commit 훅 스크립트
│   ├── scripts/                   # Semgrep -> CodeMender 변환기 등
│   ├── templates/                 # 9단계에서 완성할 파이프라인 템플릿
│   └── solutions/                 # 완성본 정답 파이프라인 파일
└── .github/                       # 🤖 GitHub Actions 설정
    ├── README.md                  # Actions 워크플로우 구조 및 동작 원리
    ├── scripts/                   # PR 생성, 패치 추출 등 파이프라인 보조 스크립트
    └── workflows/                 # 실습자가 최종 배치할 CI/CD 워크플로우 폴더
```

---

## 🚀 전체 단계별 실습 순서 (Step-by-Step)

### [1단계] Google Cloud 사전 준비 (GCP Console 또는 Cloud Shell)
실습에 사용할 GCP 프로젝트 ID를 환경변수로 지정하고 필요한 API를 활성화합니다.

```bash
# 1. 프로젝트 ID 설정 (본인의 GCP 프로젝트 ID로 변경)
export PROJECT_ID="YOUR_GCP_PROJECT_ID"
gcloud config set project $PROJECT_ID
export PROJECT_NUM=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')

# 2. 필수 Google Cloud API 활성화
gcloud services enable \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  cloudresourcemanager.googleapis.com \
  aiplatform.googleapis.com \
  serviceusage.googleapis.com
```

---

### [2단계] Workload Identity Federation (WIF) 설정 (Keyless 인증)
GitHub Actions 러너가 비밀 키 없이 인증할 수 있도록 WIF 풀과 프로바이더를 생성합니다.

```bash
# 1. Workload Identity Pool 생성
gcloud iam workload-identity-pools create "github-actions" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# 2. Workload Identity Provider 생성 (OIDC 연동)
# 본인의 GitHub 사용자명(Owner)을 지정합니다. (예: ldu1225)
export GITHUB_OWNER="YOUR_GITHUB_USERNAME"

gcloud iam workload-identity-pools providers create-oidc "github-oidc" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github-actions" \
  --display-name="GitHub Actions OIDC Provider" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository_owner == '${GITHUB_OWNER}'"
```

---

### [3단계] 서비스 계정(Service Account) 및 IAM 권한 부여
CodeMender가 Gemini 모델(`Vertex AI`)을 호출할 수 있는 권한을 서비스 계정에 부여하고, WIF와 연결합니다.

```bash
# 1. 서비스 계정 생성
gcloud iam service-accounts create "codemender-ci" \
  --project="${PROJECT_ID}" \
  --display-name="CodeMender CI Service Account"

export SA_EMAIL="codemender-ci@${PROJECT_ID}.iam.gserviceaccount.com"

# 2. Vertex AI 호출 권한 및 서비스 사용 권한 부여
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/serviceusage.serviceUsageConsumer"

gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/aiplatform.admin"

# 3. WIF Provider와 서비스 계정 바인딩 (이 저장소에서만 권한 대행 허용)
export REPO_NAME="codemender-security-lab" # 본인 저장소 이름
gcloud iam service-accounts add-iam-policy-binding "${SA_EMAIL}" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUM}/locations/global/workloadIdentityPools/github-actions/attribute.repository/${GITHUB_OWNER}/${REPO_NAME}"
```

---

### [4단계] GitHub 저장소 생성 및 환경 변수 등록
1. GitHub 웹에서 `codemender-security-lab` 저장소를 생성하고 이 코드를 푸시합니다.
2. **저장소 권한 설정**:
   - `Settings` > `Actions` > `General` > **Workflow permissions**
   - **Read and write permissions** 체크
   - **Allow GitHub Actions to create and approve pull requests** 체크 후 저장
3. **Repository Variables 등록**:
   - `Settings` > `Secrets and variables` > `Actions` > `Variables` 탭에서 추가:
     - `GCP_WIF_PROVIDER`: `projects/<PROJECT_NUM>/locations/global/workloadIdentityPools/github-actions/providers/github-oidc`
     - `GCP_SA_EMAIL`: `codemender-ci@<PROJECT_ID>.iam.gserviceaccount.com`
     - `GCP_QUOTA_PROJECT`: `<PROJECT_ID>`

---

### [5단계] 로컬 Shift-Left 보안 실습 (Inner Loop)
로컬에서 Git Hook을 등록하고, 코드를 커밋할 때 Semgrep이 취약점을 사전에 차단하는 것을 체험합니다.

```bash
# 1. Semgrep 설치 및 Git pre-commit 훅 활성화
pip install semgrep
./lab/hooks/install.sh

# 2. 취약점 수정 실습 (src/api/controllers/admin.controller.js)
# eval() 함수를 제거하고 정규식 기반 안전 계산 로직으로 수정한 뒤 커밋합니다.
git add src/api/controllers/admin.controller.js
git commit -m "fix(security): sanitize dynamic eval in admin.controller.js"
git push origin main
```
> 👉 자세한 수정 방법은 [`src/README.md`](./src/README.md)를 참고하세요.

---

### [6단계] CI/CD 파이프라인 완성 및 실행 (Outer Loop)
1. 파이프라인 템플릿 복사:
   ```bash
   mkdir -p .github/workflows
   cp lab/templates/codemender-pipeline.template.yaml .github/workflows/codemender-pipeline.yml
   ```
2. `.github/workflows/codemender-pipeline.yml` 파일 내부의 **FIXME 1, 2, 3**을 완성합니다.
   *(막힐 때는 [`lab/solutions/codemender-pipeline.solution.yaml`](./lab/solutions/codemender-pipeline.solution.yaml)을 참고하세요)*
3. 커밋 및 푸시하여 파이프라인을 트리거합니다:
   ```bash
   git add .github/workflows/codemender-pipeline.yml
   git commit -m "ci: add codemender security pipeline"
   git push origin main
   ```

---

### [7단계] 결과 확인 및 배포 게이트 체험
1. GitHub 저장소의 **[Actions]** 탭으로 이동하여 파이프라인 실행 과정을 확인합니다.
2. CodeMender가 남은 취약점(SSRF 등)을 감지하고, Gemini AI가 스스로 작성한 **[Pull Request]**가 새로 열렸는지 확인합니다!
3. 파이프라인의 **Security Gate** 단계가 왜 실패(`exit 1`)로 끝났는지 이유를 확인합니다 (취약점 잔존 상태에서 자동 배포 차단).
4. 생성된 Pull Request를 리뷰하고 **[Merge pull request]** 버튼을 누르면, 파이프라인이 다시 실행되며 검증하는 선순환 과정을 관찰합니다.
