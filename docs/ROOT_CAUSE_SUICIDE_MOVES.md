# 봇이 자살수만 두는 현상 — 근본 원인 분석

---

## (추가) 첫 수부터 가장자리만 도는 현상 — 보드 사이즈 불일치

### 현상
- **첫 수부터** 자살에 가깝거나 말이 안 되는 수.
- 봇이 **바둑판 맨 가장자리만** 돌아가며 둠 (테두리 한 바퀴).

### 원인: 정책 배열의 보드 사이즈와 앱이 쓰는 보드 사이즈가 다름

1. **네이티브 정책 배열**
   - `nativeGetPolicyHeatmap`는 **`g_boardSize`**만 사용한다.
   - `g_boardSize`는 **`nativeInitEngine(modelPath, ..., boardSize)`** 호출 시점에 한 번 설정된다.
   - 반환 배열 크기 = `g_boardSize * g_boardSize` (예: 17×17 → 289, 19×19 → 361).
   - 배열은 `out[y * g_boardSize + x]` 로 채워진다 (행 우선, `x,y ∈ [0, g_boardSize)`).

2. **앱에서의 사용**
   - `getPolicyHeatmapForCurrentGame()`은 **`snapshot.game.boardSize`**로 `expected = boardSize * boardSize`를 계산한다.
   - 반환된 배열이 `expected`보다 작으면 **길이 그대로** 쓴다 (`else arr`).
   - `loadPolicyMoves()`에서는 **`state.game.boardSize`**로 `idx = y * boardSize + x`를 써서 `policy[idx]`를 읽는다.

3. **불일치 시**
   - 예: 앱 게임은 **19×19** (`state.game.boardSize == 19`), 네이티브는 **17로 초기화**되어 있음 (`g_boardSize == 17`).
   - 네이티브는 289개 float 반환 (17×17).
   - 앱은 19×19라고 가정하고 `idx = y*19 + x`로 0~360 인덱스를 사용.
   - 17×17 배열을 19×19 **stride**로 읽으면:
     - `idx=17` → 앱 (17,0)인데, 네이티브에서는 (0,1)에 해당.
     - `idx=18` → 앱 (18,0)은 네이티브 (1,1)에 해당.
   - 즉 **앱 좌표의 “가장자리” (x=17,18 또는 y=17,18 부근)**에, 네이티브 쪽 **중앙/내부** 셀의 높은 정책값이 매핑됨.
   - 그 결과, 봇이 “가장자리”를 높은 확률로 골라서 **가장자리만 도는 것처럼** 보인다.

4. **왜 첫 수부터인가**
   - 분석 추천(`topSuggestions`)이 비어 있거나 모두 걸러지면, **항상 정책 폴백**만 사용한다.
   - 그때 쓰는 정책 배열이 위와 같이 **보드 사이즈가 어긋난 상태**면, 첫 수부터 잘못된(가장자리) 수가 선택된다.

### 요약
- **원인:** 네이티브 `g_boardSize`(init 시점 값)와 앱 `state.game.boardSize`(현재 게임)가 다를 때, 정책 배열을 **앱 보드 크기 기준 인덱스**(`y*boardSize+x`)로 읽어서 **stride가 틀어짐**.
- 그로 인해 **앱 기준 가장자리**에 **네이티브 기준 내부**의 높은 확률이 붙고, 봇이 가장자리만 두는 것처럼 보인다.
- 확인할 것: Play(또는 대국) 시작/진입 시 **initEngine에 넘기는 boardSize**와 **현재 게임의 `game.boardSize`**가 항상 같은지, 그리고 **정책 요청 시점의 `snapshot.game.boardSize`**와 **네이티브가 쓰는 보드 크기**가 일치하는지.

---

## (기존) 자살수만 두는 현상 — 스테일 추천

### 요약

**근본 원인:** 분석 결과가 **현재 수순과 맞지 않아 무시될 때** `topSuggestions`를 비우지 않아, **예전 국면용 추천이 그대로 남고**, 그걸 **지금 국면**으로 합법성 검사하면 대부분 불합법/자살로 걸러진다. 그 결과 **항상 정책망 폴백**으로 가고, 정책망이 자살에 높은 확률을 주는 경우 봇이 자살수만 두게 된다.

---

## 1. 데이터 흐름

- **분석 요청:** `triggerAnalysis()`에서 `snapshot.game.moves`로 엔진에 수순 전달 → 분석 실행.
- **결과 적용:** `handleAnalysisResult()`에서 `result.analyzedMovesHash == computeMovesHash(current)` 및 `analyzed.size == current.size`일 때만 `topSuggestions`와 `game.stones` 등을 갱신.
- **봇 착수:** `selectBotMove(attemptState, ...)`에서 `attemptState.analysis.topSuggestions`와 `attemptState.game.moves` 사용.

즉, **추천은 “분석에 쓰인 수순”에만 유효**하고, **봇은 “현재 수순”으로 합법성만 검사**한다.

---

## 2. 근본 원인: 무시된 결과일 때 추천을 안 지움

