# 엔진 작동 관련 함수·로직 효율성 검토

엔진 초기화, 중지, 재시작, 세션 전환, 대국 모드 기력별 엔진 교체 등 **엔진 작동에 관여하는 모든 경로**를 정리하고 효율성·정확성을 평가한 문서입니다.

---

## 1. 아키텍처 요약

### 1.1 이중 엔진 구조

| 구분 | 서비스 | EngineRepository | EngineController | 용도 |
|------|--------|------------------|------------------|------|
| 분석 | `KataGoService` | `analysisEngineRepository` | `analysisEngineController` | 분석 탭, HINT/참고도 |
| 대국 | `KataGoPlayService` | `playEngineRepository` | `playEngineController` | 대국 모드(PLAY) |

- **프로세스 분리**: 분석/대국 각각 별도 프로세스에서 동작. 메모리·크래시 격리.
- **현재 세션에 따른 사용**: `engineController`는 `controllerFor(analyzingSession)`으로 **현재 세션**의 컨트롤러만 반환. 분석 요청·초기화는 모두 이 컨트롤러 경유.

### 1.2 주요 진입점

| 진입점 | 트리거 | 대상 엔진 | 동작 |
|--------|--------|-----------|------|
| 앱 기동 | `startEngineAfterLocaleSelected()` → `startInitializationSequence()` | 분석 | 모델 준비 후 `onPreparationCompleted()` → **분석 엔진만** 초기화 |
| 세션 전환 | `setActiveSession(ANALYSIS|PLAY|OFFLINE)` | 전환 **후** 세션 | 상태 저장/복원, 필요 시 config 갱신 + **shutdown + init** |
| 대국 시작 | `PlayViewModel.startGame()` → `ensurePlayEngineReadyForLevel(level)` | 대국 | `setActiveSession(PLAY)` 후 `isEngineReady` 폴링 |
| HINT 준비 | `ensureAnalysisEngineReadyForHint(boardSize)` | 분석 | 분석 엔진만 바인드/초기화, play.cfg 미갱신 |
| 설정 변경 | `restartEngineForNewConfig()` | 현재 세션 | 현재 `engineController`만 stop + 재초기화 |
| 크래시/폴백 | `onPreparationCompleted` 내 catch, `EngineRepository` CPU 폴백 | 현재 세션 | 재시도·CPU 전환·ensureConfigFile 후 재초기화 |

---

## 2. 경로별 상세

### 2.1 앱 기동 → 첫 엔진 초기화

```
startEngineAfterLocaleSelected()
  → startInitializationSequence()
      → (모델 준비 또는 스킵)
      → onPreparationCompleted()
          → GPU/CPU 체크, ensureConfigFile(둘 다)
          → initializeEngine(forceLowMem = false)
```

- **효율**: 앱 시작 시 **분석 엔진 1개만** 초기화(`analyzingSession` 기본값 ANALYSIS). 대국 엔진은 첫 PLAY 전환 시 지연 바인딩. 중복 초기화 없음.
- **주의**: `onPreparationCompleted`에서 사용하는 `engineController`는 아직 세션 변경 전이므로 **분석** 엔진이 맞음.

### 2.2 세션 전환 (setActiveSession)

```
setActiveSession(session)
  → 이전 세션 상태 저장
  → engineOpJob 취소, engineOpMutex.withLock {
      stopAllAnalysis(양쪽 다)
      analyzingSession = session   // ★ 여기서 현재 세션이 바뀜
      상태 복원
      desired = desiredEngineProfile(session, ...)
      if (currentEngineProfile != desired) {
        ensureConfigFile(전환 대상 한쪽만)
        engineController.shutdown()  // ★ 현재 = 새 세션의 컨트롤러
        delay(350)
        initializeEngine(...)
      }
      triggerAnalysis(...)
    }
```

