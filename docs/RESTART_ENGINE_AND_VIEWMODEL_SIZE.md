# restartEngineForNewConfig() 호출처 없음 / EngineViewModel 크기 — 설명

## 1. restartEngineForNewConfig() — 호출처가 없는 이유와 연결 방법

### 1.1 이 함수가 하는 일

**위치**: `EngineViewModel.restartEngineForNewConfig()` (라인 2697~2727)

**동작 요약**:
1. 현재 진행 중인 엔진 작업(`engineOpJob`) 취소
2. UI에 "모델 준비 중" 상태 표시 (`statusMessage = msg_model_preparing`)
3. 현재 세션의 분석 중단 (`stopAnalysis`)
4. **현재 활성 세션(분석 또는 대국)의 엔진만** `initializeEngine(forceLowMem = false)`로 재초기화
5. 실패 시 "Engine Restart Failed"로 상태 메시지 설정

즉, **설정을 바꾼 뒤 “지금 쓰는 엔진만 다시 켜고 싶을 때”** 쓰는 API다.  
(CPU 모드, 스레드 수, 레이턴시, 모델 등이 바뀌면 **새 설정이 적용되려면 엔진을 한 번 껐다 켜야** 한다.)

### 1.2 왜 호출처가 없나

- **정의**: `EngineViewModel` 안에만 있고, **다른 클래스에서 이 함수를 호출하는 코드가 전혀 없다** (검색 결과 호출처 0).
- **리소스**: `engine_restarting` ("엔진 재시작 중...") 문자열은 `strings.xml`에 있지만, **이 함수에서는 사용하지 않고** `msg_model_preparing`만 쓴다. 즉 “엔진 재시작” 전용 문구도 현재는 미사용.
- **설정 변경 흐름**:  
  사용자가 설정 화면에서 CPU 모드·스레드·모델 등을 바꾸면 `SettingsRepository` 등에만 반영되고, **이미 떠 있는 엔진 프로세스는 예전 설정으로 계속 동작**한다.  
  엔진이 다시 뜨는 시점은 “세션 전환”(탭 바꿀 때)이나 “앱 재시작”일 뿐이라, **설정만 바꾼 직후에는 새 설정이 적용되지 않는다.**

그래서 “설정 변경 후 엔진 재시작”을 하려면 **어딘가 UI에서 `restartEngineForNewConfig()`를 호출해 줘야** 한다.

### 1.3 “설정 화면에서 엔진 재시작 버튼과 연결”이 의미하는 것

- **연결**: 설정(또는 엔진 설정) 화면에 **“엔진 재시작”** 버튼을 두고, 그 버튼 클릭 시 `EngineViewModel.restartEngineForNewConfig()`를 한 번 호출하도록 연결하는 것이다.
- **효과**:
  - CPU 모드, 스레드 수, 레이턴시, 모델 등 설정을 바꾼 뒤, 사용자가 **버튼 한 번**으로 현재 화면(분석/대국)에서 쓰는 엔진만 재시작할 수 있다.
  - 세션 전환 없이도 새 설정이 적용되고, 앱을 완전히 재시작하지 않아도 된다.
- **노출 위치 후보**:
  - 분석 탭/대국 탭의 “설정” 또는 “엔진 설정” 화면
  - `AnalysisUiViewModel` / `PlayViewModel` 등에서 `engineViewModel`을 쓰고 있으므로, 해당 설정 UI가 있는 Compose/Activity에서 `engineViewModel.restartEngineForNewConfig()`를 호출하면 된다.
- **문자열**: 이미 있는 `engine_restarting`을 재시작 중일 때 `statusMessage`에 넣어 주면, “엔진 재시작 중...” 메시지를 사용자에게 보여 줄 수 있다.

**정리**:  
`restartEngineForNewConfig()`는 “설정 변경 후 엔진 재시작”용으로 설계돼 있지만, **호출하는 UI가 없어서** 지금은 쓰이지 않고 있다.  
설정 화면(또는 엔진 설정)에 “엔진 재시작” 버튼을 두고 이 함수와 연결해 두는 것이 좋다.

