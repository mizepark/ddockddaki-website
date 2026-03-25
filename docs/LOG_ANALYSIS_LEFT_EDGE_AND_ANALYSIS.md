# 로그 정확 분석: 처음 실행 문제 → 왼쪽 가장자리 착수 불가 → 분석 모드 현상

## 사용자 진술

- 처음에는 실행이 제대로 되지 않았다.
- 이어서 **바둑판의 왼쪽 가장자리에 착수가 안 되는** 현상이 있었다.
- 그 다음 **지금의 (분석 모드) 현상**이 생겼다.

아래는 각 단계를 로그·코드와 맞춰 정리한 **근본 원인**이다.

---

## 1. “처음 실행이 제대로 되지 않다”

### 로그 근거 (device_play_log_current.txt, 02-04 00:46:46)

```
02-04 00:46:46.601 22295 22336 E .katago.android: No package ID 6a found for resource ID 0x6a0b0013.
02-04 00:46:46.688 22295 22295 E .katago.android: Invalid resource ID 0x00000000.
02-04 00:46:46.723 22295 22295 E .katago.android: Invalid resource ID 0x00000000.
```

- **No package ID 6a found for resource ID 0x6a0b0013**  
  다른 패키지(예: WebView/광고 등) 리소스가 앱 컨텍스트에서 참조되면서 리소스 ID 해석 실패.  
  → 특정 UI가 안 그려지거나 초기화 중 예외로 “실행이 제대로 안 된다”처럼 보일 수 있음.
- **Invalid resource ID 0x00000000**  
  null/미설정 리소스 참조.  
  → 테마·아이콘·drawable 등이 0으로 넘어가면 그리기 실패·깨진 화면·초기 진입 불안정으로 이어질 수 있음.

추가로 같은 시점 로그에 `Choreographer: Skipped 58 frames!`가 있어, 메인 스레드 과부하(모델 복사 등)로 첫 화면이 버벅이거나 “잘 안 켜진다”는 느낌을 줄 수 있다.

**정리**: “처음 실행이 제대로 되지 않다”의 원인은,  
- 리소스 ID 오류(다른 패키지 리소스 참조, 0x00000000 참조)와  
- 초기 프레임 드롭  
이 겹친 것으로 보는 것이 로그와 맞다.

---

## 2. “바둑판 왼쪽 가장자리에 착수가 안 된다”

### 코드 근거 (Board.kt)

- **그리드 그리기(보이는 바둑판)**  
  `calculateBoardGeometry()`  
  - `marginPx = 10f * density`  
  - 주석: “외곽 여백 10dp 추가”  
  - → 그리드가 화면 가장자리에서 **10dp 안쪽**으로 그려짐.

- **터치 → 좌표 변환(착수 판단)**  
  `mapPointerToCoord()`  
  - 주석: “Use same margin logic as calculateBoardGeometry”  
  - 실제 값: **`marginPx = 4f * density`**  
  - → 터치용 그리드는 **4dp 여백**으로 계산됨.

즉, **그리는 그리드(10dp)와 터치용 그리드(4dp)가 서로 다른 여백**을 쓰고 있다.

### 결과 (왼쪽 가장자리)

- 그리는 그리드의 **왼쪽 첫 줄**(x=0)은 화면에서 **더 오른쪽**에 있음 (10dp 여백).
- 터치용 그리드의 x=0은 **더 왼쪽**에 있음 (4dp 여백).
- 사용자가 **보이는 왼쪽 가장자리(그리드 x=0)**를 탭하면:
  - 그 픽셀 위치는, 터치용 그리드 기준으로는 **한 칸 오른쪽** 구간에 해당하고,
  - `rawGx.roundToInt().coerceIn(0, boardSize-1)` 때문에 **gx=1**로 해석됨.
- 따라서 **실제로는 (0, y)를 누른 것처럼 보이는데, 앱은 (1, y)로 착수**하게 됨.  
  → “왼쪽 가장자리(첫 번째 줄)에 착수가 안 된다” = 첫 줄을 눌러도 돌이 그 왼쪽 줄에 안 놓이고, 오른쪽 줄에 놓이거나 착수 느낌이 이상한 현상으로 이어짐.

**정리**:  
**근본 원인은 터치 좌표 계산(`mapPointerToCoord`)의 여백이 4dp이고, 그리드 그리기(`calculateBoardGeometry`)의 여백이 10dp로 서로 달라서, 보이는 바둑판과 터치 판단용 그리드가 어긋나 있는 것**이다.  
그 결과 **왼쪽 가장자리(첫 번째 세로선)를 눌렀을 때만 (0,y)가 아니라 (1,y)로 인식되는 현상**이 발생한다.

---

## 3. “그 다음 생긴 현상” (분석 모드)

### 로그 근거 (device_log_analysis_fail.txt, device_play_log 등)

