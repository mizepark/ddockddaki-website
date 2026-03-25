# 로그 기반 현재 상태 분석 (설명만)

수집 로그: `device_log_collected.txt` (버퍼 비운 뒤 ~6,947줄, 02-06 20:46:31 ~ 20:47:16)

---

## 1. 앱 생명주기 요약

| 시각 | 내용 |
|------|------|
| 20:46:36 | `com.katago.android`(PID 3424) 프로세스에서 I/O (Read_top) 기록 — 앱 이미 실행 중 |
| 20:47:09 | MainActivity 포커스, SurfaceFlinger에 HighHint 등 (앱 전면 사용) |
| 20:47:11 | **앱이 홈으로 나감**: MainActivity onPause → 백그라운드, launcher로 포커스 이동 |
| 20:47:11 | **Memory trim critical (level=20)**: `clearCache+trimMemory only, no stopAnalysis` — 분석 중단은 하지 않고 캐시/메모리만 정리 |
| 20:47:11 | WindowStopped, `com.katago.android is not in foreground` |
| 20:47:12 | 사용자가 다시 KataGo 아이콘으로 **앱 재실행** (같은 프로세스 3424, 기존 태스크 복원) |
| 20:47:15 ~ 20:47:16 | **착수 시도 시 에러 반복** (아래 2번) |

즉, **앱 켜기 → 잠깐 홈으로 나감 → 다시 앱 들어옴 → 보드 탭해서 착수 시도** 순서로 사용한 구간이 로그에 남아 있음.

---

## 2. 착수가 안 되는 직접 원인 (로그에 찍힌 예외)

동일 예외가 여러 번 반복됨:

```
W/Katago: EngineViewModel: isLegal_failed
java.lang.IllegalArgumentException: NativeSafetyGate: size must be positive, got 0
    at NativeSafetyGate.withIntArray(NativeSafetyGate.kt:17)
    at EngineViewModel.ensureMoveIsLegal(EngineViewModel.kt:2869)
    at EngineViewModel.playMove(EngineViewModel.kt:1114)
```

- **직접 원인**: `ensureMoveIsLegal()` 안에서 `snapshot.game.moves`로 `withMoveInts(moves)`를 호출하는데, 이때 **`moves.size == 0`** 이라 `NativeSafetyGate.withIntArray(moves.size * 2)` = `withIntArray(0)` 이 호출됨.
- `NativeSafetyGate`는 `size > 0`만 허용하므로 `"size must be positive, got 0"` 로 예외 발생 → legal 체크 실패 → 착수 반영 안 됨.

즉, **“착수 직전에 쓰인 UI 스냅샷의 `game.moves`가 비어 있는 상태”**에서 legal 검사가 호출되고 있음.

---

## 3. 현재 상태 해석 (왜 moves가 비어 있었나)

- **가능한 원인 1 — 백그라운드/복귀 후 상태 불일치**  
  앱이 홈으로 나갔다가 같은 프로세스로 다시 포그라운드로 돌아왔을 때, 화면에 보이는 게임 상태와 `playMove`에 넘기는 `snapshot`이 일치하지 않았을 수 있음.  
  예: 복원된 화면은 “몇 수 둔 국면”인데, 탭 처리 시점에 참조된 스냅샷만 “빈 국면(moves 빈 리스트)”인 경우.

- **가능한 원인 2 — 타이밍/레이스**  
  사용자가 재진입 직후 빠르게 탭했을 때, 아직 최신 게임 상태가 반영되지 않은(또는 초기화 직후의) 스냅샷으로 `playMove`가 호출되었을 수 있음.  
  그러면 `snapshot.game.moves`가 아직 비어 있거나, “빈 보드”용 스냅샷으로 legal 검사가 호출됨.

- **가능한 원인 3 — 빈 보드 첫 수**  
  실제로 “빈 보드에서 첫 수”를 두려고 탭했지만, 그 경로에서 `playMove`에 넘기는 스냅샷이 `moves = emptyList()`로만 구성되어 있고, `withMoveInts`는 “현재까지의 수 순서”를 넘기는 용도라 **길이 0을 허용하지 않는** 현재 구현과 맞지 않을 수 있음.  
  (빈 보드 첫 수는 legal이면 통과시켜야 하는데, 지금은 배열 길이 0 자체가 막혀 있음.)

위 세 가지가 **로그만 보고 말할 수 있는 “현재 상태”에 대한 설명**이다.  
코드 수정은 하지 않고, 원인 가능성만 정리한 것.

---

## 4. 로그에서 보이는 기타 사항

- **엔진 헬스체크/크래시**: 이번 수집 구간에는 **없음**. 예전에 보이던 `Engine Health Check Failed: Returned empty analysis (Zombie State)` 는 이번 로그에는 없음.
- **시스템/타 앱 에러**: `thermal_core`, `ccci_fsd`, `SatelliteController`, `ImsSettingsProvider` 등은 KataGo 앱과 무관한 기기/통신 로그임.
- **앱 크래시**: 이번 구간에서는 **앱이 죽지 않음**. `isLegal_failed` / `isLegal_retry_failed` 는 로그만 남기고 실패 처리되는 경로로, 착수만 막고 있음.

---

## 5. 한 줄 요약

- **현재 상태**: 앱은 동작 중이고, 사용자가 보드에 탭해 착수를 시도했을 때 **`ensureMoveIsLegal`에서 `snapshot.game.moves`가 비어 있어 `NativeSafetyGate.withIntArray(0)` 예외가 나고**, 그 결과 **해당 수가 두어지지 않음**.
- **원인 후보**: (1) 백그라운드 복귀 후 게임 스냅샷 불일치, (2) 재진입 직후 탭으로 인한 스냅샷 타이밍/레이스, (3) 빈 보드 첫 수 경로에서 `moves`가 비어 있는데 `withIntArray(0)`을 허용하지 않는 구현.

수정 방향까지 포함한 대응은 별도로 요청해 주시면, 이 분석을 전제로 제안하겠습니다.
