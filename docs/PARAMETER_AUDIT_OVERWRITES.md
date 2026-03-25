# 설정 파라미터 감사: 덮어쓰기·로직 차단 정리

대국모드(play) 관련 설정이 **덮어쓰이거나** **로직/코드에 막혀 제대로 반영되지 않는** 구간을 전체 함수·경로 기준으로 정리한 문서입니다.

---

## 1. 수정 완료된 항목 (이번 감사에서 반영)

### 1.1 chosenMoveTemperature (play.cfg 기록 시 상한 1.0으로 묶임)

- **위치**: `AssetInstaller.kt` — play.cfg 작성 시 `getChosenMoveTemperature().coerceIn(0.0f, 1.0f)` 사용.
- **문제**: 사양은 0.05~2.2(레벨 30에서 2.2)인데, 1.0 초과 값이 **전부 1.0으로 잘려서** 기록됨.
- **수정**: `coerceIn(0.05f, 2.2f)` 로 변경해 레벨별 2.2까지 play.cfg에 기록되도록 함.

### 1.2 대국모드 분석 예산(착수당 시간) 3~6초로 고정

- **위치**: `EngineViewModel.kt` — `triggerAnalysis()` 내  
  `playBudgetMs = settingsRepository.getMaxTime().coerceIn(3f, 6.0f) * 1000`.
- **문제**: `setMaxTime(1)`(레벨 30) 또는 1.6~2초(레벨 7 등)로 저장해도, **항상 3~6초**로만 사용됨.
- **수정**: PLAY 세션일 때는 `getMaxTime()` 을 그대로 사용하도록 변경  
  (`playBudgetMs = settingsRepository.getMaxTime() * 1000`).  
  `getMaxTime()` 은 이미 `SettingsRepository` 에서 1~6초로 제한됨.

---

## 2. 파라미터별 경로·덮어쓰기 요약

### 2.1 play.cfg에 기록되는 값 (AssetInstaller)

| 파라미터 | 설정 소스 | 기록 시 클램프 | 비고 |
|----------|-----------|----------------|------|
| chosenMoveTemperature | SettingsRepository | **0.05~2.2** (수정 반영) | 이전 0~1에서 상한 확대 |
| wideRootNoise | SettingsRepository | 0~0.2 | 사양과 일치 |
| cpuctExploration | SettingsRepository | 0~10 | 사양과 일치 |
| cpuctExplorationLog | SettingsRepository | 0~10 | 사양과 일치 |
| cpuctExplorationBase | SettingsRepository | 10~100000 | 사양과 일치 |
| avoidRepeatedPatternUtility | SettingsRepository | 0~2 | 사양과 일치 |
| staticScoreUtilityFactor | SettingsRepository | 0~1 | 사양과 일치 |
| avoidRepeatedPattern | SettingsRepository | bool | true 고정 |
| maxTime (play.cfg 파일) | 고정 | 30.0 | 실제 예산은 Java에서 전달하므로 무관 |
| maxVisits | 고정 | 100000 | 실제 호출 시 jMaxVisits=0 이면 시간 제한만 사용 |

### 2.2 대국모드 분석 호출 경로 (analyzeTop4)

- **EngineViewModel.triggerAnalysis**  
  - PLAY일 때 `budgetMs = getMaxTime() * 1000` (수정 반영).  
  - `budgetMs = targetMs.coerceAtLeast(1000L)` → 최소 1초.
- **AnalysisUseCase.analyze**  
  - `maxTimeSeconds = budgetMs / 1000`, `maxVisits`(PLAY에서는 0) 전달.
- **NativeBridge.analyzeTop4**  
  - `jMaxTimeSeconds`, `jMaxVisits` 수신.

**JNI `analyzeTop4()` (katago_jni.cpp):**

- `localParams = g_globalParams` (play 세션일 때는 play.cfg에서 로드된 값).
- **덮어쓰는 항목** (의도적):  
  `maxTime`(요청값), `maxVisits`(jMaxVisits>0일 때), `maxPlayouts`, `lagBuffer=0`,  
  `useLcbForSelection=true`, `lcbStdevs=5.0`, `numVirtualLossesPerThread=1.0`.
