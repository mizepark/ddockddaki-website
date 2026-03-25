# 근본 원인: 추천수 안 뜨고, "해당 지점에는 둘 수 없습니다" 나오는 이유

수정 제안이 아니라 **원인 설명만** 정리한 문서다.

---

## 1. "추천수 안 뜨는" 근본 원인

### 1.1 로그에서 보이는 현상

캡처 로그(`device_log_root_cause.txt`)에서 다음이 **반복**된다:

```
KataGoJNI: extractAnalysisData: zero suggestions after filter n=64 totalVisits=134 maxVisits=1 <=1=64 eq1=... eq0=...
```

- `analyzeTop4`는 끝까지 돌아가서 **Search finished visits=134** (또는 1006 등)으로 **루트 총 방문 수**는 쌓인다.
- 그런데 **필터 통과 후 suggestion 개수 = 0**이라서, 앱으로 나가는 추천이 하나도 없다.

### 1.2 필터가 무엇인가 (코드)

**파일**: `app/src/main/cpp/katago_jni.cpp` — `extractAnalysisData()`

- `search->getAnalysisData(...)` 로 후보 노드들을 가져온 뒤, `numVisits` 기준으로 정렬한다.
- **실제로 앱으로 넘기는 건 `numVisits > 1` 인 노드만** 이다.
  - `if (data[i].numVisits <= 1) continue;` 로 **1 이하 방문 수**인 후보는 전부 제외.
- 로그의 `m` 은 이렇게 **2 이상 방문한 노드 개수**이고, 여기서 `m == 0` 이면 위 경고를 찍고 **추천 0개**를 반환한다.

즉, **“분석은 돈다(루트 방문 수는 오른다)”** 하지만 **“어떤 한 수에도 방문이 2번 이상 쌓이지 않는다”** 상태다.

### 1.3 시간/방문 제어는 이미 하고 있음

- **분석 모드·대국 모드 모두** `budgetMs`(시간)와 `maxVisits`(방문 상한)로 제어하고 있다.  
  - `EngineViewModel.triggerAnalysis`에서 `budgetMs`, `maxVisits` 설정 → `AnalysisUseCase`에서 `maxTimeSeconds = budgetMs/1000`으로 넘김.  
  - 네이티브 `analyzeTop4`는 `reqMaxTime`(시간)과 `jMaxVisits`(방문 상한) 둘 다 받아서, **둘 중 먼저 도달하는 쪽**에서 검색을 멈춘다.
- 즉, “시간 제한 없이 방문만 쓰는” 구조가 아니라 **시간 제한 + 방문 상한**으로 이미 제어 중이다.

### 1.4 그런데도 추천 0개가 나는 이유

- **방문 상한이 작을 때** (예: QUICK에서 `visitsBudgeter.computeAdaptiveVisits`가 128 같은 값을 주는 경우, 또는 설정에서 작은 maxVisits를 쓰는 경우):
  - 루트 **총 방문 수**가 128(또는 1000)에서 끊긴다.
  - 합법 수가 수십 개(예: 60개)면, 128번의 플레이아웃이 **많은 수에 1~2번씩만** 분산된다.
  - 그 결과 **자식 노드마다 numVisits가 1 이하**로만 쌓이고, **numVisits > 1** 인 노드가 하나도 없을 수 있다.
- 또는 **검색 국면과 UI 국면이 어긋나면**, 방문이 우리가 기대하는 “현재 추천”에 몰리지 않고 흩어져서, 역시 자식별 방문이 1 이하로만 나올 수 있다.

정리하면:

- **직접 원인**: JNI 쪽에서 **“numVisits > 1 인 노드만 추천으로 씀”** → 그런 노드가 0개면 추천 0개.
- **그렇게 만드는 조건**:  
  - 시간/방문 제어는 하고 있지만, **방문 상한(예: 128)이 작아서** 루트 방문이 많은 수에 나뉘어 **자식당 1 이하**가 되거나,  
  - 검색 국면과 UI 국면이 어긋나서 방문이 분산되면,  
  **필터를 통과하는 추천이 0개**가 됨 → 추천수/참고도/힌트가 안 뜨는 것처럼 보임.

---

## 2. "해당 지점에는 둘 수 없습니다" 나오는 근본 원인

이 메시지(`move_illegal`)는 **다음 세 경우에만** 설정된다.

1. **좌표가 바둑판 밖**  
   `coord.x/y` 가 `boardSize` 범위 밖.
2. **그 좌표에 이미 돌이 있음**  
   `snapshot.game.stones` 에 해당 좌표가 있음.
3. **엔진 합법성 검사 실패**  
   `ensureMoveIsLegal(...)` 이 `false` 를 반환.

실제로 빈 칸을 눌렀는데도 메시지가 뜬다면, 대부분 **3번**이다.

### 2.1 ensureMoveIsLegal 이 false 를 주는 경로

- **앱**: `EngineViewModel.ensureMoveIsLegal()`  
  - `isModelReady == false` 이면 **엔진 안 부르고 true** (일단 허용).  
  - `isModelReady == true` 이면 `engineFacade.isLegal(session, ints, coord, ...)` 호출.
- **네이티브**: `katago_jni.cpp` — `nativeIsLegal()`  
  - `g_eval == null` 이면 **즉시 false** 반환하고 로그: `"isLegal: Engine not initialized"`.
  - 그 외에는 전달된 수순(`movesXY`)으로 보드를 만들고, 해당 좌표가 합법인지 `hist.isLegal(board, q, next)` 로 판단.

