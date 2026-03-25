# 엔진 결과가 1로 잘려 나오는 원인 후보 (조사용)

조건/강제 추가 없이, **왜 1로 잘리는지** 찾기 위한 후보만 나열함.

---

## 0. "1 이하 막기" vs 로그의 1 — 왜 둘 다 있을 수 있는지

**막는 대상**과 **로그에 찍히는 값**이 다름.

| 구분 | 의미 |
|------|------|
| **막는 것** | **후보(수 추천) 리스트**에서 방문수 ≤1인 수를 넣지 않음. → 추천 후보가 비거나 줄어듦. (자살수/노이즈 방지용 최소 장치) |
| **로그에 1이 찍히는 것** | **totalVisits**(루트 방문수). 후보 필터와 **별개**. 엔진이 실제로 수행한 전체 방문수라서 1이어도 그대로 찍힘. |

즉, "방문수 1 이하를 막는다" = 후보에 1짜리 수를 넣지 않는다.  
"로그에 1이 찍힌다" = 검색이 거의 시작하자마자 중단돼 루트 방문수가 1에서 끝났다.  
→ **후보 필터가 있어도, 검색이 즉시 중단되면 totalVisits=1 로그는 계속 나올 수 있음.**

(방문수 1 이하 필터는 제거됨. 인벤토리: docs/VISIT_CONSTRAINTS_INVENTORY.md)

---

## 0-1. 로그 maxVisits 의미 (오해 방지)

`extractAnalysisData` 로그의 **maxVisits** 는 **검색 상한이 아님**.

- **의미**: 추출한 자식 노드들(`data[i].numVisits`) 중 **최댓값**.
- 코드: `int maxVisits = 0; for(...) if (data[i].numVisits > maxVisits) maxVisits = data[i].numVisits;`
- 따라서 `maxVisits=1` = “자식 노드 방문수 최대가 1” → `numVisits > 1` 필터 후 추천 0개.

**totalVisits**: 해당 시점의 루트 총 방문수(플레이아웃 수).  
예: `totalVisits=2, maxVisits=1` → 검색이 **2번**만 돌고 멈춤 + 그 2가 서로 다른 자식에 1씩 분배.

---

## 1. 검색이 1에서 멈추는 조건 (KataGo search.cpp)

- `runWholeSearch` 안에서 `shouldStop = true` 가 되는 경우:
  - `numPlayouts >= maxPlayouts`
  - `numPlayouts + numNonPlayoutVisits >= maxVisits`
  - `hasMaxTime && numPlayouts >= 2 && timeUsed >= maxTime`
  - `shouldStopNow` (stopAnalysis 등으로 세팅)

→ **원인 후보**: 실제 검색에 쓰이는 `searchParams.maxPlayouts` 또는 `searchParams.maxVisits` 가 1이면 1에서 멈춤.

---

## 2. searchParams 가 1이 되는 경로

- **Search 객체 생성 시**: `g_search = std::make_unique<Search>(g_globalParams, ...)`  
  → `g_globalParams` 에 maxPlayouts/maxVisits 가 1이면, 생성 직후에는 1.
- **analyzeTop4 에서**: `localParams = g_globalParams` 후  
  `localParams.maxVisits = (jMaxVisits > 0) ? jMaxVisits : 1000000`  
  `localParams.maxPlayouts = max(localParams.maxVisits, g_globalParams.maxPlayouts)`  
  그 다음 `searchToUse->setParams(localParams)` 호출.

→ **원인 후보**:
- **설정 파일**: init 시 읽는 설정 파일(analysis.cfg / play.cfg)에 `maxPlayouts = 1` 또는 `maxVisits = 1` 이 있으면 `g_globalParams` 가 1로 로드됨.  
  (지금 앱이 쓰는 analysis.cfg 템플릿에는 없음. 단, **기기 실제 파일**에 다른 값이 있거나 예전 버전이 쓴 파일이 남아 있을 수 있음.)
- **setParams 가 안 불리는 경로**: analyzeTop4 가 아니라 다른 진입점(헬스체크, getPolicy 등)에서만 검색이 돌고, 그쪽은 maxVisits=1, maxPlayouts=1 로 고정돼 있음. 그 결과가 “분석 결과”로 쓰이면 1로 잘려 나옴.