- **문제 1 — shutdown 대상 오류**:  
  `analyzingSession = session` 이후에 `engineController`를 쓰므로, **새 세션**의 엔진을 shutdown 함.  
  - PLAY → ANALYSIS 전환 시: **분석** 엔진을 shutdown 한 뒤 분석 엔진을 다시 init → 불필요한 재시작 + **대국 엔진(PLAY)은 종료되지 않아 메모리 유지**.
  - ANALYSIS → PLAY 전환 시: **대국** 엔진을 shutdown 한 뒤 대국 엔진 init → 의도에 맞음(이전에 분석만 켜져 있었으므로 play 쪽 shutdown은 no-op일 수 있음).
- **권장**: 세션 전환 시 **이전 세션(previousSession)** 의 컨트롤러를 shutdown 해야 함. `controllerFor(previousSession).shutdown()`.

### 2.3 대국 모드 기력(레벨) 변경 → 엔진 교체

```
PlayViewModel.startGame()
  → ensurePlayEngineReadyForLevel(selectedLevelIndex)
      → setPlayLevelForModelSelection(levelIndex)  // repo에 레벨 저장
      → setActiveSession(PLAY)
  → ensureConfigFile(play만)
  → newGame(), ...
```

- **문제 2 — 레벨만 바꾼 경우 재초기화 누락**:  
  `desiredEngineProfile(session, boardSize, forceLowMem)`에는 **playLevelIndex(또는 모델 경로)** 가 포함되어 있지 않음.  
  이미 PLAY 세션이고, 보드/CPU/forceLowMem만 같으면 `currentEngineProfile == desired`가 되어 **재초기화를 건너뜀**.  
  그 결과:
  - `play.cfg`는 `ensureConfigFile(play)`로 새 레벨이 반영되지만,
  - **이미 떠 있는 대국 엔진**은 이전 레벨의 설정/모델(B10 vs g170 등)로 계속 동작할 수 있음.
- **권장**: PLAY용 프로필에 `playLevelIndex`(또는 사용 중인 모델 경로)를 넣어, 레벨 변경 시 `desired != current`가 되도록 하여 재초기화가 일어나게 함.

### 2.4 ensurePlayEngineReadyForLevel 대기 방식

- **구현**: `setActiveSession(PLAY)` 호출 후 `isEngineReady`를 200ms 간격으로 최대 약 125초까지 폴링.
- **효율**: 폴링은 단순하지만, 준비 완료 시점까지 불필요한 대기가 있을 수 있음.  
  `StateFlow`/`SharedFlow`로 `isEngineReady`를 구독해 한 번만 반응하도록 하면 더 효율적(선택 개선).

### 2.5 대국 시작 시 ensureConfigFile 중복

- `setActiveSession(PLAY)` 내부에서 이미 `ensureConfigFile(..., writePlayConfig = true)` 호출.
- `startGame()`에서 다시 `ensureConfigFile(..., writePlayConfig = true)` 호출.
- **효율**: 동일 설정으로 두 번 쓰기. 기능상 문제는 없으나, 한 곳에서만 갱신하도록 정리하면 불필요한 I/O 제거 가능.

### 2.6 HINT용 분석 엔진 준비

- `ensureAnalysisEngineReadyForHint(boardSize)`:  
  `desiredEngineProfile(ANALYSIS, ...)` 비교 후, 필요 시 `ensureConfigFile(analysis만)`, `analysisEngineController.initEngine(...)` 만 호출.
- **효율**: play.cfg를 건드리지 않아 대국 설정과 분리되어 있음. 적절함.

### 2.7 엔진 재시작 (restartEngineForNewConfig)

- `restartEngineForNewConfig()`:  
  현재 세션의 `engineController`에 대해 `stopAnalysis` 후 `initializeEngine(forceLowMem = false)`.
- **효율**: 현재 사용 중인 엔진만 재시작. 세션 전환 없음. 적절함.

### 2.8 크래시·폴백 경로

