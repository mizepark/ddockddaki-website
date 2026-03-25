# 전환·분석·추천수·봇 착수 — 전체 테스트 및 자동 재시도

세션 전환, 앱 재실행, 기능 전환 등 **모든 변환 후**에  
엔진 중단·분석 의도 수락·분석 모드 추천수 표시·봇 착수가 정상인지 **미리 테스트**하고,  
실패 시 **앱이 파악해 자동 재시도**할 수 있는지 정리한 문서입니다.

---

## 1. 앱에 존재하는 전환·중단·의도 제출 목록

### 1.1 분석 의도 제출 원인 (IntentUpdateSource)

| 원인 | 발생 시점 | 전환 성격 | 의도 제출 후 기대 |
|------|-----------|-----------|-------------------|
| **SET_ACTIVE_SESSION** | 탭 전환(분석↔대국↔오프라인) 완료 시 | 세션 전환 | 새 세션 국면에 대해 RunNow로 분석 수락 |
| **INIT_ENGINE** | 엔진 초기화/앱 시작 후 | 앱/엔진 생명주기 | 현재 국면 분석 의도 수락 |
| **RESTORE_SAVED_GAME** | 저장 게임 복원 | 게임/UI 전환 | 복원된 국면 분석 |
| **SNAPSHOT_RESTORE** | 스냅샷 복원 | 게임 전환 | 복원 국면 분석 |
| **NEW_GAME** | 새 게임 시작 | 게임 전환 | 빈 국면 분석 |
| **PLAY_MOVE** | 착수 후 | 게임 진행 | 새 국면 분석 |
| **UNDO** | 무르기 후 | 게임 진행 | 무른 국면 분석 |
| **JUMP** | 수순 점프(다양한 진입점) | 게임 진행 | 점프한 국면 분석 |
| **LOAD_SGF** | SGF 로드 후 | 게임 전환 | 로드 국면 분석 |
| **SET_TARGET_LATENCY** | 지연 설정 변경 | 설정 전환 | 동일 국면 재분석 |
| **TRIGGER_QUICK** | 퀵 분석 요청(대국/분석 탭) | 사용자 트리거 | RunNow 수락 후 분석 |
| **TRIGGER_HINT** | 힌트 모드 요청 | 모드 전환 | 힌트 예산으로 분석 |
| **CONFIRM_DEEP** | 심층 분석 확인 | 모드 전환 | 심층 예산으로 분석 |
| **RESULT_EMPTY_RETRY** | 빈 분석 결과 수신 후 | 내부 재시도 | 동일 국면 재분석(재시도 토큰 증가) |

### 1.2 엔진 분석 중단이 일어나는 지점

| 지점 | 조건 | 이후 기대 |
|------|------|-----------|
| **onSessionSwitched (ViewModel)** | 탭 전환 시 | `analysisJob?.cancel()`, `pendingAnalysis = false` → 새 세션 의도 수락 가능 |
| **EngineOrchestrator SwitchSession** | 탭 전환 시 | Orchestrator만 `analysisJob?.cancel()`; ViewModel 정리 필수(위) |
| **onLowMemory (KataGoApp)** | 시스템 메모리 부족 | `stopAnalysis(ANALYSIS/PLAY)` 호출; 재진입 시 INIT_ENGINE 등으로 의도 재제출 |
| **ViewModel onCleared** | 화면/ViewModel 파괴 | `stopAllAnalysis()`; 앱 재실행 시 새 초기화 |
| **KataGoService 콜백 실패** | Binder/콜백 오류 | 내부 에러 카운팅·복구; 필요 시 재연결 |

### 1.3 앱/기능 전환 요약

- **세션 전환**: 분석 탭 ↔ 대국 탭 ↔ 오프라인 (setActiveSession → SwitchSession → onSessionSwitched)
- **앱 재실행**: Process death 후 재시작 → AppStart → ensureReady → INIT_ENGINE 의도
- **설정/기능 전환**: 레벨 변경, 지연 변경, 힌트/심층 모드, SGF 로드, 새 게임, Undo/Jump
- **메모리 압박**: onLowMemory / onTrimMemory → 분석 중단 또는 캐시 정리

---

## 2. 전환 후 기대 동작 (정상 기준)

