# 시간제한(Time Limit) 적용 방식

코드 기준으로 분석·대국 시 **시간제한이 어떻게 적용되는지** 정리한 문서입니다.

---

## 1. 전체 흐름 요약

```
[설정/모드] → targetMs(budgetMs) → AnalysisUseCase: maxTimeSeconds = budgetMs/1000
    → EngineRepository → KataGoService → NativeBridge.analyzeTop4(maxTimeSeconds, maxVisits)
    → katago_jni: localParams.maxTime = reqMaxTime
    → Search::runWholeSearch() → timeUsed >= maxTime 시 검색 중단
```

- **Kotlin**: `budgetMs`(밀리초) → `maxTimeSeconds`(초)로 변환해 엔진에 전달.
- **Native**: `SearchParams.maxTime`(초)를 사용해 `runWholeSearch()` 내부에서 `timeUsed >= maxTime`일 때 검색을 멈춤.

---

## 2. 분석 모드별 예산(시간제한) — EngineViewModel.kt

| 모드 | targetMs (budgetMs) | 비고 |
|------|---------------------|------|
| **QUICK** | 대국(PLAY)이면 `playBudgetMs`, 아니면 **3000ms** | playBudgetMs = 설정 "최대 시간" |
| **NORMAL** | 대국이면 `playBudgetMs`, 아니면 `targetLatencyMs` 또는 **30000ms** | targetLatencyMs 기본 3000 |
| **HINT** | **20000ms** (20초) | 고정 |
| **DEEP** | **30000ms** (30초) | 고정 |

### playBudgetMs (대국 시 한 수당 예산)

- **출처**: `settingsRepository.getMaxTime().coerceIn(1f, 6.0f) * 1000` (초 → ms)
- **설정 키**: `max_time_dynamic` (기본 **3.0초**, 범위 1~6초)
- 즉, **대국 모드 QUICK/NORMAL**에서는 설정의 "최대 시간(초)"이 그대로 **한 수당 시간제한(ms)**으로 쓰입니다.

---

## 3. targetLatencyMs / maxTimeSeconds (일반 분석·UI 연동)

- **targetLatencyMs**: 사용자 설정 또는 `computeDynamicLatencyMs()`.
- 현재 `computeDynamicLatencyMs(moveCount)`는 **고정 3000ms** 반환 (락 해제 시 `prefTargetLatencyMs`).
- **maxTimeSeconds** = `latencySeconds(latencyMs)` = **targetLatencyMs / 1000.0** (초).
- 분석 요청 시 `budgetMs`가 위 `targetMs`로 정해지고, 그대로 `maxTimeSeconds`로 전달됩니다.

---

## 4. AnalysisUseCase — 엔진에 넘기는 값

```kotlin
// budgetMs: EngineViewModel에서 정한 예산(ms)
val maxTimeSeconds = (budgetMs.toDouble() / 1000.0)

engineController.analyzeTop4(
    ints = ints,
    maxTimeSeconds = maxTimeSeconds,  // 시간제한(초)
    maxVisits = maxVisits,
    ...
)
```

- **시간제한만 보면**: `budgetMs`(ms) → `maxTimeSeconds`(초)로 변환해 엔진에 넘기는 구조입니다.

---

## 5. Native (katago_jni.cpp) — 실제 검색 제한

```cpp
double reqMaxTime = (jMaxTimeSeconds > 0.0) ? jMaxTimeSeconds : g_globalParams.maxTime;
localParams.maxTime = reqMaxTime;
// ...
searchToUse->setParams(localParams);
searchToUse->runWholeSearch(g_stopFlag);
```

- **jMaxTimeSeconds**: Java에서 넘긴 `maxTimeSeconds`.
- **Search::runWholeSearch()** (search.cpp):  
  `hasMaxTime && timeUsed >= maxTime` 일 때 검색 중단.  
  즉, **실제 시간제한은 Native의 `maxTime`(초)으로 적용**됩니다.

### 안전장치

- `kMaxAnalysisSeconds = 5.0`: 절대 상한(일반 maxTime보다 큰 값으로 사용).

---

## 6. 설정에서 시간 제어 (현재 기기 적용값 확인 방법)

- **한 수당 최대 시간(초)**  
  - `SettingsRepository.getMaxTime()`  
  - SharedPreferences `max_time_dynamic` (기본 **3.0**, 1~6초 범위).
- **targetLatencyMs**  
  - 사용자가 "대기 시간" 등을 바꾸면 `prefTargetLatencyMs` / `targetLatencyMs`가 바뀌고,  
  - NORMAL 등에서 `snapshot.targetLatencyMs`로 `targetMs`가 정해짐.

---

## 7. 로그로 확인하는 방법 (기기 연결 후)

앱을 켠 뒤 분석이 돌아가는 동안:

```bash
adb logcat | findstr /i "triggerAnalysis budget KataGoJNI Time Visits"
```

- **triggerAnalysis start mode=... budget=... maxVisits=...**  
  → 이번 분석에 쓰인 **budget(ms)** = 시간제한에 대응.
- **KataGoJNI: analyzeTop4: Threads=... Time=... Visits=...**  
  → 엔진에 넘긴 **Time(초)** = 실제 적용된 시간제한.
- **analyzeTop4: Search finished visits=... elapsed=...**  
  → 실제 검색에 걸린 시간(초).

정리하면, **시간제한은 설정/모드에서 정해진 예산(budgetMs) → maxTimeSeconds → Native maxTime** 순으로 적용되고, Native 검색 루프에서 `timeUsed >= maxTime`일 때 종료됩니다.
