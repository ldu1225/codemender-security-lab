# 🤖 GitHub Actions Workflow Guide (`.github/`)

이 디렉토리는 GitHub Actions 자동화 파이프라인의 핵심 설정 파일들과 보조 스크립트들을 포함합니다.

---

## 📂 디렉토리 구성

- **`workflows/`**:
  - GitHub Actions 엔진이 자동으로 감지하는 작업 지시서(`.yml`)가 배치되는 표준 경로입니다.
  - 실습 6단계에서 `lab/templates/codemender-pipeline.template.yaml`을 복사하여 `codemender-pipeline.yml`로 배치해야 GitHub Actions 탭에서 인식합니다.
- **`scripts/`**:
  - `cm_verdict.py`: CodeMender의 취약점 검증 결과(JSON)를 파싱하여 Exploit이 실제로 성공했는지 판정하는 스크립트입니다.
  - `extract_cm_diff.py`: CodeMender 실행 로그에서 AI가 제안한 Git diff 패치 코드를 안전하게 추출하여 파일에 적용해 주는 헬퍼 스크립트입니다.

---

## ⚙️ 필수 GitHub 저장소 설정 체크리스트

파이프라인이 정상적으로 동작하려면 GitHub 웹 저장소 설정에서 아래 항목들이 반드시 구성되어 있어야 합니다:

### 1. Workflow 쓰기 권한 허용 (필수)
AI 에이전트가 새로운 브랜치를 만들고 Pull Request를 생성할 수 있도록 권한을 열어주어야 합니다.
- 경로: **Settings > Actions > General > Workflow permissions**
- 설정:
  - [x] **Read and write permissions** 선택
  - [x] **Allow GitHub Actions to create and approve pull requests** 체크박스 활성화 후 Save

### 2. 저장소 변수(Repository Variables) 등록 (필수)
WIF 및 구글 클라우드 연결 정보는 코드에 하드코딩하지 않고 저장소 변수로 주입합니다.
- 경로: **Settings > Secrets and variables > Actions > Variables (Secrets 아님)**
- 등록할 변수 3개:
  - `GCP_WIF_PROVIDER`: Google Cloud에서 생성한 WIF Provider 전체 리소스 경로
    - 예: `projects/1234567890/locations/global/workloadIdentityPools/github-actions/providers/github-oidc`
  - `GCP_SA_EMAIL`: CodeMender 전용 서비스 계정 이메일
    - 예: `codemender-ci@your-project-id.iam.gserviceaccount.com`
  - `GCP_QUOTA_PROJECT`: API 할당량을 청구할 GCP 프로젝트 ID
    - 예: `your-project-id`