---

## 3. 설정 로드 (Setup::loadParams, setup.cpp)

- `cfg.contains("maxPlayouts")` 일 때만 `params.maxPlayouts` 설정.
- 없으면 **설정 파일로는 안 건드림** → `SearchParams` 기본값 유지.  
  기본값: `searchparams.cpp` 에서 `maxPlayouts = (1<<50)`.

→ **원인 후보**:  
- 기기에서 읽는 **실제 설정 파일 경로**와 **그 파일 내용** 확인 필요.  
  (assets 에서 복사한 것인지, ensureConfigFile 으로 덮어쓴 것인지, 수동/다른 앱이 만든 파일이 섞였는지.)

---

## 4. setParams 호출 여부 (search.cpp)

- `Search::setParams(SearchParams params)` → `clearSearch(); searchParams = params;`
- analyzeTop4 에서는 `searchToUse->setParams(localParams)` 후 `setPosition` 호출.

→ **원인 후보**:  
- 같은 Search 인스턴스에 **setParams 호출 후** 다른 스레드/코드에서 **다시 searchParams 를 1로 덮어쓰는지** 여부.  
- 또는 **analyzeTop4 가 아닌** (maxVisits=1, maxPlayouts=1 쓰는) **다른 API**가 호출되고, 그 반환값이 “분석 결과”로 쓰이는 경로가 있는지.

---

## 5. 헬스체크 / 다른 API (katago_jni.cpp)

- **initEngine 워밍업**: `wParams.maxVisits = 5`, `maxPlayouts = 1` 등으로 짧게만 검색.
- **getPolicy (nativeGetPolicyHeatmap)**: `localParams.maxVisits = 1`, `maxPlayouts = 1`, `maxTime = 0.02`.
- **헬스체크 (KataGoEngineFacade)**: `maxVisits = 2` 로 analyzeTop4 호출.

→ **원인 후보**:  
- “분석 결과”로 사용하는 쪽이 **실제 분석 요청의 반환값**이 아니라  
  **헬스체크/ getPolicy / 워밍업** 등 **1 또는 2 플레이아웃만 도는 경로의 반환값**을 쓰고 있으면, 1로 잘려 나온 것처럼 보임.

---

## 6. 프로세스/설정 분리 (ANALYSIS vs PLAY)

- ANALYSIS 프로세스: analysis.cfg  
- PLAY 프로세스: play.cfg  
- 각각 init 시 **자기 설정만** 로드 → `g_globalParams` 가 프로세스마다 다름.

→ **원인 후보**:  
- 분석 요청이 **PLAY 프로세스**로 가는데, 그 프로세스의 설정(또는 그 프로세스에서만 쓰는 캐시/검색 객체)에 maxPlayouts=1 이 들어가 있거나,  
- ANALYSIS 프로세스는 정상인데 **결과를 받는 쪽**이 PLAY 프로세스 응답(또는 헬스체크 응답)을 “분석 결과”로 잘못 쓰는 경우.

---

## 7. 확인하면 좋은 것 (원인 찾기용)

1. **기기 실제 설정 파일**  
   - `getConfigFile()` / `getPlayConfigFile()` 이 가리키는 경로의 **analysis.cfg, play.cfg 내용**  
   - 특히 `maxPlayouts`, `maxVisits` 라인 존재 여부와 값.
2. **analyzeTop4 진입 여부**  
   - 분석 버튼/자동 분석 시 **실제로 analyzeTop4**가 호출되는지,  
   - 그때 넘기는 `maxVisits`(0이면 시간 제한만) 값.
3. **어떤 Search 가 도는지**  
   - analyzeTop4 에서 `searchToUse` 가 `g_search` 인지, 임시 Search 인지.  
   - `setParams(localParams)` 직후와 `runWholeSearch` 직전에 **실제 searchParams.maxPlayouts / maxVisits** 로그로 확인.
