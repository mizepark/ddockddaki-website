# 엔진 초기화 전체 구조 점검 — 이벤트·모드·오류 가능성

## 1. 엔진 초기화 진입점 정리

| 진입점 | 호출 경로 | 세션(대상 엔진) | 비고 |
|--------|-----------|------------------|------|
| **startInitializationSequence()** | EngineViewModel init (locale 있음) | **실행 시점의 analyzingSession** (기본 ANALYSIS) | 비동기 modelInstallJob → onPreparationCompleted() → initializeEngine() |
| **startEngineAfterLocaleSelected()** | 사용자가 언어 선택 후 | **실행 시점의 analyzingSession** | startInitializationSequence() 재실행 |
| **setActiveSession(session)** | 분석/대국/오프라인 탭 전환 | **전환 대상 session** | 이전 엔진 shutdown → delay → initializeEngine(새 세션) |
| **onPreparationCompleted()** | 모델 준비 완료(또는 이미 준비) | **job 실행 시점의 analyzingSession** | engineInitJob에서 initializeEngine() 호출. **레이스**: 사용자가 그 전에 PLAY로 전환하면 PLAY 엔진만 초기화됨 |
| **restartEngineForNewConfig()** | EngineConfigChangeNotifier (설정 변경) | **현재 analyzingSession** | isEngineInitOrRestartInProgress로 중복 방지 |
| **ensureAnalysisEngineReadyForHint()** | triggerAnalysis 시 (PLAY+HINT/DEEP 또는 **ANALYSIS 세션**) | **항상 ANALYSIS** | 분석 전용 엔진만 초기화, 이미 준비 시 스킵 |
| **ensurePlayEngineReadyForLevel()** | 대국 시작 전 (PlayViewModel) | **PLAY** | 대국 엔진만 초기화 |

- **initializeEngine()** 내부에서는 `engineController = controllerFor(analyzingSession)`으로 **현재 analyzingSession**에 해당하는 엔진만 초기화한다.
- 따라서 **세션 전환 타이밍**과 **비동기 job 실행 시점**이 어긋나면, “지금 쓰는 화면”과 “지금 초기화되는 엔진”이 달라질 수 있다.

---

## 2. 모드·화면별 이벤트와 엔진 상태

### 2.1 앱 기동

- **locale 없음**: init에서 `startInitializationSequence()` 호출 안 함 → 엔진 초기화 없음. 사용자가 언어 선택 → `startEngineAfterLocaleSelected()` → 그때 `analyzingSession`으로 초기화.
- **locale 있음**: `startInitializationSequence()` 비동기 실행.
  - `modelInstallJob`에서 모델 준비(또는 스킵) 후 `onPreparationCompleted()` 호출.
  - **onPreparationCompleted()**는 **별도 launch**로 `engineInitJob` 실행 → **그 시점의 analyzingSession**으로 `initializeEngine()` 호출.
  - **레이스**: 메인/분석 화면이 먼저 보이고 사용자가 그 전에 대국 탭으로 전환하면 `analyzingSession == PLAY` → PLAY 엔진만 초기화, 분석 엔진은 미초기화. (이미 수정: triggerAnalysis에서 ANALYSIS일 때 ensureAnalysisEngineReadyForHint 호출로 보완)

### 2.2 모드 전환 (setActiveSession)

- **ANALYSIS ↔ PLAY ↔ OFFLINE** 전환 시:
  - 이전 세션 상태 저장, `analyzingSession = session`, UI 상태 복원.
  - `currentEngineProfile != desired`이면:
    - `isModelReady = false` 설정.
    - 이전 세션 엔진 shutdown → delay → `initializeEngine(triggerInitialAnalysis = false)`.
  - 초기화 **성공 시에만** `triggerAnalysis()` 호출; 실패 시 "모델 준비 중" 등으로 표시하고 분석 트리거 안 함.