- **덮어쓰지 않는 항목** (이전 수정 반영):  
  `cpuctExploration`, `cpuctExplorationLog`, `cpuctExplorationBase`,  
  `wideRootNoise`, `fpuReductionMax`, `rootFpuReductionMax` → **config 그대로 사용**.

### 2.3 play.cfg에 없지만 Setup 기본값으로 들어가는 항목

- **rootNoiseEnabled**: play.cfg에 없으면 `Setup::loadSingleParams` 기본값 false.
- **fpuReductionMax**, **rootFpuReductionMax**: play.cfg에 없으면 Setup 기본값 사용.  
  → JNI에서 더 이상 덮어쓰지 않으므로, config/기본값이 검색에 반영됨.

### 2.4 다른 네이티브 경로 (대국 검색과 분리)

- **nativeGetPolicy** (정책만):  
  `localParams = g_globalParams` 후 `maxVisits=1`, `maxTime=0.02` 등으로 덮어씀.  
  → 정책 전용 경로이므로 play 분석 설정과 무관.
- **getOwnershipHeatmap**:  
  `localParams = g_globalParams` 후 threads/visits/time만 덮어씀.  
  → 소유권 전용, play 분석과 무관.
- **getLeadEstimation**:  
  동일하게 time/visits만 덮어씀.  
  → 수치 추정 전용, play 분석과 무관.

이들 경로는 **대국모드 MCTS/설정과 분리**되어 있어, play 파라미터가 “막히는” 구간은 아님.

### 2.5 tuned.cfg (분석 쪽 동기화)

- **EngineViewModel** 내 tuned.cfg 동기화 시  
  `chosenMoveTemperature = getChosenMoveTemperature().coerceIn(0.0f, 1.0f)` 사용.
- **의도**: 분석/튜닝용 설정은 0~1 범위 유지.  
  play.cfg는 AssetInstaller에서만 쓰며 0.05~2.2로 이미 수정됨.

---

## 3. 설정 적용 순서 (play 세션)

1. **PlayViewModel.startGame()**  
   레벨별로 SettingsRepository에  
   `setMaxTime`, `setChosenMoveTemperature`, `setWideRootNoise`, `setCpuct*`, `setAvoidRepeatedPatternUtility`, `setStaticScoreUtilityFactor` 등 설정.
2. **ensurePlayEngineReadyForLevel()**  
   PLAY 엔진 준비(모델/세션).
3. **AssetInstaller.ensureConfigFile(writePlayConfig = true)**  
   위 저장소 값으로 **play.cfg** 생성(이때 chosenMoveTemp 0.05~2.2 반영).
4. **엔진 초기화**  
   play.cfg 경로로 `initEngine` 호출 시 `g_globalParams = loadSingleParams(play.cfg, ...)`.
5. **analyzeTop4**  
   `localParams = g_globalParams` + 시간/방문/lagBuffer/LCB 등만 덮어쓰고, cpuct·wideRootNoise·fpu 등은 **config 유지**.

**참고**: play.cfg 기록이 `ensurePlayEngineReadyForLevel` **다음**에 이루어지므로, 레벨 변경 직후 첫 수는 **이전 play.cfg**로 검색될 수 있음.  
(문서 `PLAY_STRENGTH_DIAGNOSIS.md` §3.3 참고. 필요 시 기록 순서 변경 또는 재초기화 검토.)

---

## 4. 요약

- **이번에 고친 것**:  
  (1) play.cfg **chosenMoveTemperature** 상한 1.0 → 2.2,  
  (2) 대국모드 **착수당 시간** 3~6초 고정 제거(1~6초가 저장값대로 적용).
- **이미 반영된 것**:  
  analyzeTop4에서 cpuct·wideRootNoise·fpu 계열은 **덮어쓰지 않음** → config(play.cfg)가 그대로 MCTS에 반영됨.
- **의도적으로 덮어쓰는 것**:  
  시간/방문/lagBuffer/LCB·가상손실 등만 JNI에서 고정. 나머지 강도·탐색 관련은 config 우선.
- **다른 경로**:  
  policy/ownership/lead 전용 호출은 별도 SearchParams로 제한적 덮어쓰기만 하며, play 설정과 분리됨.

이후에도 새 파라미터를 넣을 때는  
**설정 저장소 → play.cfg 기록(클램프) → initEngine(loadSingleParams) → analyzeTop4(덮어쓰기 여부)** 순으로 한 번씩 확인하면 됨.