- **onPreparationCompleted** 내부:  
  CPU 강제 전환 → ensureConfigFile(둘 다) → `engineController.shutdown()` → delay → `initializeEngine()`.  
  CPU 모드에서 실패 시 레벨 상향 후 ensureConfigFile(둘 다) → 재초기화.  
  OOM 시 Safe Mode 재시도, 실패 시 CPU 폴백 → ensureConfigFile(둘 다) → shutdown → 재초기화.
- **EngineRepository**:  
  서비스 크래시 시 CPU 폴백 플로우에서 `ensureConfigFile(context)`(둘 다) 후 서비스 재시작.
- **효율**: 모두 **현재 사용 중인** 엔진(현재 세션의 `engineController`)에 대해 동작. 세션 전환 로직과 분리되어 있어 일관적.

### 2.9 직렬화·동기화

- **EngineController**: 단일 채널 + 백로그로 모든 호출 직렬화. `init`/`analyze`/`shutdown`이 동시에 들어가지 않음. 적절함.
- **EngineRepository**: `initMutex`로 `initEngine`/`shutdown`/`ensureServiceBound` 직렬화. 바인딩 타임아웃 15초.
- **EngineViewModel**: `engineOpMutex`로 세션 전환·초기화 등 엔진 작업 직렬화. `engineOpJob` 취소로 이전 전환 취소 가능.

---

## 3. 요약 표

| 항목 | 평가 | 비고 |
|------|------|------|
| 앱 기동 시 분석 엔진만 초기화 | ✅ 효율적 | 대국 엔진 지연 바인딩 |
| 세션 전환 시 shutdown 대상 | ❌ 오류 | 새 세션 엔진을 shutdown 함 → **이전 세션** shutdown 해야 함 |
| 대국 레벨 변경 시 재초기화 | ❌ 누락 | EngineProfile에 레벨 미포함 → 레벨만 바꾸면 재init 안 함 |
| ensurePlayEngineReadyForLevel 폴링 | ⚠️ 개선 여지 | Flow 구독으로 대체 가능 |
| startGame 시 ensureConfigFile 이중 호출 | ⚠️ 사소 | 한 번으로 통합 가능 |
| HINT용 분석만 준비 | ✅ 적절 | analysis만 갱신 |
| restartEngineForNewConfig | ✅ 적절 | 현재 세션만 재시작 |
| 크래시/폴백 경로 | ✅ 일관 | 현재 엔진만 대상 |
| Controller/Repository 직렬화 | ✅ 적절 | 레이스 방지 |

---

## 4. 권장 수정 사항 (적용 완료)

1. **setActiveSession** — 적용됨  
   - `engineController.shutdown()` → `controllerFor(previousSession).shutdown()` 으로 변경.  
   - 이전 세션 엔진만 종료 후, 새 세션 엔진만 `initializeEngine()`로 초기화.

2. **대국 레벨 변경 시 재초기화 보장** — 적용됨  
   - `EngineProfile`에 `playLevelIndex: Int?` 추가.  
   - `desiredEngineProfile`에서 PLAY일 때 `settingsRepository.getPlayLevelIndex()` 반영.  
   - `initializeEngine` 성공 시에도 동일 필드로 저장. 레벨만 바꿔도 `current != desired`가 되어 재초기화됨.

3. **(선택)** `ensurePlayEngineReadyForLevel`:  
   - `isEngineReady`를 `StateFlow`로 구독해 준비 시점에 한 번만 완료 처리. 폴링 제거.

4. **(선택)** `startGame`:  
   - `ensureConfigFile(play)` 호출을 한 곳으로 통합(예: `setActiveSession(PLAY)` 쪽만 사용하거나, `startGame`에서만 호출하고 setActiveSession에서는 제거).

이 문서는 `LOGIC_CONFLICTS_AND_INTERFERENCE_AUDIT.md`, `PLAY_MODE_BOT_SPEC.md`와 함께 참고하면 됩니다.