- **대부분**:  
  `KataGoJNI: extractAnalysisData: empty data`  
  → 엔진에서 내려주는 분석 데이터가 비어 있음.
- **가끔 데이터가 있을 때**:  
  `extractAnalysisData: zero suggestions after filter n=64 totalVisits=1 maxVisits=0 <=1=64 eq1=0 eq0=64`  
  → totalVisits=1, 모든 후보가 방문수 ≤1이라 JNI 단에서 추천이 0개로 걸러짐.
- **속도**:  
  `Analysis Metrics: visits=1 time=5.993s vps=0.2 ms/visit=5993.00`  
  → 약 6초에 방문 1번만 쌓임.

즉,  
- 분석 결과가 비어 있거나,  
- 있어도 **방문수가 1**이라 `numVisits > 1` 필터에 걸려 추천이 0개가 되고,  
- 그 과정에서 **분석 모드 UI 잠금(`suspendUiDuringAnalysis`)이 풀리지 않아** 착수·UI가 막힌 상태가 유지될 수 있다.

**정리**:  
분석 모드 문제의 근본 원인은  
- **분석 엔진이 거의 돌지 않거나(empty / totalVisits=1)**  
- **한 번 실패·스킵될 때 UI 잠금 해제가 안 되는 설계**  
가 겹친 것이다.  
(자세한 내용은 `ANALYSIS_MODE_ROOT_CAUSE.md`, `ANALYSIS_FAILURE_ROOT_CAUSE.md` 참고.)

---

## 4. 요약 표

| 현상 | 근본 원인 (로그·코드 기준) |
|------|----------------------------|
| 처음 실행이 제대로 안 됨 | 리소스 ID 오류(No package ID 6a, Invalid resource ID 0x00000000) + Choreographer 58프레임 스킵으로 초기 화면·리소스 불안정. |
| 왼쪽 가장자리 착수 안 됨 | `mapPointerToCoord`는 4dp, `calculateBoardGeometry`는 10dp 여백 사용 → 터치 그리드와 그리드 그리기 불일치 → 왼쪽 첫 줄 터치가 gx=1로 인식됨. |
| 분석 모드 현상 | 분석 empty / totalVisits=1 + minVisits 필터로 추천 0개 + (실패·스킵 시) suspendUiDuringAnalysis 미해제. |

---

## 5. 분석 모드 “완벽 해결” 3단계 조치에 대한 의견

제안하신 **1) UI 잠금 강제 해제, 2) 트리거 완전 차단, 3) 엔진 실행 환경 안정화**는 현재 로그·코드와 맞고, 적용 순서도 적절합니다. 보완 의견만 정리합니다.

### 1) UI 잠금 강제 해제 (Safety Net)

- **의견**: 필수입니다. `EngineViewModel.handleAnalysisResult()`를 보면, **최종 결과가 비어 있을 때** `next <= MAX_EMPTY_RETRY`인 경우에만 `suspendUiDuringAnalysis = false`를 넣고(2390줄), 재시도 후 **return**합니다. 반대로 **재시도 횟수를 초과한 경우**(`next > MAX_EMPTY_RETRY`)에는 이 블록을 빠져나가지만, 그 다음 경로에서 **빈 suggestions에 대한 UI 잠금 해제가 보장되지 않을 수 있습니다**.  
  따라서 “분석 결과가 도착했고, 에러/진행 중이 아닌데 suggestions가 비어 있으면 **무조건** `suspendUiDuringAnalysis = false`”를 한 번 더 넣는 **공통 해제 지점**을 두는 것이 안전합니다.  
  예: `handleAnalysisResult()` 진입 직후 또는 `if (!isIntermediate && result.suggestions.isEmpty())` 블록 **안**에서, 재시도 여부와 관계없이 **한 번은** overlay를 `suspendUiDuringAnalysis = false`로 갱신한 뒤, 재시도/return 로직을 타도록 하면 “먹통”을 막을 수 있습니다.

### 2) 트리거 완전 차단 (Final Shield)

- **지연 실행 재검사(300ms 등)**: `analysisStateMachineJob`(425줄) 쪽에서 `runAnalysisInternal(pending)` 호출 **직전**에, “지금 의도(intent)의 바둑판 상태가 방금 분석을 시작한 상태와 동일한가?”를 한 번 더 확인하는 것은 **중복 트리거 제거**에 매우 유효합니다.  
  현재도 `samePosition(lastRunIntent)`(449줄)와 보호 구간(`MIN_ANALYSIS_PROTECT_MS`)이 있지만, `minIntervalTimeoutJob`에서 **지연 후** `runAnalysisInternal`을 호출할 때(443줄) 직전 시점의 intent만 비교하므로, 그 사이에 **동일 국면으로 또 intent가 들어온 경우**까지 막으려면 “실행 직전 재검사”를 넣는 것이 좋습니다.