- **동시성**: `engineOpJob` 하나로 세션 전환 전체가 묶여 있어, 연속 전환 시 이전 job이 취소되고 새 job만 실행됨.

### 2.3 분석 탭에서의 분석 (triggerAnalysis)

- **isModelReady == false**면 분석 job 자체를 시작하지 않음.
- **ANALYSIS 세션**이면 **항상** `ensureAnalysisEngineReadyForHint()` 호출 → 분석 엔진이 한 번도 초기화되지 않았으면 그 시점에 초기화 후 분석 진행.
- PLAY+HINT/DEEP일 때도 `ensureAnalysisEngineReadyForHint()` 호출 (참고용 분석 엔진).

### 2.4 대국 시작 (PlayViewModel)

- 대국 시작 전 `ensurePlayEngineReadyForLevel()`로 PLAY 엔진 준비.
- 실패/타임아웃 시 대국 시작하지 않고 메시지 표시.

### 2.5 설정 변경 (EngineConfigChangeNotifier)

- CPU 모드, 스레드 수, 레이턴시, 방문 수 등 변경 시 `restartEngineForNewConfig()`.
- `isEngineInitOrRestartInProgress == true`면 재시작 스킵(중복 방지).
- 재시작 시 **현재 analyzingSession** 엔진만 재초기화.

### 2.6 오프라인 모드

- `setActiveSession(OFFLINE)` 시 분석/대국 엔진 사용하지 않음. `triggerAnalysis()` 내부에서 `analyzingSession == OFFLINE`이면 분석 job을 시작하지 않음.

---

## 3. 돌발/경계 상황별 점검

| 상황 | 동작 | 잠재 이슈·대응 |
|------|------|------------------|
| **앱 기동 직후 사용자가 재빨리 PLAY 탭으로 이동** | onPreparationCompleted의 engineInitJob이 아직 안 돌았을 수 있음 → job 실행 시점에 analyzingSession=PLAY → PLAY 엔진만 초기화 | 분석 탭으로 돌아가면 triggerAnalysis에서 ensureAnalysisEngineReadyForHint로 분석 엔진 초기화 (수정 반영됨) |
| **세션 전환 중 연속 탭 전환** | engineOpJob이 cancel되고 새 세션용 job만 실행 | 괜찮음. 마지막 전환만 유효. |
| **initializeEngine() 실패 (타임아웃, 크래시, 모델 없음)** | runCatching 또는 onPreparationCompleted 내 catch에서 처리. setActiveSession에서는 isModelReady가 false로 유지 → triggerAnalysis 스킵 | 재시도/CPU 폴백 등은 onPreparationCompleted 쪽 로직에 의존. setActiveSession에서는 “초기화 실패 시 분석 트리거 안 함”으로 빈 결과 반복 방지. |
| **엔진 초기화 진행 중 설정 변경** | restartEngineForNewConfig에서 isEngineInitOrRestartInProgress면 스킵 | 중복 재시작 방지됨. |
| **엔진 초기화 진행 중 앱 백그라운드/종료** | job 취소 또는 프로세스 종료. 재진입 시 locale 있으면 startInitializationSequence가 다시 호출되지는 않음(isInitialized). 언어 선택 등으로 startEngineAfterLocaleSelected 호출 시에만 재초기화. | 정상. |
| **모델 준비 실패 (prepareModel 실패)** | startInitializationSequence catch에서 isModelReady=false, isInitialized=false로 두어 재시도 가능 | 정상. |
| **restoreSavedGame()에서 triggerAnalysis()** | 세션별 저장 게임 복원 후 분석 트리거. 이때 analyzingSession은 이미 해당 세션. ANALYSIS면 ensureAnalysisEngineReadyForHint로 분석 엔진 보장. | 수정으로 분석 엔진 미초기화 시에도 보완됨. |
| **ensureAnalysisEngineReadyForHint 실패 (모델 없음 등)** | 예외 전파 → triggerAnalysis의 catch에서 statusMessage 설정. 분석 결과는 빈 채로 처리 가능 | UI 메시지로 알 수 있음. |
| **Health check 실패 (Zombie)** | initializeEngine 내부에서 예외 throw → 호출부에서 처리. onPreparationCompleted에서는 CPU 레벨 상승/폴백 등 시도 | 정상. |
| **바인딩 타임아웃 (15초)** | EngineRepository에서 예외 → initializeEngine 실패로 전파 | 사용자에게 준비 실패 등으로 표시됨. |