따라서 **“실제로는 둘 수 있는데 illegal 로 나오는”** 경우는 다음이 가능하다.

1. **네이티브 엔진 미초기화**  
   앱은 `isModelReady == true` 로 알고 있는데, 엔진 프로세스 쪽에서는 **g_eval 이 null** (한 번도 init 성공 안 했거나, `NativeBridge_shutdown` 등으로 리셋된 뒤 아직 재초기화 전).  
   → `isLegal` 이 무조건 false → **“해당 지점에는 둘 수 없습니다”**.
2. **수순/패스 인코딩 불일치**  
   `nativeIsLegal` 은 전달된 `movesXY` 로만 보드를 만든다.  
   루프에서 `if (x < 0 || y < 0) break;` 로 **패스(-1,-1)를 “끝”으로만 처리**하고, 패스를 한 수로 적용하지 않는다.  
   → 수순에 패스가 있으면 **보드/next 플레이어가 앱과 어긋남** → 잘못된 국면에서 합법성 판단 → 일부 정상 수가 illegal 로 나올 수 있음.
3. **예외**  
   `isLegal` 내부에서 예외 발생 시 `getOrElse { false }` 로 false 반환 → 역시 같은 메시지.

로그에는 **"isLegal: Engine not initialized"** 가 이번 캡처에는 없지만,  
**NativeBridge_shutdown / Engine state cleared** 뒤 재 init 되기 전에 사용자가 착수를 시도하면 1번 상황이 발생할 수 있다.

---

## 3. 분석 모드에서 “착수도 안 된다”고 느끼는 이유

- **참고도 모드** (`isVariationAnalysisMode == true`) 일 때는 **추천수 밖의 칸을 누르면** `onCellClicked` 에서 **아무 것도 안 하고 return** 한다.  
  → “착수 시도 자체가 무시”되므로, “둘 수 없다”기보다 “반응이 없다”에 가깝다.
- 그 외에는 `playMove(coord)` 까지 가고, 위 2절처럼 **ensureMoveIsLegal 이 false** 이면 **optimistic update 를 되돌리면서** `move_illegal` 메시지를 띄운다.

즉, **“착수도 안 된다”** 는 경험은  
- 참고도 모드: **추천 밖 클릭 무시** (착수 시도 자체가 없음),  
- 일반 착수: **엔진 합법성 검사에서 false** (2절의 1~3)  
두 가지가 섞여 있을 수 있다.

---

## 4. 대국 모드에서도 "해당 지점에는 둘 수 없습니다" 나오는 이유

- 대국 모드에서의 사용자 착수도 **같은 `playMove` → ensureMoveIsLegal** 경로**를 탄다.  
  따라서 **2절과 동일**:  
  - 엔진이 **g_eval == null** 인 순간이 있거나,  
  - **수순/패스 인코딩** 이 맞지 않거나,  
  - **예외**로 false 가 나오면  
  동일한 메시지가 뜬다.
- 봇이 둔 수에 대한 “해당 지점에는 둘 수 없습니다”는, 봇이 illegal 한 수를 두었다고 판정되면 **되돌리면서** 같은 메시지를 쓸 수 있는 경로가 있다.

---

## 5. 요약 표

| 현상 | 근본 원인 (코드/구조) |
|------|----------------------|
| **추천수 안 뜸** | 분석/대국 모두 **시간(budgetMs) + 방문 상한(maxVisits)** 로 이미 제어 중. 그런데 **방문 상한이 작을 때**(예: 128) 루트 방문이 수십 개 수에 나뉘어 **자식당 numVisits ≤ 1**만 쌓임 → `extractAnalysisData` 의 **numVisits > 1** 필터를 통과하는 노드가 0개 → 추천 0개. |
| **"해당 지점에는 둘 수 없습니다"** | `ensureMoveIsLegal` → 네이티브 `isLegal` 이 **false** 를 반환하는 경우. 가능 원인: **(1) g_eval == null (엔진 미초기화/리셋)** **(2) 패스 포함 수순 인코딩 불일치로 보드/턴 어긋남** **(3) 예외 시 false 반환.** |
| **분석 모드 착수 “안 됨”** | 참고도 모드에서 추천 밖 클릭은 **의도적으로 무시**; 그 외는 위 illegal 경로와 동일. |
| **대국 모드에서도 같은 메시지** | 사용자/봇 착수 모두 **같은 ensureMoveIsLegal 경로** 사용 → 위와 동일 원인. |

---

## 6. 로그로 추가로 확인하면 좋은 것

- **추천 0개**:  
  `extractAnalysisData: zero suggestions after filter` 와 함께 **totalVisits, maxVisits, <=1** 값을 보면, “루트는 방문이 쌓이는데 자식은 전부 ≤1” 인지 바로 확인 가능.
- **착수 illegal**:  
  `isLegal: Engine not initialized` 가 **착수 시도 시점**에 찍히는지 확인하면, **g_eval null** 여부를 구분할 수 있다.
- **엔진 리셋**:  
  `NativeBridge_shutdown: Engine state cleared` 직후 ~ 재 init 전에 착수하면 1번(미초기화) 상황이 재현될 수 있다.

이 문서는 **원인 규명용**이며, 수정 방법은 별도로 정할 수 있다.
