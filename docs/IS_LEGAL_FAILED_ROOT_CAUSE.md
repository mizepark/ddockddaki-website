# isLegal_failed 원인

로그에 나오는 예외:

```
EngineViewModel: isLegal_failed
java.lang.IllegalArgumentException: NativeSafetyGate: size must be positive, got 0
    at NativeSafetyGate.withIntArray(NativeSafetyGate.kt:17)
    at EngineViewModel.ensureMoveIsLegal → withMoveInts(snapshot.game.moves)
    at EngineViewModel.playMove(EngineViewModel.kt:1114)
```

---

## 직접 원인 (왜 예외가 나는가)

1. 사용자가 보드에서 한 점을 탭하면 `playMove(coord)` 가 호출된다.
2. `snapshot = _uiState.value` 로 현재 UI 상태를 읽고, 그 안의 `snapshot.game.moves` 로 “지금까지 둔 수 목록”을 넘긴다.
3. `ensureMoveIsLegal(snapshot, coord, ...)` → `withMoveInts(snapshot.game.moves)` → **`NativeSafetyGate.withIntArray(moves.size * 2)`** 를 호출한다.
4. **`snapshot.game.moves` 가 비어 있으면** `moves.size == 0` 이라 **`withIntArray(0)`** 이 호출된다.
5. `NativeSafetyGate.withIntArray(size)` 는 **`require(size > 0)`** 이라서 **size가 0이면 위 IllegalArgumentException 을 던진다.**
6. 그 결과 legal 체크가 실패하고, 착수도 되지 않는다.

즉, **“지금까지 둔 수 목록”이 비어 있을 때** legal 검사 경로에서 **길이 0 배열을 허용하지 않아서** `isLegal_failed` 가 발생한다.

---

## 왜 moves 가 비어 있나 (가능한 경우)

- **빈 보드 첫 수**  
  새 게임/첫 수를 두는 경우가 정상적으로 존재한다. 이때 `game.moves` 는 빈 리스트가 맞다.
- **세션 복원이 없을 때**  
  `EngineSessionOrchestrator.switchSession()` 에서 대상 세션에 대한 복원 데이터가 없으면 `buildEmptySessionState()` 로 **moves = emptyList()** 인 상태가 만들어진다.  
  이 빈 게임 상태가 `_uiState` 에 적용된 채로 있고, 사용자가 그 화면에서 탭하면 `snapshot.game.moves` 가 계속 비어 있다.
- **분석/대국 진입 직후**  
  앱을 켠 뒤 분석/대국 화면으로 들어올 때, 위와 같이 “빈 세션” 상태가 적용되면 처음부터 `game.moves` 가 비어 있을 수 있다.

사용자가 “한 번도 착수가 된 적이 없다”면, **항상 이 빈 상태로 playMove 가 호출되고 있다**고 보는 것이 맞다.

---

## 네이티브 쪽은 빈 배열을 받을 수 있는가

`katago_jni.cpp` 의 `nativeIsLegal` 를 보면:

- `len == 0` 이어도 `len < 0 || len > 1000` 에 걸리지 않는다.
- for 루프는 `i+1 < len` 이라 len=0 이면 한 번도 돌지 않고, 빈 보드·흑 선으로 `hist.isLegal(board, q, next)` 만 호출한다.

즉 **네이티브는 “수 목록 길이 0 = 빈 보드 첫 수”를 정상 처리할 수 있다.**  
막는 것은 **Kotlin 쪽 `NativeSafetyGate.withIntArray(0)` 의 `require(size > 0)`** 뿐이다.

---

## 한 줄 요약

- **원인**: 착수 시 `snapshot.game.moves` 가 비어 있는데, legal 검사에서 이걸 `withMoveInts` → `NativeSafetyGate.withIntArray(moves.size * 2)` 로 넘기면서 **size == 0** 이 되고, **`withIntArray` 가 size > 0 만 허용해서** 예외가 나며 legal 실패 → 착수 불가.
- **추가**: 빈 보드 첫 수는 정상 케이스이고, 네이티브는 빈 배열을 받아도 동작하도록 되어 있음. 따라서 **원인은 “빈 moves 를 허용하지 않는 Kotlin 쪽 조건”이다.**
