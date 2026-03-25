# 스트레스 테스트 100,000회 실행 방법

대국 모드에서 **모든 기력(0~30)·엔진·이벤트**를 섞어 100,000 스텝 자동 스트레스 테스트를 실행하는 방법입니다.

## 테스트 항목 (전체 — 말씀하신 내용 전부 반영)

| 구분 | 항목 | 방법 |
|------|------|------|
| **자동 10만회** | 대국 모드: 기력 0~30 순환, 새 게임, 무르기/다시두기, 게임으로 돌아가기, 힌트 켜기/해제, 맨 처음/끝 점프, 합법 수 랜덤 착수(300수까지), 300수 시 새 게임 | 아래 ADB로 100,000 스텝 실행 |
| **수동 확인** | 앱 종료 시 "엔진 복구에 실패했습니다" 재시작 프롬프트 **미표시** (버그 수정 검증) | 테스트 전/후에 앱 한 번 종료해 보고 다이얼로그 안 뜨는지 확인 |
| **수동 확인** | 분석 모드·왼쪽 가장자리 탭 정상 (터치/그리드 정렬 수정 검증) | 테스트 전/후에 분석 화면 진입 → 왼쪽 가장자리 등 착수·분석 정상 동작 확인 |

## 준비

- 기기 연결 (`adb devices`)
- 클린 빌드 후 번들툴로 설치 (아래 참고)

## 권장: 전체 사이클 한 번에 (캐시 삭제 → 클린 빌드 → 설치 → 10만회 테스트)

**이전 데이터/캐시가 섞이지 않도록** 캐시·빌드·기존 앱을 지우고, **최신 코드만으로** 클린 빌드한 뒤 번들툴로 기기에 설치하고, 곧바로 10만회 스트레스 테스트를 시작하려면:

```powershell
# 프로젝트 루트에서 (캐시/이전 데이터 삭제 → 클린 빌드 → 설치 → 10만회 테스트 시작)
.\stress_test_full_cycle.ps1
```

- **테스트가 종료된 뒤** 로그에 나오는 **COVERAGE**에서 **0인 항목 = 해당 기능이 테스트되지 않은 것 = 로직에서 막혀 있을 가능성**입니다.
- **0인 기능이 있으면** → 원인 파악 후 코드 수정 → **같은 스크립트를 다시 실행** (캐시 삭제·클린 빌드·설치·10만회 테스트 반복).

```powershell
# 테스트 종료까지 대기 후, COVERAGE 로그와 "미테스트 기능(0인 항목)" 목록을 자동 출력
.\stress_test_full_cycle.ps1 -WaitForCoverage
```

- `-WaitForCoverage`: 테스트가 완료되거나 중단될 때까지 logcat 대기 후, COVERAGE 한 줄과 **0인 항목 목록**을 콘솔에 출력합니다. (최대 6시간 타임아웃)

## 빌드 및 설치 (수동 시)

```powershell
# 프로젝트 루트에서 (버전 자동 +1, 클린 빌드, 번들툴 설치)
.\build_and_install.ps1
```

또는 수동:

```powershell
.\gradlew --stop
# app\build, app\.cxx, .gradle 캐시 등 삭제 후
.\gradlew bundleProdDebug --no-build-cache --rerun-tasks --no-daemon
java -jar bundletool.jar build-apks --bundle=app/build/outputs/bundle/prodDebug/app-prod-debug.aab --output=app.apks --connected-device --overwrite
java -jar bundletool.jar install-apks --apks=app.apks
```

## 토큰 없이 10만회 테스트 (자동 부여)

스트레스 테스트 인텐트(`STRESS_TEST`)로 진입 시 **50만 토큰을 자동 부여**하므로, 토큰 부족 없이 10만 스텝이 진행됩니다.

## 검증 (정상완료 조건)

스트레스 테스트는 **시도 횟수만 세는 것이 아니라** 아래를 검증합니다.

