# 엔진 구조 단순화 (Engine Facade) 요약

simple_pad_app 참고로, **엔진 작동 구조만** 단순화했습니다.  
UI·분석/대국/오프라인/게임 모드·참고도·힌트·결제·기력/방문수/파라미터 등 **모든 설정값과 기능은 그대로** 두었습니다.

---

## 1. 추가된 파일

| 파일 | 역할 |
|------|------|
| `engine/EngineFacade.kt` | 엔진 퍼사드 **인터페이스** (ensureReady, runAnalysis, stopAnalysis, restart, close) |
| `engine/KataGoEngineFacade.kt` | **구현체**: 단일 Mutex로 모든 엔진 호출 직렬화 → 레이스 제거, B10(분석)/B6(대국), CPU 전용 |
| `engine/EngineInitCoordinator.kt` | **단일 초기화 경로**: 모델 설치(prepareModelWithRetry) → CPU 강제 → 퍼사드 ensureReady → 재시도. ViewModel은 코디네이터만 호출하고 UI 상태 반영만 담당. |
| `engine/AnalysisStateMachine.kt` | **분석 의도 수집·필터·실행 결정만 담당**. 분석 트리거는 이 상태머신 루프에서만 결정되고, 실제 실행은 `onRunRequested` 콜백으로 ViewModel에 위임. |
| `engine/EngineSessionOrchestrator.kt` | **세션 전환·저장/복원 전담**. 캡처 → persist 콜백 → 대상 세션 복원(캐시/디스크). ViewModel은 오케스트레이터 호출 후 반환된 UI 상태만 반영. |
| `engine/EngineOrchestrator.kt` | **명시적 4상태(Idle/Ready/Analyzing/Switching) + 단일 이벤트 큐**. AppStart/StartAnalysis/SwitchSession/ChangeConfig/AnalysisDone을 Channel로 직렬 처리. ViewModel은 post()만 하고 UI는 콜백으로 반영. |

---

## 2. 동작 방식

- **단일 진입점**: 분석·대국 요청은 전부 `EngineFacade`를 통해서만 수행.
- **직렬화**: `KataGoEngineFacade` 내부에서 하나의 `Mutex`로 `ensureReady`, `runAnalysis`, `stopAnalysis`, `restart`를 감싸서 **동시에 하나의 엔진 작업만** 실행.
- **초기화 한 경로**: 앱 시작·재시작·대국 준비 시 모두 `EngineInitCoordinator.ensureEngineReady()`만 호출.  
  코디네이터 내부에서 모델 설치(필요 시) → CPU 강제 → `engineFacade.ensureReady()` → 실패 시 재시도.  
  분석 엔진(B10)·대국 엔진(B6)을 **한 번에** 초기화하고, 세션 전환 시에는 **shutdown/재초기화 없음**.
- **CPU 전용**: 퍼사드에서 `settingsRepository.setCpuModeEnabled(true)` 강제.
- **모델**: 분석 = B10, 대국 = B6만 사용 (기존 `AssetInstaller.getModelFile` / `getModelFileForPlayLevel` 유지).

---

## 3. EngineViewModel 변경 요약

- **초기화**: `startInitializationSequence()`에서 **한 경로만** 사용: `engineInitCoordinator.ensureEngineReady(app, 19)` 호출.  
  코디네이터가 모델 설치·CPU 강제·ensureReady·재시도를 담당하고, ViewModel은 결과(Success/Failure)에 따라 UI만 갱신.  
  `onPreparationCompleted()` 제거됨.
- **분석**: `triggerAnalysis` 경로에서 `useCaseFor(session).analyze(...)` 대신  
  `engineFacade.runAnalysis(session, moves, boardSize, budgetMs, ...)` 호출.  
  `ensureAnalysisEngineReadyForHint` 등 세션별 수동 초기화 호출 제거.
- **세션 전환**: `setActiveSession()`에서 **shutdown + 재초기화 제거**.  
  퍼사드가 이미 두 엔진 모두 초기화해 두므로, 세션 전환 시에는 상태 정리·`updateAnalysisIntent`만 수행.