**위치:** `EngineViewModel.kt` — `handleAnalysisResult()`

- **수순 불일치 시 (hash 불일치):**
  - `result.analyzedMovesHash != SnapshotValidator.computeMovesHash(current)` 이면
  - `return@withContext`만 하고 **`topSuggestions` / `isInitialAnalysisReady`를 갱신하지 않음.**
  - 따라서 **이전에 적용된 `topSuggestions`가 그대로 유지됨.**

- **길이 불일치 시 (size 불일치):**
  - `analyzed.size != current.size` 이면
  - `statusMessage`, `overlay`만 바꾸고 **`topSuggestions`를 비우지 않음.**

결과적으로:

- 분석이 **예전 수순(예: 8수)** 기준으로 끝나고,
- 그 사이에 **사람이 수를 두어 현재 수순(예: 11수)**이 되었으면,
- 위 검사 때문에 이번 결과는 **적용되지 않지만**,
- **예전 8수용 `topSuggestions`가 그대로 남아 있음.**

봇 턴에서:

- `selectBotMove(attemptState, ...)`는 **현재 수순(11수)**의 `attemptState.game.moves`로 합법성 검사.
- 8수용 추천을 11수 국면에 대입하면 **대부분 불합법/자살**이라
- `suggestions.filter { isLegalMove(state, it.coord) }` 결과가 **거의 항상 비어 있음.**
- 그러면 `suggestions.isNotEmpty()` 분기를 타지 못하고
- **항상 `loadPolicyMoves()` → 정책망 폴백**으로만 착수하게 됨.

그래서 “최소 방문수 조건을 아무리 올려도” 의미가 없음.  
엔진이 준 고방문수 추천은 **다른 국면용**이라 걸러지고, 실제로 두는 수는 **전부 정책망 폴백**이기 때문이다.

---

## 3. 왜 “자살수만” 두는가

- **실제 선택 경로:** 위와 같이 대부분의 턴에서 `suggestions`가 비어 → **정책망만 사용.**
- 정책망은 **합법성 검사 없이** 확률만 낼 수 있어, 특정 국면에서 **자살점에 높은 확률**을 줄 수 있음.
- `loadPolicyMoves()`에서 `fromCurrentStateAndVariation`으로 합법만 넣었더라도,
  - `state`와 **정책을 요청할 때의 `_uiState.value`**가 다를 수 있고 (레이스),
  - 또는 **정책 자체가 자살을 합법으로 오인**하는 경우가 있으면,
  자살이 후보에 섞일 수 있음.
- 더 큰 문제는, **봇이 거의 항상 정책망 경로만 타기 때문에** “엔진이 검토한 수”가 아니라 “정책만 본 수”만 두게 되고, 그 결과 자살수가 자주 나오는 것이다.

즉, **원인은 “방문수 조건”이 아니라, “다른 국면용 추천이 남아 있어서 추천 경로는 거의 쓰이지 않고, 정책망 폴백만 쓰이게 되는 구조”**이다.

---

## 4. 검증 포인트 (수정 시 확인할 것)

1. **`handleAnalysisResult()`**
   - `result.analyzedMovesHash != computeMovesHash(current)` 일 때:
     - **`topSuggestions`를 `emptyList()`로 갱신**
     - 필요하면 `isInitialAnalysisReady = false` 등으로 “아직 이 국면용 분석 없음”을 표시.
   - `analyzed.size != current.size` 일 때:
     - 위와 동일하게 **`topSuggestions` 비우기** (그리고 필요 시 ready 플래그 정리).

2. **추천 사용 측**
   - `selectBotMove()`(또는 그 호출부)에서,
     - “이 추천이 현재 수순에 대응하는지”를 알 수 있다면 (예: 분석 수순 해시를 상태에 넣어 두었다가 비교),
     - **수순이 다르면 추천을 아예 쓰지 않고**, 정책/폴백만 사용하도록 하는 것도 가능.

3. **정책 폴백**
   - `loadPolicyMoves` / `getPolicyHeatmapForCurrentGame()` 이 **항상 `selectBotMove`에 넘어온 `state.game.moves`와 동일한 수순**을 쓰는지 확인 (현재 `getPolicyHeatmapForCurrentGame()`는 `_uiState.value` 기준이라, `state`와 다른 수순일 수 있음).

---

## 5. 결론

- **직접 원인:**  
  분석 결과가 “현재 수순과 달라” 적용되지 않을 때 **`topSuggestions`를 비우지 않음** → 예전 국면용 추천이 남음 → 현재 국면으로 검사하면 대부분 불합법 → **항상 정책망 폴백**으로만 수를 둠.
- **그 결과:**  
  최소 방문수 조건과 관계없이, 실제로는 “방문수 1짜리처럼” 정책만 보는 수만 두게 되고, 정책이 자살을 높게 주면 **자살수만 두는 것처럼** 보인다.
- **수정 방향:**  
  결과가 무시될 때마다 **해당 국면용 추천을 버리도록** `topSuggestions`(및 필요 시 관련 플래그)를 명시적으로 비우는 것이 근본 대응이다.