- **착수**: 매 착수 시도 후 500ms 뒤 `game.moves.size`가 증가했는지 확인. 증가하지 않으면 **거절**로 집계(`playMoveRejected`). "해당 지점에는 둘 수 없습니다" 메시지가 뜨는 경우가 여기 해당.
- **힌트**: 힌트 트리거 후 7초 뒤 `topSuggestions`에 PV가 있는지 확인. 5회 이상 힌트를 트리거했는데 한 번도 PV가 없으면 **힌트 분석 결과 없음**으로 간주.

**정상완료가 아닌 경우** (테스트 실패):
- 착수 거절 비율이 50% 초과이거나
- 힌트를 5회 이상 트리거했는데 `hintHadPv`가 0이면

`AssertionError`를 던지고, `StressTestProgress.error`에 사유를 넣으며, 로그에 `검증 실패 (정상완료 아님)`이 출력됩니다. COVERAGE 로그에는 `playMoveRejected`도 포함됩니다.

## 스트레스 테스트 실행 (100,000회)

### 방법 1: ADB 인텐트로 자동 시작

앱을 **대국 화면으로 띄우고** 스트레스 테스트를 바로 시작하려면:

```bash
adb shell am start -n com.katago.android/.MainActivity -a com.katago.android.STRESS_TEST --ei steps 100000
```

- 앱이 켜지면서 **대국(PLAY)** 화면으로 이동하고, 곧바로 **100,000 스텝** 스트레스 테스트가 시작됩니다.
- 세션이 아직 시작되지 않았으면 먼저 기력 0으로 게임을 시작한 뒤 테스트가 진행됩니다.

### 방법 2: 앱에서 수동

1. 앱 실행 → 홈에서 **대국** 진입
2. (선택) 기력 선택 후 게임 시작
3. 아래 ADB로 테스트 시작 (동일):

```bash
adb shell am start -n com.katago.android/.MainActivity -a com.katago.android.STRESS_TEST --ei steps 100000
```

(앱이 이미 대국 화면이면 포그라운드만 되고, 인텐트가 전달되면 그 시점에 스트레스 테스트가 시작됩니다. **재진입 시에만** 인텐트가 적용되므로, 대국 화면에서 테스트를 넣으려면 한 번 홈으로 갔다가 다시 위 명령으로 실행하는 것이 안전합니다.)

## 테스트 내용 (한 스텝 = 1회)

매 스텝마다 **확률적으로** 다음 중 하나가 실행됩니다.

| 확률 | 동작 |
|------|------|
| 1% | 기력 변경(0~30 순환) 후 새 게임 시작 |
| 2% | 무르기(Undo) |
| 1% | 다시 두기(Redo) |
| 1% | 게임으로 돌아가기(참고도/힌트 해제) |
| 1% | 힌트 켜기 → 곧바로 게임으로 돌아가기 |
| 2% | 맨 처음으로 / 맨 끝으로 점프 |
| 나머지 | 합법 수 중 랜덤 착수 (300수 미만일 때) 또는 300수 도달 시 새 게임(기력 순환) |

- **기력**: 0(프로) ~ 30까지 순환하며 새 게임을 시작합니다.
- **300수**가 되면 자동으로 다음 기력으로 새 게임을 시작합니다.
- 모든 엔진/설정은 앱에서 사용하는 그대로 사용됩니다 (레벨별 play.cfg·분석 등).

## 로그 확인 (돌발 상황 포착)

**권장 필터 (엔진 재초기화·오류·충돌·비정상 흐름 모두 잡기):**
```bash
adb logcat -s StressTest:I StressTest:W StressTest:E EngineRepo:W
```

- `StressTest: started totalSteps=100000 levels=0..30 maxMovesPerGame=300` — 테스트 시작
- `StressTest: step=N total=100000 level=X moves=Y engineReady=true|false status=...` — 1,000 스텝마다 진행 + **엔진 준비 여부·상태 메시지**
- `StressTest: playMove blocked: insufficient tokens` — 토큰 부족으로 수 차단
- `StressTest: ABORTED step=N error=...` — 예외로 테스트 중단 (스택 트레이스 포함)
- `EngineRepo: markServiceDead: engine died, will rebind in 800ms` — 엔진 프로세스 죽음·재바인딩
- 완료 시 `StressTest: Completed steps=100000`