- **엔진 중단**: 세션 전환 시 기존 분석 job만 취소되고, 새 세션에서 새 분석이 스케줄러에 의해 수락되어 실행된다.
- **의도 수락**: 위 각 IntentUpdateSource 발생 후, 스케줄러가 `getRunnerState()`가 비어 있으면(또는 보호 구간 이후) **RunNow**를 반환하고, 분석이 한 번 실행된다.
- **분석 모드 추천수 표시**: 분석이 끝나 `handleAnalysisResult`에서 suggestions가 반영되면, `topSuggestions` / `lastTotalVisits` / `isInitialAnalysisReady`가 UI에 반영되어 추천수가 표시된다.
- **봇 착수**: 대국 턴에서 `isBotMoveReady(engine)`가 true가 되면(분석 완료 또는 lastTotalVisits≥1) 착수 시도하고, 실패 시 재시도·연속 실패 시 `maybeKickQuickAnalysis`로 분석 재시작한다.

---

## 3. 미리 테스트해야 할 것 (설치 전·설치 후)

### 3.1 단위 테스트 (설치 전)

| 테스트 | 검증 내용 | 파일 |
|--------|-----------|------|
| 스케줄러 계약 | runner state 비어 있으면 RunNow; 살아 있으면 Defer | `AnalysisSchedulerSessionSwitchTest` |
| 결과 게이트/스냅샷 | epoch·requestId·movesHash 불일치 시 결과 폐기 | `EngineEventSequenceSimulationTest`, `SnapshotValidatorTest` |
| 빈 결과 재시도 | 동일 movesHash 기준 재시도 횟수·토큰 | (SafetyLimits + handleAnalysisResult 로직) |

**추가 권장**:  
- 각 **IntentUpdateSource**가 호출되는 경로를 시뮬레이션하는 테스트(ViewModel 또는 하네스에 이벤트 주입 → `updateAnalysisIntent` 호출 여부 또는 `getRunnerState` 정리 여부).
- **세션 전환 직후** `getRunnerState()`가 `(isAnalysisRunning=false, pendingAnalysis=false)`를 반환하는지 검증하는 테스트(ViewModel이 테스트용 getter를 노출하거나, 의도 제출 → RunNow 간접 검증).

### 3.2 시나리오/통합 테스트 (설치 전, 가능한 범위)

- **전환 시퀀스**: [분석 탭에서 분석 실행] → [대국 탭 전환] → [대국 국면에 대한 의도 제출] → 스케줄러가 RunNow 반환하는지(또는 `onRunRequested` 호출되는지).
- **의도 제출 경로**: PLAY_MOVE, UNDO, JUMP, LOAD_SGF, NEW_GAME, RESTORE_SAVED_GAME, SET_ACTIVE_SESSION 등 각각에서 `updateAnalysisIntent`가 호출된 뒤, 한 번은 분석이 실행되는 경로가 있는지(모킹된 엔진으로).

### 3.3 기기/수동 테스트 (설치 후)

- **세션 전환**: 분석 탭에서 분석 중 → 대국 탭 전환 → 봇 턴에서 분석이 돌고 착수가 되는지.
- **앱 재실행**: 앱 종료 후 재실행 → 분석 탭/대국 탭 각각에서 추천수·봇 착수 정상 여부.
- **기능 전환**: Undo, Jump, SGF 로드, 새 게임, 힌트/심층 모드 전환 후 추천수·봇 착수.
- **메모리 압박**: onLowMemory 시뮬레이션(가능하다면) 후 재진입 시 분석 의도 수락·추천수 표시.

---

## 4. 앱이 실패를 파악하고 자동 재시도하는지

### 4.1 이미 구현된 자동 재시도·복구

| 상황 | 처리 | 위치 |
|------|------|------|
| **빈 분석 결과** (suggestions 0개) | 동일 movesHash 기준 재시도(최대 제한), `RESULT_EMPTY_RETRY`로 의도 재제출 | `EngineViewModel.handleAnalysisResult` |
| **봇 착수 검증 실패** (착수했는데 보드 반영 안 됨) | 같은 턴에서 지연 후 재시도, 최대 BOT_MOVE_MAX_ATTEMPTS | `PlayViewModel` bot move job |
| **봇 연속 실패** (BOT_MAX_CONSECUTIVE_FAILURES 미만) | `maybeKickQuickAnalysis(engine)`로 분석 재시작 | `PlayViewModel` |
| **엔진 초기화 실패** | ensureReady 재시도, CPU 모드 변경 후 재시도 | `EngineInitCoordinator`, `EngineRepository` |
| **AIDL/엔진 호출 실패** | 최대 1회 재시도 | `EngineCalls` |

