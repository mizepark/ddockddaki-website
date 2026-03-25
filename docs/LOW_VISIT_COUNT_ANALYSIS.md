# 방문수 1(또는 낮은 방문수)이 많이 나오는 이유 — 로그·코드 분석

수정 없이 원인만 정밀 분석한 문서입니다.

---

## 1. 로그에서 보이는 현상

캡처된 `selectBotMove` 로그에서 **실제 착수한 수**의 visits 값은 대부분 **2**, 일부 **4, 7, 12, 15, 133** 등입니다. “방문수 1”은 **코드 경로상 가능**하고, 실제로는 **2가 압도적으로 많음** (1은 특수 조건에서만 노출).

아래는 “낮은 방문수(1 또는 2)”가 자주 나오는 **전체 경로**를 코드·로그 기준으로 정리한 것입니다.

---

## 2. 원인 요약 (체인)

1. **봇이 “착수 가능”으로 판단하는 시점이 매우 이르다**  
   → `lastTotalVisits >= 1` 이면 곧바로 착수 시도.
2. **대국모드(QUICK)에서는 중간 결과도 곧바로 topSuggestions로 쓴다**  
   → 총 방문수가 적은 초기/중간 결과가 그대로 반영됨.
3. **분석 시간 예산이 짧다**  
   → 레벨 30은 착수당 약 1초라, 수많은 수에 방문이 고르게 쌓이지 않음.
4. **후보 필터가 “방문수 ≥ 1”만 요구한다**  
   → `BOT_MIN_SUGGESTION_VISITS = 1`이라 1~2 방문 수도 후보로 허용.
5. **parseRawResult에서 “전원 1방문”이면 minVisits 필터를 우회한다**  
   → 그 결과 **방문수 1**인 후보가 topSuggestions에 들어갈 수 있음.
6. **minVisits는 최소 2**  
   → 정상 필터 경로에서는 1방문은 걸러지고, **2방문**이 최소로 많이 남음.

그래서 **로그에는 주로 visits=2**가 많이 찍히고, **방문수 1**은 “전체 후보가 다 1방문일 때”만 선택될 수 있는 구조입니다.

---

## 3. 코드 경로별 상세

### 3.1 봇이 “착수 가능”하다고 보는 조건 (너무 이른 시점)

**위치**: `PlayViewModel.kt` — `isBotMoveReady()`

```text
return state.analysis.isInitialAnalysisReady || state.analysis.lastTotalVisits >= 1
```

- **lastTotalVisits >= 1** 이면, 즉 **한 번이라도 분석 결과가 오면** 착수 가능으로 본다.
- DEEP/HINT처럼 “1000방문 도달” 같은 문턱이 없음.
- 따라서 **첫 번째 중간 결과**(총 방문수 수십~백 단위)가 오는 시점에 이미 `isBotMoveReady == true`가 되어, 그 시점의 `topSuggestions`로 `selectBotMove`가 호출될 수 있다.

→ **초기/중간 결과에 포함된 “낮은 방문수” 후보가 그대로 착수 선택에 사용되는 첫 번째 이유.**

---

### 3.2 QUICK 모드에서 중간 결과를 곧바로 추천으로 사용

**위치**: `EngineViewModel.kt` — `handleAnalysisResult()` (중간 결과 분기)

- DEEP/HINT: `isGoalReached = (result.totalVisits >= 1000)` 등으로 “목표 도달” 후에만 추천 표시.
- **QUICK(대국모드)**  
  - `isGoalReached = true` (고정)  
  - `shouldShowSuggestions = isGoalReached && result.suggestions.isNotEmpty()`  
  → **추천이 1개라도 있으면** 곧바로 `topSuggestions`를 그 결과로 갱신하고 `isInitialAnalysisReady = true`로 만든다.

즉, **총 방문수가 아직 적은 중간 결과**가 와도, 그 시점의 `result.suggestions`(각 수의 visits 포함)가 그대로 UI/봇에 전달된다.  
→ **낮은 방문수 후보가 topSuggestions에 들어가는 두 번째 이유.**

---

### 3.3 분석 시간 예산이 짧음 (레벨 30)

**위치**: `EngineViewModel.kt` — `playBudgetMs`, `getPlayTimeRangeForLevel` 등.

- 레벨 30: `timeRange=1.0s~1.0s` → 착수당 약 **1초**만 분석에 씀.
- 엔진은 이 1초 안에 MCTS를 돌리며, **많은 수에 방문을 고르게 쌓기 전에** 시간이 끝난다.
- 결과적으로 “한두 수에만 방문이 몰리고, 나머지는 1~2 방문”인 분포가 자주 나옴.

→ **낮은 방문수(1~2) 후보가 통째로 많이 만들어지는 구조적 이유.**

---

### 3.4 봇 후보 필터: “방문수 ≥ 1”만 요구

**위치**: `PlayViewModel.kt` — `selectBotMove()`, 상수

```text
private const val BOT_MIN_SUGGESTION_VISITS = 1
...
val suggestions = state.analysis.topSuggestions
    .filter { it.visits >= BOT_MIN_SUGGESTION_VISITS }
```