---

## 4. 세션·엔진 매핑 일관성

- **analyzingSession**: 현재 “활성” 세션. ANALYSIS / PLAY / OFFLINE.
- **controllerFor(session)**: ANALYSIS → analysisEngineController (KataGoService), PLAY → playEngineController (KataGoPlayService).
- **useCaseFor(session)**: ANALYSIS → analysisUseCase, PLAY → playAnalysisUseCase. 각 useCase는 해당 controller에 고정.
- **triggerAnalysis**에서 사용하는 엔진:
  - `analysisSessionSnapshot == Session.ANALYSIS` → analysisUseCase (분석 엔진).
  - PLAY+HINT/DEEP → analysisUseCase (참고용 분석 엔진).
  - PLAY 일반 → playAnalysisUseCase (대국 엔진).
- **일관성**: ANALYSIS 화면에서 분석할 때는 항상 ensureAnalysisEngineReadyForHint로 **분석 엔진**을 보장하므로, onPreparationCompleted 레이스로 PLAY만 초기화된 경우에도 분석 시점에 분석 엔진이 채워짐.

---

## 5. 권장 추가 점검(선택)

1. **onPreparationCompleted에서 “모델 준비 완료 시점”의 세션 고정**  
   - 현재는 job **실행 시점**의 analyzingSession으로 초기화.  
   - “모델 준비가 끝난 순간”의 세션을 저장해 두고, 그 세션 엔진만 초기화하도록 바꾸면 레이스 가능성을 더 줄일 수 있음 (선택 사항).

2. **isModelReady의 세션별 분리**  
   - 현재는 전역 플래그 하나. 세션별로 “분석 엔진 준비됨 / 대국 엔진 준비됨”을 나누면, 전환 후 잘못된 준비 상태를 참조할 일이 줄어듦. (구조 변경이 커서 추후 검토 권장.)

3. **로그**  
   - 이미 추가된 항목: initializeEngine 시작 시 session, ensureAnalysisEngineReadyForHint 호출 시 everInit, 빈 결과 시 totalVisits/session/isModelReady.  
   - 필요 시 “setActiveSession 진입/종료”, “onPreparationCompleted 진입 시 analyzingSession” 로그를 추가하면 디버깅에 유리.

---

## 6. 요약

- **엔진 초기화 진입점**: startInitializationSequence, startEngineAfterLocaleSelected, setActiveSession, onPreparationCompleted, restartEngineForNewConfig, ensureAnalysisEngineReadyForHint, ensurePlayEngineReadyForLevel.
- **모드·이벤트**: 앱 기동, 모드 전환, 분석 요청, 대국 시작, 설정 변경, 오프라인 전환을 위 흐름으로 정리했고, 각각에서 “어떤 세션 엔진을 언제 초기화하는지”와 “실패 시 트리거/UI 동작”을 맞춰 점검함.
- **돌발 상황**: 빠른 탭 전환, 초기화 실패, 설정 변경 중복, Health check 실패, 바인딩 타임아웃 등에 대한 동작과 보완(분석 모드 ensureAnalysisEngineReadyForHint, setActiveSession 시 초기화 실패 시 트리거 스킵)을 반영함.
- **엔진 초기화 구조**와 **연관 로직(세션, useCase, controller)**은 서로 어긋나지 않도록 위 표와 매핑을 유지하면 됨.