- **중단**: `stopAllAnalysis` 및 각종 `controller.stopAnalysis(...)` 호출을  
  `engineFacade.stopAnalysis(Session.ANALYSIS/PLAY, requestId)` 로 통일.
- **설정 변경 재시작**: `restartEngineForNewConfig()`에서 `engineFacade.restart()` 후  
  `engineInitCoordinator.ensureEngineReady(app, 19, skipModelInstall = true)` 호출 (단일 경로 유지).
- **대국 엔진 준비**: `ensurePlayEngineReadyForLevel()`에서 `engineInitCoordinator.ensureEngineReady(..., skipModelInstall = true)` 호출.

---

## 4. KataGoApp

- `engineFacade: EngineFacade` 추가 (lazy `KataGoEngineFacade`).
- 기존 `analysisEngineController` / `playEngineController`는 유지 (퍼사드가 내부에서 사용).

---

## 5. 유지된 것

- 기력·방문수·파라미터·참고도/힌트·결제·오프라인/게임 모드 등 **모든 설정 및 기능**.
- `SettingsRepository`, `AssetInstaller`, `AnalysisUseCase`, `EngineController` 등 **기존 컴포넌트** (퍼사드가 이들을 사용).
- `PlayViewModel`, `AnalysisUiViewModel` 등 UI 쪽은 **EngineViewModel API만 사용**하므로 수정 없음.

---

## 6. 수정한 기타 사항

- CPU 전용 분기만 유지.
- **AssetInstaller**: `stripLegacyTuningLines` 미정의 오류 수정 (로컬 no-op 함수 추가).

---

이 구조로 **엔진 호출이 한 갈래로 모이고**, **초기화는 한 번·한 경로**만 수행되며, **세션 전환 시 shutdown/재초기화가 없어** 레이스 가능성이 줄어듭니다.

---

## 7. 퍼사드 단일 경로 원칙 (유지 사항)

- **레거시 경로 비활성화**: `initializeEngine()`는 첫 줄에서 `error()`로 즉시 실패. 호출 시 퍼사드 우회로 인한 레이스 재발을 막기 위함.
- **ensureAnalysisEngineReadyForHint**: 분석 엔진 수동 초기화 제거 → no-op. 분석은 앱 시작 시 `engineFacade.ensureReady()`에서만 초기화.
- **shutdown/close**: `shutdownAllEngines()`는 `engineFacade.restart()`만 호출. `onCleared()`에서는 `stopAllAnalysis()` → `shutdownAllEngines()` → `engineFacade.close()` 순으로만 정리. Controller 직접 `shutdown()`/`close()` 호출 제거.
- **EngineInitCoordinator**: CPU 강제·크래시 루프 처리·재시도는 코디네이터 내부에서만 수행. ViewModel은 코디네이터 호출 + 결과에 따른 UI 반영만 담당.
- **원칙**: 엔진 초기화·분석·중단·재시작은 반드시 `EngineFacade`로만 수행. EngineController 직접 호출을 추가하면 퍼사드 Mutex와 `engineOpMutex`가 따로 있어 동시성 충돌 가능성이 커짐.

---

## 8. 핵심 조건: 퍼사드 외 직접 엔진 호출 금지

**단순화 효과를 유지하려면 퍼사드 외 직접 엔진 호출 경로를 다시 만들지 않는 것이 핵심 조건이다.**

- **반드시 퍼사드만 쓸 것**: `initEngine` / `runAnalysis`(분석 실행) / `stopAnalysis` / `restart` / `close` 는 **오직 `EngineFacade`** 를 통해서만 호출.
- **금지**: `EngineController`(또는 `analysisEngineController` / `playEngineController`)에 대한 위 API 직접 호출. Application(예: `KataGoApp`의 onTrimMemory/onLowMemory)에서도 중단 시 **퍼사드** `stopAnalysis(Session, requestId)` 사용.
- **예외 없음**: 메모리 압박·저장/복원·설정 변경 등 어떤 경로에서도 init/analyze/stop/restart/close 는 퍼사드 한 곳으로만 진입.
- **검증**: 새 코드 추가 시 `controller.initEngine`, `controller.shutdown`, `controller.stopAnalysis`, `controller.analyzeTop4` 등 직접 호출이 생기지 않았는지 확인.