---

## 2. EngineViewModel 크기 — 전담 클래스 분리 (적용됨)

**적용 내용**: 엔진 생명주기/세션 전환 관련 타입과 순수 로직을 **EngineLifecycleCoordinator**와 **Session**으로 분리함.

- **Session.kt**: `enum class Session { ANALYSIS, PLAY, OFFLINE }` — 세션 구분을 engine 패키지 공용으로 두어 ViewModel·UI에서 동일 타입 사용.
- **EngineLifecycleCoordinator**:  
  - **타입**: `EngineProfile`, `SessionState` (데이터 클래스).  
  - **로직**: `desiredEngineProfile(session, boardSize, forceLowMem)`, `createEngineProfile(...)`, `getSessionSwitchShutdownDelayMs()`, `savedGameKey(session)`, `captureSessionState(state)`, `applySessionState(base, session)`, `buildEmptySessionState(base, statusMessage, targetLatencyMs, maxTimeSeconds)`.  
  - 생성자: `(app: Context, settingsRepository: SettingsRepository)`.
- **EngineViewModel**:  
  - `private val lifecycleCoordinator = EngineLifecycleCoordinator(app, settingsRepository)` 보유.  
  - 세션 상태·프로필 타입과 위 헬퍼는 모두 coordinator 위임; `setActiveSession`, `ensurePlayEngineReadyForLevel`, `initializeEngine` 등은 그대로 두고 내부에서 coordinator 사용.
- **UI**: `PlayViewModel`, `AnalysisUiViewModel`, `OfflineViewModel`에서 `EngineViewModel.Session` 대신 `com.katago.android.engine.Session` import 후 `Session.PLAY` 등 사용.

이제 “엔진 생명주기/세션 전환” 관련 타입과 순수 로직은 Coordinator와 Session에 모여 있어, 해당 부분 수정 시 EngineViewModel 전체가 아닌 이 둘만 보면 된다.

---

## 3. EngineViewModel 크기 — 왜 나누라고 했는지 (참고)

### 3.1 현재 상태

- **파일**: `EngineViewModel.kt` 한 파일에 **약 3,200줄 이상**, **공개/비공개 함수만 약 88개**.
- **한 파일 안에 모여 있는 역할**:
  - **엔진 생명주기**: 앱 기동 시 초기화(`startInitializationSequence`, `onPreparationCompleted`), `initializeEngine`, `restartEngineForNewConfig`, 크래시/폴백 후 재시도
  - **세션 전환**: `setActiveSession`, `analyzingSession`, ANALYSIS/PLAY/OFFLINE별 상태 저장·복원, `ensurePlayEngineReadyForLevel`, shutdown 후 delay
  - **분석**: `triggerAnalysis`, 분석 요청/결과 처리, `analysisJob`, `stopAllAnalysis`, 분석 모드/레이턴시
  - **오버레이/참고도**: 정책·집약 오버레이, 참고도 재생, variation popup
  - **게임/착수**: `playMove`, `undo`, `redo`, `onCellClicked`, `newGame`, 저장/불러오기
  - **기타**: 토큰/결제, 사운드, 보드/돌 스타일, 로그 공유 등

즉, **“엔진 켜기/끄기/세션 전환”**, **“분석 스케줄/결과”**, **“게임 상태/착수”**, **“UI 오버레이”** 등이 한 ViewModel에 같이 들어 있어서, **파일이 크고** “엔진만 건드리려 해도 분석/게임 코드를 함께 보게 되는” 구조다.

### 2.2 “엔진 생명주기/세션 전환 전담 클래스로 나누면 유리하다”는 말의 의미