4. **반환값이 어디서 오는지**  
   - “1로 잘린 결과”가 **analyzeTop4 의 반환값**인지,  
   - **다른 API(헬스체크, getPolicy 등)** 의 반환값인지,  
   - **다른 프로세스(PLAY)** 의 응답인지.

---

## 8. 원인 정리 (로그/코드 기준)

| 현상 | 가능 원인 |
|------|-----------|
| **totalVisits=2, maxVisits=1** (검색이 2에서 멈춤) | (1) 헬스체크는 `maxVisits=2` 로 analyzeTop4 호출. 콜백은 requestId/movesHash 로 걸러지므로 정상이라면 섞이지 않음. (2) 같은 세션에서 검색 상한이 2로 설정된 경로. (3) 조기 stop 또는 매우 짧은 타임아웃. |
| **totalVisits=7~18, maxVisits=1** (검색은 돌았지만 자식당 최대 1) | 방문수가 여러 자식에 분산. 예: QUICK maxVisits=128, 자식 64개. 100ms 중간 콜백 시점에 7~18만 쌓여 있으면 자식당 최대 1일 수 있음. "아직 쌓일 때 중간 스냅샷" 가능성. |
| **최종이 아니라 중간 결과만 적용** | 분석 중 취소/타임아웃/네비게이션으로 runAnalysis 가 끝나기 전에 종료되면, 마지막 적용이 100ms 중간 콜백일 수 있음. 그 시점의 totalVisits(2~수십), maxVisits=1 이 UI에 남음. |

---

## 9. 방법 제안 (의견)

### (A) 방문수 1짜리 제거 — 한 곳에서만, 다른 로직에 영향 없이

- **목표**: 1-visit 후보는 어떤 경로로 오든 **한 번만** 걸러서 제거하고, totalVisits·winrate·lead 등 다른 값에는 손대지 않기.
- **현재**: 네이티브 `extractAnalysisData`(numVisits≤1 제외) + Kotlin `AnalysisUseCase.parseRawResult`(minVisits 최소 2) 두 곳에서 걸러짐.
- **제안 1 — 단일 관문을 Kotlin 파싱 한 곳으로**
  - **유일한 진입점**: 엔진 float 배열 → 추천 리스트으로 가는 경로는 `AnalysisUseCase.parseRawResult` 하나뿐임.
  - 여기서 **추천 하나씩 만들 때** `visits <= 1` 이면 리스트에 넣지 않기만 하면, 네이티브가 실수로 1을 보내도 앱에서는 절대 1짜리 추천이 나오지 않음.
  - totalVisits·winrate·lead·stones 는 그대로 두면 되므로 **다른 로직 무영향**.
  - 선택: 네이티브 쪽 `numVisits <= 1` 필터는 유지(이중 방어)하거나, “한 곳만” 원하면 네이티브에서는 제거하고 Kotlin 한 곳만 쓰기.
- **제안 2 — 단일 관문을 네이티브 한 곳으로**
  - “1 제거”는 **네이티브 extractAnalysisData 에서만** 하고, Kotlin 의 minVisits 는 **1이 아닌 다른 목적**(예: 레벨별 minShare)만 쓰고 `coerceAtLeast(2)` 제거.
  - 그러면 “1 이하 막기”는 네이티브 한 곳뿐. Kotlin 은 “상대적” 필터만 해서 다른 로직과 역할이 섞이지 않음.

### (B) 1로 잘리는 현상 — 원인 찾아서 제거

- **목표**: 검색이 1~2에서 끊기거나, 결과가 1짜리로만 나오는 **원인**을 찾고 제거.
- **제안 1 — 진입 시 로그로 원인 수집**
  - `analyzeTop4` 진입 시 `jMaxVisits`, `jMaxTimeSeconds`, `requestId`(또는 헬스체크 여부) 한 줄 로그.
  - 기기에서 totalVisits=1 로그 나올 때 같은 프로세스 로그에 jMaxVisits 가 1/2 인지 1000 인지 보면, “헬스체크/잘못된 호출” vs “검색 조기 중단” 구분 가능.
