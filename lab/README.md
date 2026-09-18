# 🛠️ CodeMender DevSecOps Engine & Tooling (`lab/`)

이 디렉토리는 고객사의 **어떤 애플리케이션 저장소(Java, Python, Go, Node.js, C++ 등)**에든 CodeMender 자율 보안 기능을 즉시 이식할 수 있도록 지원하는 핵심 엔진 바이너리와 파이프라인 템플릿을 포함합니다.

---

## 📂 하위 디렉토리 구성 및 역할

- **`bin/`**:
  - `cm-linux`: CodeMender의 핵심 자율 보안 CLI 바이너리(Go 기반)입니다. CI/CD 러너(Linux) 환경에서 실행되며, 언어에 구애받지 않고 소스코드 정적 분석(`find`), 가상 샌드박스 공격 검증(`verify`), Gemini 3.7 기반 코드 자동 패치(`fix`)를 수행합니다.
- **`templates/`**:
  - `codemender-pipeline.template.yaml`: 고객사 저장소에 복사하여 붙여넣을 수 있는 **다국어 자동 감지 지원 범용 CI/CD 파이프라인 템플릿**입니다. 실습용 FIXME 주석이 포함되어 있습니다.
- **`solutions/`**:
  - `codemender-pipeline.solution.yaml`: 모든 설정이 완료된 프로덕션 레디(Production-ready) 완성본 파이프라인 파일입니다.
- **`hooks/`**:
  - `install.sh` / `pre-commit`: 개발자 로컬 PC(Inner Loop)에서 코드를 커밋하기 전 Semgrep 검사를 통해 위험 코드를 사전에 차단하는 Git 훅 스크립트입니다.
- **`scripts/`**:
  - `semgrep-to-cm.py`: 로컬 Semgrep 스캔 결과를 CodeMender 내부 데이터베이스 포맷으로 변환해 주는 브릿지 유틸리티입니다.

---

## 🔌 고객사 기존 저장소에 3분 만에 적용하는 방법 (Plug-and-Play)

고객사의 기존 프로젝트에 CodeMender를 적용하려면 딱 2가지만 복사해 넣으면 됩니다:

1. **엔진 복사**: 고객사 저장소 루트에 `lab/bin/cm-linux` 복사
2. **파이프라인 복사**: 
   ```bash
   mkdir -p .github/workflows
   cp lab/solutions/codemender-pipeline.solution.yaml .github/workflows/codemender-pipeline.yml
   ```
3. 저장소의 `package.json`, `requirements.txt`, `pom.xml`, `go.mod` 등을 파이프라인이 자동으로 감지하여 의존성을 설치하고 소스코드 전체(`.`)를 자율 스캔 및 패치합니다.