### 4.2 “추천수 안 나옴” / “봇이 안 둠”을 앱이 즉시 파악해 재시도할 수 있는지

- **현재**:  
  - **추천수**: “의도는 냈는데 오래 돼도 추천 0개”를 **직접** 감지하는 주기적 검사는 없음. 다만 **빈 결과가 엔진에서 오면** 재시도는 함(RESULT_EMPTY_RETRY).  
  - **봇**: `isBotMoveReady`가 false인 동안은 대기하다가, 준비되면 착수 시도 → 실패 시 재시도 → 연속 실패 시 `maybeKickQuickAnalysis`로 분석을 다시 킥함. 즉 “봇이 안 둠” 중 **분석이 한 번이라도 끝났는데 착수가 안 되는 경우**는 재시도·분석 재킥으로 일부 복구됨.  
  - **스케줄러가 의도를 안 받는 경우**(예: 세션 전환 후 state가 안 비워져서 계속 Defer)는, **의도는 계속 제출**되지만 **실행만 안 되는** 상태라, “추천 0개”로 들어오는 결과가 없어서 RESULT_EMPTY_RETRY도 동작하지 않음. 이건 세션 전환 시 ViewModel 정리(onSessionSwitched)로 해결한 부분.

- **가능한 확장 (선택)**  
  - **“분석 의도는 냈는데 N초 동안 실행도 안 되고 추천도 0”**인 경우를 주기적으로 검사해,  
    - `getRunnerState()`가 이미 비어 있고(`!isAnalysisRunning && !pendingAnalysis`),  
    - 현재 세션에서 `lastTotalVisits == 0`이고,  
    - 마지막 의도 제출 시각이 N초(예: 15초) 이상 이전이면  
    **의도를 한 번 다시 제출**하는 “분석 스티치 복구”를 넣을 수 있음.  
  - 이때 **중복 실행**을 막기 위해 쿨다운(예: 30초에 한 번)과 “실제로 분석 중이 아닐 때만” 조건이 필요함.  
  - 구현 위치 후보: `EngineViewModel`의 주기적 체크 또는 분석 탭/대국 탭이 포그라운드일 때만 도는 작은 “health” job.

요약하면, **지금도** 빈 결과 재시도·봇 착수 재시도·연속 실패 시 분석 재킥으로 **일부** 자동 재시도가 되고, **“의도는 냈는데 스케줄러가 계속 거절해서 추천이 영원히 0”** 같은 상황을 앱이 **직접** 감지해 즉시 재시도하는 로직은 없지만, **선택적으로** “N초 동안 추천 0 + 분석도 안 돌고 있음” 조건으로 의도 재제출을 넣을 수 있습니다.

---

## 5. 체크리스트 (전환 후 정상 여부)

- [ ] **세션 전환** 후: 새 탭에서 분석 의도가 RunNow로 수락되어 한 번은 실행되는가?
- [ ] **앱 재실행** 후: 분석 탭/대국 탭에서 추천수가 표시되는가? 봇이 착수하는가?
- [ ] **Undo/Jump/로드/새 게임/설정 변경** 후: 해당 국면에 대해 분석이 돌고 추천수/봇이 정상인가?
- [ ] **봇 턴**에서: 분석 완료 후 착수 시도 → 실패 시 재시도 → 연속 실패 시 분석 재킥이 동작하는가?
- [ ] **빈 분석 결과** 시: 동일 국면 재시도(RESULT_EMPTY_RETRY)가 제한 횟수 내에서 동작하는가?

이 체크리스트를 단위 테스트·시나리오 테스트·기기 테스트에 매핑해, 전환·중단·의도 수락·추천수·봇 착수를 **미리** 검증하고, 필요 시 “추천 0 + 분석 미실행” 감지 재시도까지 확장할 수 있습니다.
