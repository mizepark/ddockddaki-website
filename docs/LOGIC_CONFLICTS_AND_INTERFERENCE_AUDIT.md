# 로직 충돌·불필요한 간섭 감사

앱 전체의 함수·로직을 경로별로 분석해 **로직 간 충돌**이나 **불필요하게 간섭하는 구조**가 있는지 정리한 문서입니다.

---

## 1. 아키텍처 요약

- **엔진**: 분석용 `KataGoService`, 대국용 `KataGoPlayService` — **서로 다른 프로세스**. 각 프로세스에 네이티브 엔진 1개씩.
- **설정 저장소**: `SettingsRepository` 단일. `max_time_dynamic`, `chosen_move_temperature`, `wide_root_noise`, `cpuct_*` 등은 **대국 레벨과 분석용이 같은 키를 공유**.
- **설정 파일**: `AssetInstaller.ensureConfigFile(writeAnalysisConfig, writePlayConfig)` 로 **쓰기 대상을 지정**. 분석만/대국만/둘 다(디바이스 전역 변경 시) 호출부에서 명시.
- **tuned.cfg**: analysis/play **공용 1개**. `sanitizeTunedConfigForFp32()`가 **안정화·하드웨어 키만** 기록(ponderingEnabled, openclUseFP16, reportAnalysisWinratesAs). 검색·강도 키는 넣지 않아 메인 캐시그가 우선하도록 정리됨.

---

## 2. 발견된 충돌·간섭

### 2.1 없음: PLAY vs ANALYSIS 엔진 설정 혼선

- **가능성**: 대국 화면에서 HINT/DEEP(참고도) 사용 시 분석 엔진을 쓰고, 이후 일반 대국 분석(QUICK)이 잘못된 설정으로 돌 수 있음.
- **확인**: `KataGoApp` 에서 분석은 `KataGoService`, 대국은 `KataGoPlayService` 로 **프로세스가 분리**되어 있음.  
  → 분석 전용·대국 전용 네이티브 엔진이 각각 유지되므로, 참고도 사용 후에도 대국 쪽 설정이 덮어씌워지는 구조 아님.

### 2.2 없음: play.cfg 기록 순서 (이미 문서화)

- **현상**: `startGame()` 에서 `ensurePlayEngineReadyForLevel()` 후에 `ensureConfigFile(writePlayConfig=true)` 호출.
- **영향**: 레벨 변경 직후 첫 한 수는 **이전 play.cfg** 로 검색될 수 있음.
- **문서**: `PLAY_STRENGTH_DIAGNOSIS.md` §3.3.  
  필요 시 `ensureConfigFile` → `ensurePlayEngineReadyForLevel` 순서 변경 또는 재초기화 검토.

### 2.3 정리 완료: tuned.cfg 공용

- **구조**:  
  - `analysis.cfg` / `play.cfg` 둘 다 앞부분에 `@include 'tuned.cfg'` 포함(공용 1개).  
  - `EngineViewModel.sanitizeTunedConfigForFp32()` 는 **안정화·하드웨어 키만** 기록: `ponderingEnabled`, `openclUseFP16`, `reportAnalysisWinratesAs`.  
  - 검색·강도 관련 키(`chosenMoveTemperature`, `maxVisits` 등)는 tuned에 쓰지 않고, 메인 캐시그에서만 사용.
- **결과**: tuned를 통한 오버라이드 위험 제거. 메인 캐시그 값이 항상 우선.
### 2.4 정리 완료: ensureConfigFile 쓰기 대상 분리

- **API**: `ensureConfigFile(context, writeAnalysisConfig = true, writePlayConfig = true)`.
  - **분석만**: `writeAnalysisConfig=true`, `writePlayConfig=false` (예: HINT용 엔진 준비).
  - **대국만**: `writeAnalysisConfig=false`, `writePlayConfig=true` (예: startGame, 세션→PLAY 전환).
  - **둘 다**: 디바이스 전역 변경(설정 전환, 크래시 복구) 시에만 둘 다 true.