- **제안 2 — 분석 경로는 상한 보장**
  - 분석용 호출(헬스체크 제외)일 때만, `jMaxVisits` 가 0이면 시간 제한만 쓰는 것이므로 문제없고, 1~2 같은 값이 들어오면 **로그 경고 + 빈 결과 반환** 같이 “실패로 인지”하게 하면, 잘못된 호출이 분석 결과로 쓰이지 않게 할 수 있음. (상한을 강제로 올리는 게 아니라 “이건 분석이 아니다”라고 처리.)
- **제안 3 — 최종 결과만 UI에 반영**
  - 중간 콜백(100ms) 결과는 “진행 중 미리보기”로만 쓰고, **실제 채택은 runAnalysis 가 반환한 최종 결과 한 번만** 하도록 하면, “검색이 끝나기 전에 취소돼서 중간 스냅샷(방문수 적음)이 최종처럼 남는” 경우를 줄일 수 있음. (이미 requestId/movesHash 로 걸러지지만, “최종이 왔을 때만 suggestions 덮어쓰기” 같은 규칙을 명확히 하는 것.)

---

## 10. 원인 조사 결과 (코드 추적)

코드 기준으로 **검색이 1에서 멈추는 경로**만 정리함.

### 10.1 분석 경로에서 넘기는 maxVisits

- **EngineViewModel.runAnalysisInternal**: 항상 `maxVisits = 0` (시간만 사용).
- **runAnalysis** → **analysisUseCase.analyze** → **analyzeTop4(ints, maxTimeSeconds, maxVisits=0, ...)**.
- **헬스체크만** `maxVisits = 2`, `requestId = -999` 로 호출. (이미 비헬스체크 1~2 호출은 네이티브에서 빈 결과 반환하도록 처리함.)

→ 정상 분석 호출은 **jMaxVisits=0** 이고, 네이티브에서 `localParams.maxVisits = 1000000`, `localParams.maxPlayouts = max(1000000, g_globalParams.maxPlayouts)` 로 설정됨.

### 10.2 검색이 1에서 멈추는 조건 (search.cpp)

`runWholeSearch` 에서 `shouldStop = true` 가 되는 경우:

1. `numPlayouts >= maxPlayouts`
2. `numPlayouts + numNonPlayoutVisits >= maxVisits`
3. `hasMaxTime && numPlayouts >= 2 && timeUsed >= maxTime`
4. **`shouldStopNow`(g_stopFlag)** — 명시적 중단

jMaxVisits=0 이면 1·2번은 최소 100만이라서, **1 플레이아웃에서 멈추려면 4번뿐**임.

### 10.3 결론: 1로 잘리는 원인

- **원인**: **g_stopFlag** 가 검색 시작 직후 true 가 되어, `shouldStopNow` 로 곧바로 루프가 끝남.
- **g_stopFlag** 는 **stopAnalysis()** (NativeBridge → KataGoService) 에서만 true 로 세팅됨.
- stopAnalysis 는 다음 때 호출됨:
  - 분석 취소/새 분석 시작 시 (EngineViewModel: analysisJob 취소 시 ANALYSIS/PLAY 둘 다 stop)
  - 엔진 오케스트레이터·앱 생명주기에서 엔진 정리할 때

즉, **“엔진에서 1로 잘려 나오는” 현상은**  
- 분석이 시작된 뒤 **거의 동시에** stopAnalysis 가 호출되거나,  
- (이론상) 같은 프로세스에서 **이전 분석이 끝나기 전에** stop 하고 새 분석이 같은 g_stopFlag 를 쓰는 타이밍이 겹칠 때  
**검색이 1~2 플레이아웃만 하고 중단**되면서 발생할 수 있음.

### 10.4 확인 권장 사항

- 기기 로그에서 **totalVisits=1 (또는 2)** 가 나올 때, 직전에 **analyzeTop4 진입 로그**(jMaxVisits, requestId) 와 **stopAnalysis 로그**가 찍혀 있는지 확인.
- 같은 시점에 **새 분석 시작/취소/화면 전환**이 있었는지 확인하면, 조기 stop 호출 여부를 판단할 수 있음.

---

이 문서는 **원인 후보·확인 포인트·방법 제안·조사 결과**를 담았으며, 실제 적용은 선택 사항임.