- **1 이상**이면 모두 후보로 쓴다.  
- 따라서 **topSuggestions에 들어온 1방문/2방문 수**는 그대로 최저기력 풀(pick=lowest)에 포함되고, 그중 하나가 선택될 수 있다.

→ **방문수 1·2가 “선택 가능한 후보”로 남는 직접적인 코드 이유.**

---

### 3.5 parseRawResult: minVisits 필터와 “전원 1방문”일 때의 예외

**위치**: `AnalysisUseCase.kt` — `parseRawResult()`

- `minVisits` 계산:
  - `minShare` 등으로 `minVisits = (maxVisitsValue * minShare).toInt().coerceAtLeast(2)`  
  → **maxVisitsValue > 0 이면 최소 2**.
- 필터:
  - `filteredSuggestions = sortedSuggestions.filter { s -> s.visits >= minVisits }.take(MAX_SUGGESTIONS)`  
  → 기본적으로 **visits ≥ 2**인 수만 통과.
- **예외(폴백)**:
  - `finalSuggestions = if (filteredSuggestions.isNotEmpty() || sortedSuggestions.isEmpty()) filteredSuggestions else sortedSuggestions.take(MAX_SUGGESTIONS)`  
  → **filteredSuggestions가 비어 있으면** (즉, **모든 후보가 minVisits 미만**이면) `sortedSuggestions`를 그대로 쓴다.
  - 이때 **모든 수가 1방문**(maxVisitsValue=1 → minVisits=2)이면, 1방문 수들이 그대로 `finalSuggestions` → `topSuggestions`로 전달된다.

정리하면:

- **일반적인 경우**: minVisits=2 때문에 **1방문은 걸러지고, 2방문이 최소**로 많이 남음 → 로그에 **visits=2**가 많음.
- **전체가 1방문인 결과가 오는 경우**: 폴백 때문에 **방문수 1**인 후보가 topSuggestions에 들어가고, `BOT_MIN_SUGGESTION_VISITS=1` 때문에 봇이 그대로 선택 가능 → **방문수 1**이 가끔 나올 수 있음.

---

### 3.6 레벨 30일 때 minShare

**위치**: `AnalysisUseCase.kt` — `minShareForPlayLevel(30)`

- `if (idx >= 30) return 0f` → **minShare = 0**.
- 그래도 `minVisits = (maxVisitsValue * minShare).toInt().coerceAtLeast(2)` 에서 **coerceAtLeast(2)** 때문에 minVisits는 최소 2.
- 즉, 레벨 30에서도 “2 미만 방문은 정상 경로에서는 제거”이고, **1방문은 오직 “전원 1방문 → 폴백”일 때만** 노출된다.

---

## 4. 흐름 요약 (왜 “방문수 1” 또는 “낮은 방문수”가 많이 나오는가)

1. **대국 시작** → QUICK 모드, 레벨 30이면 착수당 분석 예산 약 **1초**.
2. 엔진이 **중간 결과**를 보냄 (총 방문수 수십~백, 많은 수가 1~2 방문).
3. QUICK은 **목표 방문수 문턱 없이** “추천 있으면 표시” → 이 중간 결과가 곧바로 **topSuggestions**로 반영되고 **isInitialAnalysisReady = true**.
4. **lastTotalVisits >= 1** 이므로 **isBotMoveReady = true** → 봇이 **이 시점의 topSuggestions**로 `selectBotMove` 호출.
5. **parseRawResult**  
   - 보통은 minVisits=2라 **1방문은 제거** → **2방문이 최소**로 많이 남음.  
   - “전원 1방문”인 결과만 오면 폴백으로 **1방문 후보**가 topSuggestions에 들어감.
6. **BOT_MIN_SUGGESTION_VISITS = 1** 이라 **1·2 방문 후보 모두** 봇 후보 풀에 포함.
7. 레벨 30은 **최저 승률** 선택 → 그 풀 안에서 **가장 낮은 승률**인 수가 고르게 선택되고, 그 수의 visits 값이 로그에 **visits=1** 또는 **visits=2**로 많이 찍힘.

---

## 5. 정리 (설명만, 수정 없음)

- **방문수 1**이 나오는 경우  
  - 엔진이 **모든 후보가 1방문**인 결과를 준 때만, parseRawResult의 **폴백**으로 topSuggestions에 1방문 수가 들어가고, `BOT_MIN_SUGGESTION_VISITS=1` 때문에 봇이 그 수를 선택할 수 있음.
- **방문수 2**가 많이 나오는 경우  
  - QUICK 모드에서 **초기/중간 결과**를 곧바로 쓰고,  
  - 분석 시간이 짧아서 많은 수가 1~2 방문만 갖고,  
  - minVisits=2 필터를 통과하는 최소 값이 2이기 때문에,  
  - 봇이 “낮은 방문수” 후보를 고를 때 **2가 최소로 자주 선택**됨.

즉, “방문수 1이 많이 나온다”는 현상은 **코드상으로는 “1은 특수 조건(전원 1방문)에서만 노출, 2는 구조적으로 자주 노출”**이라서, 로그에서는 **2가 압도적**이고, **1은 가끔** 나오는 구조로 이해하면 됩니다.
