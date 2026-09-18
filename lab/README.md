# 🛠️ Lab Tooling & Solutions (`lab/`)

이 디렉토리는 핸즈온 랩 실습을 지원하기 위한 바이너리, 스크립트, 템플릿 및 정답 솔루션을 포함하고 있습니다.

---

## 📂 하위 폴더별 역할

- **`bin/`**:
  - `cm-linux`: CodeMender의 핵심 CLI 바이너리입니다. CI/CD 러너(Linux) 환경에서 실행되며, 정적 분석(`find`), 가상 공격 검증(`verify`), AI 코드 패치(`fix`)를 수행합니다.
- **`hooks/`**:
  - `install.sh`: Git Pre-commit 훅을 `.git/hooks/` 디렉토리에 복사하여 활성화하는 스크립트입니다.
  - `pre-commit`: 개발자가 `git commit`을 실행할 때마다 변경된 파일에 대해 Semgrep 정적 분석을 수행하고 위험한 코드가 있으면 커밋을 강제로 중단시킵니다.
- **`scripts/`**:
  - `semgrep-to-cm.py`: Semgrep의 JSON 출력 결과를 CodeMender 내부 취약점 데이터베이스 포맷으로 변환해 주는 브릿지 스크립트입니다.
- **`templates/`**:
  - `codemender-pipeline.template.yaml`: 6단계에서 실습자가 직접 완성해야 할 미완성 CI/CD 파이프라인 템플릿입니다. 내부에 `FIXME 1`, `FIXME 2`, `FIXME 3` 주석이 달려 있습니다.
- **`solutions/`**:
  - `codemender-pipeline.solution.yaml`: 템플릿의 빈칸(FIXME)을 모두 채운 완성형 정답 파일입니다.

---

## 🔍 파이프라인 FIXME 3가지 힌트

실습 중 `.github/workflows/codemender-pipeline.yml`을 작성할 때 채워야 하는 3가지 요소입니다:

1. **FIXME 1 (WIF 인증 권한)**:
   - WIF OIDC 토큰을 발행받으려면 workflow에 `id-token: write` 권한이 반드시 선언되어 있어야 합니다.
2. **FIXME 2 (CodeMender 취약점 스캔)**:
   - `cm find "$SCAN_PATH" -y --model "$CM_MODEL"` 명령어로 전체 스캔을 수행합니다.
3. **FIXME 3 (CodeMender AI 자동 패치)**:
   - `cm fix "$FID" -y --bypass-warning --model "$CM_MODEL"` 명령어로 검증된 취약점(`$FID`)을 Gemini 모델을 통해 자동 수정합니다.
