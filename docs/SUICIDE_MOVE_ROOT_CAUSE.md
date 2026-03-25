# 봇 자살수 원인 분석 및 수정

## 1. 왜 "엔진 미준비"는 원인이 아닌가

- 로그에 **방문수 2~4짜리 착수**가 찍힌다 → 그 턴에는 **분석이 실행됐고**, 엔진이 그 수를 제언한 상태다.
- 분석은 `triggerAnalysis`에서 `!snapshot.isModelReady`이면 아예 시작하지 않고, 봇 착수는 `isBotMoveReady(engine)`이 true일 때만 시도하는데, 이때 `state.isModelReady`가 false면 false를 반환한다.
- 따라서 **정상 경로에서는** “엔진이 준비 안 됐는데 분석하고 봇이 둔다”는 시나리오가 성립하지 않는다.  
  → “엔진 미준비일 때 ensureMoveIsLegal이 true를 반환해서 자살수가 나온다”는 설명은 **방문수 로그와 맞지 않아** 원인으로 보기 어렵다.

## 2. 자살수가 통과할 수 있는 경로

착수 허용은 `EngineViewModel.playMove` → `ensureMoveIsLegal(snapshot, coord)`가 true를 반환할 때만 일어난다.  
자살인데 true가 나오려면, **합법성 검사에 쓰는 “국면”이 잘못되었거나**, **검사 로직이 자살을 합법으로 판단**해야 한다.

가능한 원인 후보:

| 후보 | 설명 |
|------|------|
| **A. 스냅샷 불일치** | `ensureMoveIsLegal`에 넘기는 `snapshot`이 `playMove` 진입 시점의 `_uiState.value`인데, 그 시점의 `snapshot.game.moves`가 실제 현재 수순과 다르면(예: 한 수 밀림/어긋남), “다른 국면” 기준으로 합법이라 나와서 자살이 통과할 수 있음. |
| **B. 네이티브 isLegal** | KataGo JNI `nativeIsLegal`은 `movesXY`로 국면을 만들고 `hist.isLegal(board, q, next)`로 판단. 구현상 자살은 false여야 하나, 특정 국면/예외 경로에서 true가 나오는 버그 가능성(낮지만 제로는 아님). |
| **C. Kotlin 검사만 통과한 경우** | PlayViewModel에서 `isLegalMove(state, coord)`로 걸러지지만, 그때 쓰는 `state`와 `playMove` 시점의 `snapshot`이 다르면(플로우 타이밍), Play에서는 합법으로 골랐는데 실제 커밋 시점 국면에서는 자살일 수 있음. 그때 `ensureMoveIsLegal`이 같은 `snapshot`으로 한 번 더 검사하므로, **검사에 쓰는 수순이 실제 국면과 같다면** 여기서 걸러져야 함. |

즉, **“검사 시 사용하는 수순(snapshot.game.moves)이 실제 국면과 일치하는지”**가 중요하고, 일치하는데도 자살이 나온다면 네이티브/타이밍 쪽을 의심해야 한다.

## 3. 수정: Kotlin 합법성 선검사

원인을 로그만으로 특정하기 어렵기 때문에, **방어적으로** 다음을 적용했다.

- **`ensureMoveIsLegal` 진입 시, 네이티브 호출 전에 항상 Kotlin 규칙으로 한 번 검사한다.**
  - `isLegalMoveKotlin(snapshot, coord)` = `snapshot.game.moves`와 `snapshot.game.boardSize`만으로 `SimpleGoBoard.getLegalNextMoves`를 호출해, `coord`가 합법 목록에 있을 때만 true.
  - 여기서 **false면 즉시 return false** → 자살/불법/이미 둔 점은 무조건 거절.
- 그다음:
  - 엔진 미준비면 네이티브 호출 없이 **Kotlin 통과만으로 true** (이미 위에서 불법/자살은 걸러짐).
  - 엔진 준비됐으면 기존처럼 네이티브 `isLegal` 호출하고, 실패 시 Kotlin 폴백 유지.

효과:

- **수순이 맞는 한** (즉, `snapshot.game.moves`가 현재 국면을 제대로 반영하는 한) 자살·불법·중복 착수는 Kotlin 단에서 한 번 더 걸러져서 커밋되지 않는다.
- 네이티브가 잘못 true를 주거나, 타이밍 때문에 스냅샷이 약간 어긋나더라도, **같은 스냅샷으로** Kotlin이 “이 수순에서 이 좌표는 합법이 아니다”라고 하면 착수는 막힌다.
- 단, **스냅샷 자체가 잘못된 수순**(예: 인간의 마지막 수가 빠진 상태)이면, 그 잘못된 수순 기준으로는 “합법”이라 나와 통과할 수 있다. 그 경우 원인은 “엔진 미준비”가 아니라 **봇 턴/playMove로 넘어가는 시점의 state와 snapshot 동기화**이므로, 그 부분은 별도로 짚어야 한다.

## 4. 요약

- **원인**: “엔진 미준비일 때 true 반환”은 로그(방문수 찍힌 착수)와 맞지 않아 **자살수의 직접 원인으로 보기 어렵다.**  
  실제 원인은 (스냅샷 불일치 / 네이티브 예외 케이스 / Kotlin–스냅샷 타이밍 등) 중 하나일 수 있으나, 로그만으로는 특정이 어렵다.
- **수정**: `ensureMoveIsLegal`에서 **항상 `isLegalMoveKotlin(snapshot, coord)` 선검사**를 넣어, 여기서 한 번이라도 걸리면 착수를 허용하지 않도록 했다.  
  같은 스냅샷을 쓰는 한, 자살/불법/중복은 Kotlin 규칙으로 한 번 더 막히므로, “엔진이 준비된 상태에서 분석하고 방문 2~4짜리를 둔 뒤 자살이 나오는” 경우에도 방어적으로 차단된다.