- **MAX_EMPTY_RETRY = 1**: 코드에 이미 동일 requestId당 재시도 제한이 있고(2405, 2406줄), `next <= MAX_EMPTY_RETRY`로 1회만 재시도하는 구조입니다. 상수값이 1이면 제안하신 “딱 1번만 재시도”와 일치합니다. 값만 확인·고정하면 됩니다.

### 3) 엔진 실행 환경 안정화 (Engine Health)

- **DeadObjectException / 서비스 재연결**: 엔진 서비스가 죽었을 때 사용자에게만 에러를 보여주지 말고, **재바인딩·재초기화·세션 복구**를 한 번 수행하도록 하면, visits=1 또는 empty가 반복되는 상황을 줄일 수 있습니다.
- **리소스 ID 오류**: 앱 초기화 시 `Invalid resource ID 0x00000000` 등이 나오지 않도록 테마·drawable 참조를 점검하면, 엔진 초기화 직전 단계에서의 예외·지연을 줄이는 데 도움이 됩니다.

**정리**: 제안하신 세 가지는 모두 현재 구조와 잘 맞고, 특히 **1) 빈 결과일 때도 UI 잠금은 반드시 푸는 것**, **2) 분석 실행 직전 “같은 국면인지” 한 번 더 보는 것**을 코드에 명시적으로 넣으면, 로그에 visits=1이 나와도 앱이 먹통이 되는 상황을 크게 줄일 수 있습니다.

---

## 6. 왼쪽 가장자리 “포인트 자체가 이동이 안 되는” 문제 — 정확한 해결책

**증상**: 바둑판 **왼쪽 가장자리**(첫 번째 세로선, x=0)를 터치해도 **착수/미리보기 포인트가 그곳으로 이동하지 않고**, 오른쪽 한 칸(또는 아예 반응 없음)으로만 인식되는 현상.

**원인 (코드 기준)**  
- **그리드 그리기**: `Board.kt`의 `calculateBoardGeometry()`는 **`marginPx = 10f * density`**로 그리드 위치를 계산합니다(668줄).  
- **터치 → 좌표**: 같은 파일의 `mapPointerToCoord()`는 주석만 “calculateBoardGeometry와 동일”이라 되어 있고, 실제로는 **`marginPx = 4f * density`**(157줄)를 사용합니다.  
→ 터치용 그리드가 **그려진 그리드보다 6dp씩 왼쪽**으로 치우쳐 있어, **보이는 왼쪽 가장자리(그리드 x=0)**를 눌러도 터치 좌표는 **gx=1**로 해석됩니다. 그래서 “포인트가 왼쪽 가장자리로 이동하지 않는다”처럼 보입니다.

**정확한 해결책 (한 줄)**  
- **`Board.kt`의 `mapPointerToCoord()` 안에서 `marginPx`를 `calculateBoardGeometry()`와 동일하게 맞춥니다.**

**구체 수정**  
- **파일**: `app/src/main/java/com/katago/android/ui/Board.kt`  
- **위치**: `mapPointerToCoord()` 함수 내부, 156–158줄 부근  
- **현재**:  
  `val marginPx = 4f * density`  
- **변경**:  
  `val marginPx = 10f * density`  
- **이유**: 그리드를 그릴 때 쓰는 `calculateBoardGeometry()`(668줄)는 `10f * density`를 쓰므로, 터치 좌표도 **같은 여백**을 쓰면 그려진 그리드와 터치 그리드가 일치합니다.  
  그 결과 **왼쪽 가장자리(그리드 x=0)**를 탭했을 때 `gx=0`으로 계산되어, 미리보기·착수 포인트가 해당 교차점으로 정확히 이동합니다.

**검증 방법**  
- 19×19 기준, **가장 왼쪽 세로선**(x=0) 위 여러 점(예: 3·3, 9·9, 15·15)을 터치했을 때, 미리보기 돌과 최종 착수가 모두 **첫 번째 줄**에 나오는지 확인하면 됩니다.

---

## 7. 수정 방향 (참고)

- **실행 문제**: 리소스 참조가 0이거나 다른 패키지 ID를 쓰지 않도록, 테마·drawable·외부 라이브러리 리소스 참조 경로 점검.
- **왼쪽 가장자리**: 위 **§6**대로 `Board.kt` `mapPointerToCoord()`의 `marginPx`를 **`10f * density`**로 변경해 `calculateBoardGeometry()`와 동일하게 맞춤.
- **분석 모드**: §5의 3단계(UI 잠금 강제 해제, 트리거 직전 재검사·재시도 제한, 엔진 재연결·리소스 점검) 적용 + 기존 `ANALYSIS_MODE_ROOT_CAUSE.md`, `ANALYSIS_FAILURE_ROOT_CAUSE.md` 참고.

이 문서는 **로그를 정확히 확인한 뒤 코드와 대조해 원인만 기술**한 것이며, 실제 수정은 별도 작업으로 진행하면 된다.
