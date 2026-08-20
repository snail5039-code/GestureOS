# GestureOS 프로젝트 현황

최종 점검일: 2026-08-20

## 구성

- `GestureOSManager/front`: Electron + React 데스크톱 관리자
- `GestureOSManager/gestureOSManager`: 8080 제어 및 WebSocket 서버
- `GestureOSManager/py`: MediaPipe 인식, OS 입력, HUD, 휴대폰 브리지
- `GestureOSManagerWeb/frontend-react`: 5174 웹사이트
- `GestureOSManagerWeb/backend-spring`: 8082 웹 API
- `GestureOSManagerWeb/docker-compose.yml`: PostgreSQL 16

## 2026-08-20 점검에서 수정한 내용

- **마우스 모드가 첫 프레임에서 죽던 문제.** `hands_agent.py` 의 `run()` 이
  `block_by_palette` 를 대입(1410줄)하기 전에 읽고 있었다(1400줄). 같은 함수 안에서
  나중에 대입되는 이름이라 파이썬은 이를 지역변수로 잡고 `UnboundLocalError` 를 낸다.
  조건이 `mode_u == "MOUSE" and self.enabled and (not self.ui_locked)` 다음이라
  마우스 모드를 켜면 매 프레임 걸리는 자리였고, `run()` 을 감싸는 `except` 가 없어
  (`main.py` 는 `try/finally` 만 둔다) 에이전트 프로세스가 그대로 종료됐다.
  팔레트 모달 계산 블록을 핀치 고정 블록보다 앞으로 옮겨 해결했다.
- **검사 목록에 `pyflakes` 를 넣었다.** `compileall` 은 구문만 본다. 위 오류도
  구문 검사는 통과하고 있었다. 같은 것을 또 놓치지 않으려면 정적 검사가 필요하다.

  ```powershell
  py -3.10 -m pyflakes GestureOSManager\py
  ```
- **Electron 진입 파일이 lint 대상에서 빠져 있었다.** `front/eslint.config.js` 의
  `files` 가 `**/*.{js,jsx}` 라서 `electron/main.cjs`·`electron/preload.cjs` 에는
  규칙이 하나도 적용되지 않았다(적용 규칙 0개, `src/main.jsx` 는 79개).
  `**/*.cjs` 를 CommonJS + Node 전역으로 따로 넣었다. 적용 후 lint 통과.

## 2026-07-25 점검에서 수정한 내용

- 웹 프런트의 `ProtectedRoute`에 누락된 React Router 및 인증 훅 import 추가
- 두 프런트의 ESLint 설정을 실제 런타임 결함에 집중하도록 정리
- 사용하지 않는 변수와 헬퍼 제거
- ASCII 판별 정규식을 명시적인 코드포인트 검사로 변경
- Tailwind 설정의 CommonJS `require`를 ESM import로 변경
- React 훅 의존성 배열이 매 렌더마다 바뀌던 값을 메모이제이션
- 웹 API 테스트가 외부 PostgreSQL 없이 실행되도록 H2 테스트 DB 추가
- 테스트용 OAuth, 메일, OpenAI 더미 설정 추가
- 로컬 Maven/Python 설치 산출물과 로그를 루트 `.gitignore`에서 제외

## 검증 상태

| 검사 | 상태 |
|---|---|
| 데스크톱 React ESLint | 통과 |
| 데스크톱 React 프로덕션 빌드 | 통과 |
| 웹 React ESLint | 통과 |
| 웹 React 프로덕션 빌드 | 통과 |
| 8080 Spring 테스트 | 통과 |
| 8082 Spring 테스트 | 통과 |
| Python 구문 검사 (compileall) | 통과 |
| Python 정적 검사 (pyflakes) | 통과 |
| MediaPipe/OpenCV/PySide6 핵심 import | 통과 |

## 환경 의존 항목

다음 항목은 코드 오류가 아니라 로컬 또는 배포 환경 준비가 필요합니다.

- 실제 서비스 실행용 PostgreSQL 16
- OpenAI API 키
- SMTP 계정
- Google/Kakao/Naver OAuth 등록 정보
- 카메라 권한
- Windows 입력 주입 권한

## 알려진 개선 항목

- `react-hooks` 의 `immutability`·`purity`·`set-state-in-effect` 를 `off` 로 두고 있다.
  다시 켜면 데스크톱 4건·웹 5건이 나온다. 대부분 오탐이지만
  `set-state-in-effect` 3건(`PairingQrModal.jsx:62`, `ChatWidget.jsx:41`·`47`)은 확인이 필요하다
- `electron/main.cjs` 의 `shell:openExternal` 핸들러가 URL 스킴을 검증하지 않는다.
  `http`/`https` 로 제한하는 편이 안전하다
- `electron/main.cjs` 는 항상 `localhost:5173` 을 로드한다. 패키징용 분기가 없어
  설치본에서는 화면이 뜨지 않는다 (설치형 배포가 개선 과제로 남은 이유)
- Spring 테스트는 두 모듈 모두 컨텍스트 로딩 1건뿐이다. 기능 검증이 아니다
- 데스크톱 프로덕션 번들의 큰 청크를 화면별 동적 import로 분리
- npm audit 경고를 호환성 검증과 함께 단계적으로 업데이트
- 루트 `README.md`의 깨진 한글 인코딩을 UTF-8 문서로 교체
- 전체 구성을 한 번에 시작하고 종료하는 PowerShell 스크립트 추가
- 실제 PostgreSQL을 사용하는 통합 테스트 또는 Testcontainers 추가
- 카메라/에이전트 중복 실행을 앱 시작 단계에서 자동 감지

## 다음 작업 권장 순서

1. PostgreSQL과 실제 환경변수를 준비해 8082 API 통합 테스트
2. 로그인, 게시판, 댓글, 파일 업로드 흐름 점검
3. 카메라 인식과 MOUSE/KEYBOARD/PRESENTATION 모드 회귀 테스트
4. npm 의존성 보안 경고 검토
5. 배포 환경의 CORS, OAuth redirect URI, WebSocket 프록시 검증