**기능별 테스트 여부 (COVERAGE):**  
테스트 종료 시(정상 완료 또는 ABORTED) 한 번씩 `StressTest: COVERAGE [Completed]` 또는 `COVERAGE [ABORTED]` 로그가 출력됩니다.  
각 항목은 **해당 기능이 실행된 횟수**입니다. **0이면 그 기능은 테스트가 진행되지 않은 것**이므로, 로직이 막혀 있거나 조건이 만족되지 않았을 가능성이 있습니다.

| 항목 | 의미 |
|------|------|
| hintTriggered | 힌트 요청 횟수 |
| hintHadPv | 힌트 분석 결과에 PV가 있어 참고도 열림 |
| hintNoPv | 힌트 요청했지만 PV 없음(참고도 미오픈) — **계속 0이면** 엔진이 수순을 안 준 것 |
| refOpenSteps | 참고도가 열린 상태에서 진행된 스텝 수 |
| refNext, refPrev, refJumpStart, refJumpEnd, refToggle | 참고도 내 next/prev/점프/재생 토글 |
| refReturnToGame, refStartGame, refLevel, refUndo, refRedo | 참고도 열린 채 돌발(닫기/새게임/레벨변경/무르기/다시두기) |
| levelChange, undo, redo, jumpStart, jumpEnd, playMove, newGame | 일반 스텝에서의 레벨변경/무르기/다시두기/점프/착수/새게임 |

**추가로 전체 앱 로그 보기:**
```bash
adb logcat | findstr /i "StressTest EngineRepo PlayMode EngineViewModel KataGo"
```

## 오류/미테스트 시 사이클 (테스트 → 원인 파악·수정 → 클린 빌드·설치 → 10만회 재테스트)

1. **COVERAGE에서 0인 항목** 또는 **ABORTED/예외** 확인  
   - 0인 항목 = 해당 기능 미테스트 = 로직 막힘 가능성 → 원인 파악 후 수정  
   - `StressTest: ABORTED` 또는 엔진/앱 예외 확인
2. **코드 수정** 후, **이전 데이터가 섞이지 않도록** 전체 사이클로 재실행:
   ```powershell
   .\stress_test_full_cycle.ps1
   ```
   (캐시·빌드·앱 삭제 → 클린 빌드 → 번들툴 설치 → 10만회 테스트 자동 시작)
3. 테스트 종료 후 다시 COVERAGE 확인. 0인 항목이 없을 때까지 1~2 반복.

## 예상 소요 시간

- 스텝당 약 80ms 대기 + 착수/엔진 처리 시간이므로, 100,000 스텝은 **기기 성능에 따라 수 시간** 걸릴 수 있습니다.
- 중단하려면 앱을 종료하거나 (백버튼/최근 앱에서 제거) 코루틴이 취소되도록 하면 됩니다. (필요 시 `stopStressTest()` 호출하는 UI 추가 가능.)

## 권장 테스트 순서 (전체 항목 포함)

1. **앱 종료 확인** — 앱 실행 후 한 번 종료해 보기 → "엔진 복구에 실패했습니다" 메시지가 뜨지 않으면 OK.
2. **분석/왼쪽 가장자리 확인** — 분석 화면에서 왼쪽 가장자리 탭·착수 정상 동작 확인.
3. **100,000회 스트레스 테스트 실행** — 아래 명령으로 자동 진행.
4. **(선택) 테스트 후 다시 앱 종료** — 재시작 프롬프트 미표시 재확인.

## 요약

1. **전체 사이클 (권장)**  
   `.\stress_test_full_cycle.ps1`  
   → 캐시/이전 데이터 삭제 → 클린 빌드 → 설치 → 10만회 테스트 시작  
   → 테스트 종료 후 COVERAGE에서 0인 항목 있으면 원인 수정 후 같은 스크립트 다시 실행.
2. **테스트만 따로**  
   `.\build_and_install.ps1` 후  
   `adb shell am start -n com.katago.android/.MainActivity -a com.katago.android.STRESS_TEST --ei steps 100000`
3. 로그로 진행/COVERAGE 확인 → `adb logcat -s StressTest:*`