- **전담 클래스**:  
  “엔진을 언제 켜고, 언제 끄고, 세션을 어떻게 바꾸는지”만 담당하는 **별도 클래스**를 두자는 것이다.  
  예를 들어 이름을 `EngineLifecycleCoordinator` 또는 `SessionSwitchCoordinator` 정도로 두고, 다음을 이 클래스로 옮기는 식이다.
  - 초기화 시퀀스: 모델 준비 완료 후 엔진 init 트리거
  - 세션 전환: `setActiveSession`에 대응하는 로직(이전 세션 상태 저장, shutdown, delay, 새 세션 init)
  - `ensurePlayEngineReadyForLevel`처럼 “이 세션 엔진이 준비될 때까지” 대기하는 흐름
  - `restartEngineForNewConfig`처럼 “현재 세션 엔진만 재시작”하는 흐름
  - shutdown 후 delay 등 “엔진 생명주기”에만 쓰는 상수/헬퍼

- **EngineViewModel은**:
  - 이 전담 클래스를 **가지고**,
  - “엔진 준비됐을 때 분석 트리거”, “세션에 따라 어떤 controller 쓸지”, “UI 상태(세션별 상태, isEngineReady 등) 갱신”처럼 **ViewModel이 맞는 책임**만 남기고,
  - 나머지 “분석 스케줄/결과”, “게임/착수”, “오버레이” 등은 기존처럼 EngineViewModel에 두거나, 필요하면 별도 UseCase/Helper로 더 나눌 수 있다.

이렇게 나누면:

- **유지보수**: “엔진을 껐다 켜는 로직”을 바꿀 때는 전담 클래스만 보면 되고, “분석/게임 로직”을 바꿀 때는 EngineViewModel의 다른 부분만 보면 된다.
- **테스트**: 엔진 생명주기/세션 전환만 단위 테스트하고 싶을 때, 작은 전담 클래스를 테스트하기 쉽다.
- **가독성**: EngineViewModel이 “엔진 + 분석 + 게임 + 오버레이”가 한 덩어리인 상태보다는, “엔진 생명주기는 A 클래스, EngineViewModel은 A를 쓰고 나머지 orchestration”으로 역할이 나뉜다.

### 3.3 나눌 때 유의할 점 (불안정 방지)

- **상태/진입점**:  
  지금 `analyzingSession`, `currentEngineProfile`, `engineOpJob`, `engineOpMutex` 등은 전부 EngineViewModel에 있다.  
  전담 클래스로 옮기면, “현재 세션”, “현재 엔진 프로필”, “진행 중인 엔진 작업” 등이 **그 클래스와 EngineViewModel 사이에서 일치**하도록 설계해야 한다.  
  한쪽만 갱신되면 “세션은 PLAY인데 엔진은 ANALYSIS용으로 init된 상태” 같은 불일치가 생길 수 있다.
- **진입점**:  
  `setActiveSession`, `ensurePlayEngineReadyForLevel`, `restartEngineForNewConfig` 등은 **UI/다른 ViewModel에서 EngineViewModel을 통해** 호출된다.  
  나눈 뒤에도 **호출 경로는 EngineViewModel의 public 함수**로 두고, 내부에서만 전담 클래스에 위임하는 식이면, 기존 호출처를 바꿀 필요가 없어서 안전하다.
- **delay/상수**:  
  세션 전환 시 shutdown–init 사이 delay(200~600ms) 등은 **문서(§6)와 같이 “엔진 생명주기” 전용**으로 두고, 전담 클래스로 옮기면 “delay를 바꾸는 곳”이 한곳으로 모인다.  
  다른 500/1000ms delay(폴백·재시도용)와 섞이지 않도록 한다.

**정리**:  
EngineViewModel이 “초기화·세션 전환·분석·복구”를 한 파일에 다 들고 있어서 크고 복잡하다.  
**“엔진 생명주기/세션 전환”만 전담하는 클래스로 빼고**, EngineViewModel은 그 클래스를 쓰면서 “언제 분석을 트리거할지, UI 상태를 어떻게 갱신할지”에 집중하면, **유지보수와 테스트가 쉬워진다.**  
나눌 때는 “현재 세션/엔진 상태의 일관성”, “기존 진입점(EngineViewModel public API) 유지”, “delay/상수는 생명주기 전용으로만 사용”을 지키면 불안정을 줄일 수 있다.
