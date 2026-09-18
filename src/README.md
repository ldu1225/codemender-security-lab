# 🚨 Target Application Source Code (`src/`)

이 디렉토리는 보안 취약점 실습을 위해 고의로 보안 취약점(Vulnerabilities)을 내장해 둔 Node.js/Express e-커머스 백엔드 애플리케이션입니다.

---

## 📂 파일 구조 및 취약점 맵

| 파일 경로 | 취약점 유형 | 심각도 | 설명 | 해결 주체 |
| :--- | :--- | :---: | :--- | :---: |
| `api/controllers/admin.controller.js` | **RCE** (`eval` 원격 코드 실행) | **CRITICAL** | 사용자 수식을 `eval()`에 그대로 전달하여 임의의 자바스크립트/시스템 명령 실행 가능 | **개발자 로컬 실습**<br>(Shift-Left 단계) |
| `services/catalog.service.js` | **SSRF** (서버 사이드 요청 위조) | **HIGH** | `fetchRemoteAsset`에서 URL 호스트 및 내부 사설 IP 검증 누락 | **CodeMender AI 에이전트**<br>(자동 PR 생성) |
| `services/checkout.service.js` | **Memory Leak** (Buffer 미초기화) | **HIGH** | `Buffer.allocUnsafe()` 사용으로 힙 메모리의 민감 정보 노출 | CodeMender 탐지 |
| `services/admin.service.js` | **Command Injection** (OS 명령 삽입) | **HIGH** | 핑 유틸리티 옵션에 `shell: true` 인젝션 가능 | CodeMender 탐지 |
| `services/auth.service.js` | **Mass Assignment** (권한 상승) | **HIGH** | 프로필 수정 시 `role` 필드 검증 누락으로 관리자 권한 탈취 가능 | CodeMender 탐지 |

---

## 🛠️ 실습 가이드: 5단계 로컬 Shift-Left 수정하기

[`src/api/controllers/admin.controller.js`](./api/controllers/admin.controller.js) 파일의 8~14번째 줄에 있는 `previewDynamicPricing` 함수는 아래와 같이 위험한 `eval()`을 사용하고 있습니다:

```javascript
// ❌ 취약한 코드 (원래 상태)
exports.previewDynamicPricing = (req, res) => {
    try {
        res.json({ price: eval(req.body.formula) });
    } catch (e) {
        res.status(400).send("Evaluation Failed");
    }
};
```

### 올바른 수정 예시:
정규식을 사용하여 숫자와 사칙연산 기호(`0-9 + - * / ( ) .`)만 허용하도록 검증 헬퍼를 추가하고 수정합니다:

```javascript
// ✅ 안전하게 수정한 코드
function safeCalculate(formula) {
    if (typeof formula !== 'string' || !/^[0-9+\-*/().\s]+$/.test(formula)) {
        throw new Error("Invalid formula expression");
    }
    return Function(`'use strict'; return (${formula})`)();
}

exports.previewDynamicPricing = (req, res) => {
    try {
        res.json({ price: safeCalculate(req.body.formula) });
    } catch (e) {
        res.status(400).send("Evaluation Failed");
    }
};
```

수정 후 `git commit`을 시도하면, 로컬 Pre-commit 훅(Semgrep)이 검사를 통과시켜 정상적으로 커밋이 완료됩니다!