- **호출부**:
  - `ensureAnalysisEngineReadyForHint`: analysis만.
  - `setActiveSession`: 전환 대상 세션에 맞춰 한쪽만(ANALYSIS→analysis만, PLAY→play만).
  - CPU 모드 강제/폴백/재시도, EngineRepository 크래시 폴백: 둘 다(주석으로 sync both configs 명시).
### 2.5 설정 키 공유 (Repository)

- **사실**:  
  - `chosen_move_temperature`, `max_time_dynamic`, `wide_root_noise`, `cpuct_*` 등은 **하나의 SettingsRepository** 에만 있음.  
  - 대국 레벨 변경 시 이 키들이 바뀌고, analysis.cfg/play.cfg 기록 시 **같은 repo** 를 읽음.
- **현재 동작**:  
  - analysis.cfg 템플릿은 **고정값** (예: `chosenMoveTemperature = 0.0`)만 쓰고, repo의 위 키들은 **쓰지 않음**.  
  - play.cfg만 repo의 레벨 기반 값 사용.  
  → 따라서 **분석 쪽은 repo와의 직접적인 충돌 없음**.
- **참고**: tuned.cfg는 이제 안정화 키만 담으므로(§2.3), repo의 강도 키와 tuned 간 간섭 없음.

---

## 3. 경로별 요약 (충돌·간섭 관점)

| 경로 | 역할 | 충돌/간섭 |
|------|------|------------|
| PlayViewModel.startGame() | 레벨별 repo 설정 → ensurePlayEngineReadyForLevel → ensureConfigFile(writePlay만) | play.cfg만 갱신. 첫 수만 이전 캐시그일 수 있음(문서화됨). |
| EngineViewModel.setActiveSession() | 세션 전환 시 ensureConfigFile(전환 대상 한쪽만) 호출, 필요 시 엔진 재초기화 | ANALYSIS 전환→analysis만, PLAY 전환→play만 갱신. |
| EngineViewModel.triggerAnalysis() | PLAY면 getMaxTime()·playBudgetMs 사용, ANALYSIS면 고정 예산 등 | 세션·모드별로 분기되어 있음. 충돌 없음. |
| EngineViewModel.initializeEngine() | analyzingSession 에 따라 playConfigFile vs analysisConfigFile 선택 | 세션별로 올바른 캐시그 사용. |
| ensureAnalysisEngineReadyForHint() | 분석 전용 프로세스만 초기화, ensureConfigFile(analysis만) | play.cfg 미갱신으로 대국 설정과 분리. |
| AssetInstaller.ensureConfigFile() | writeAnalysisConfig/writePlayConfig로 쓰기 대상 지정, KDoc에 사용 예 정리 | 호출부에서 필요한 쪽만 갱신. |
| sanitizeTunedConfigForFp32() | tuned.cfg에 안정화·하드웨어 키만 기록 | 검색·강도 키 제외로 메인 캐시그 우선 유지. |

---

## 4. 권장 사항 (구조 정리)

1. **tuned.cfg** — 적용 완료: 안정화/하드웨어 키만 기록, 검색·강도는 메인 캐시그만 사용.
2. **ensureConfigFile** — 적용 완료: `writeAnalysisConfig`/`writePlayConfig` 로 세분화, 호출부에서 분석만/대국만/둘 다 명시.
3. **startGame() 순서** (선택): 레벨 변경 직후 첫 수를 맞추려면 `ensureConfigFile(writePlayConfig=true)` 를 `ensurePlayEngineReadyForLevel()` 앞으로 옮기거나, 기록 후 대국 엔진만 재초기화 검토.

---

## 5. 결론

- **로직 간 직접적인 충돌**은 **없음**.
- **tuned.cfg**: 검색·강도 키를 tuned에 쓰지 않도록 수정해, 분석/대국 메인 캐시그가 덮어씌워지지 않음.
- **ensureConfigFile**: `writeAnalysisConfig` / `writePlayConfig` 로 쓰기 대상을 나누고, 세션 전환·대국 시작 시 필요한 쪽만 갱신하도록 호출부 정리함.
