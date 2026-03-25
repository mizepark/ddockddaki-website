# 앱 내 방문수(visits) 관련 기능 설명

인벤토리(VISIT_CONSTRAINTS_INVENTORY.md)에 나온 항목을 **기능 단위로** 설명한 문서.

---

## 1. 추천(분석) 쪽

### 1.1 AnalysisUseCase — minVisits (minShare 기반)

- **역할**: 엔진이 준 추천 후보 중 **몇 방문 이상인 수만** UI 추천으로 쓸지 정하는 문턱.
- **동작**:
  - `maxVisitsValue` = 현재 후보들 중 **최대 방문수**.
  - `minShare` = 레벨/구간에 따라 정해지는 **비율** (예: 0.001~0.01). 대국 레벨이 있으면 `minShareForPlayLevel`, 없으면 `maxVisitsValue < 500` 이면 0.001, 아니면 0.01.
  - `maxVisitsValue` 가 1~499 구간이면 `minShare`를 500 기준으로 보정해서 더 완만하게 만듦.
  - **minVisits = (maxVisitsValue × minShare)**. (예: maxVisitsValue=2000, minShare=0.01 → 20)
  - `s.visits >= minVisits` 인 후보만 남기고, 방문수 내림차순 정렬 후 상위 64개만 최종 추천으로 씀.
- **의미**: “가장 많이 탐색된 수 위주로만 보여주고, 너무 적게 탐색된 수는 잘라낸다”는 품질 필터. 1짜리 강제 제한은 없음(minShare만 적용).

### 1.2 EngineViewModel — maxVisits = 0

- **역할**: **분석 한 번 돌릴 때** 엔진에 “방문 수 상한”을 넘기지 않고, **시간만** 쓰게 하는 설정.
- **동작**: `runAnalysisInternal` 안에서 항상 `maxVisits = 0` 으로 두고 `engineFacade.runAnalysis(..., maxVisits = 0)` 호출. 네이티브에서는 `jMaxVisits == 0` 이면 “시간 제한만 사용”으로 해석하고 `maxVisits = 1000000` 등 큰 값으로 두어, 실질적으로 **budgetMs(시간)** 만으로 분석이 끝나게 함.
- **의미**: 앱 분석은 “최대 N방문”이 아니라 “최대 N초” 기준으로 동작한다는 뜻.

### 1.3 EngineTypes — visitsMin=16, visitsMax=2000

- **역할**: 엔진/분석 **설정·상태**를 나타낼 때 쓰는 **기본 숫자**.
- **동작**: `EngineTuningState`(또는 관련 상태)의 `visitsMin`, `visitsMax` 기본값. UI에서 “최소/최대 방문” 범위를 보여주거나, 다른 모듈이 “아직 값을 안 정했을 때” 쓸 수 있는 기본값.
- **의미**: 방문수 관련 UI·설정의 **초기값/범위 표시용**. 실제 분석 호출의 상한은 위 1.2처럼 0(시간 제한)으로만 가져감.

---

## 2. 대국 봇 쪽

### 2.1 PlayViewModel — BOT_MIN_SUGGESTION_VISITS = 1

- **역할**: 봇이 **어떤 수를 후보로 허용할지**의 최소 방문수 문턱.
- **동작**: `selectBotMove` 에서 `state.analysis.topSuggestions` 를 쓸 때  
  `it.visits >= BOT_MIN_SUGGESTION_VISITS` 인 수만 남김. 1이므로 **방문수 1 이상**이면 후보로 인정. 그 다음 합법수 필터·기력별 선택 로직만 적용.
- **의미**: “엔진이 1번이라도 탐색한 수는 봇 후보로 쓸 수 있다.” 더 낮추면 0방문도 허용하게 되고, 올리면 더 보수적으로만 둠.

### 2.2 PlayViewModel — BOT_RECOVERY_MIN_VISITS = 5

- **역할**: **복구** 상황에서 “이 정도는 탐색돼 있어야 한다”는 최소 방문수 기준(상수 정의).
- **동작**: 상수만 정의돼 있고, 주석상 “복구 시 최소 방문수”. 실제로 어디서 이 값으로 비교하는지는 코드에서 한 군데만 쓰이거나, 추후 복구 로직에서 5 미만이면 재분석 등으로 쓸 수 있도록 둔 값으로 보임.
- **의미**: 봇이 실패/복구 모드로 갈 때 “너무 적게 돌린 분석”은 믿지 말고, 최소 5 방문은 있어야 한다는 가이드라인.

### 2.3 기력별 가중치에 방문수 사용

- **역할**: 봇이 **프로가 아닌 레벨**에서 “실수하는 수”를 고를 때, **방문수가 적을수록** 더 골라지게 만드는 가중치.
- **동작**: `pickGradualStrength` 등에서 후보 풀을 만든 뒤, 각 후보 `s` 에 대해  
  `(1 / (s.visits + 1))^visitPower` 같은 식으로 가중치를 줌. `visitPower` 는 레벨에 따라 0.2~1.2 구간으로 보간. 방문수 적은 수일수록 가중치가 커져서, 같은 승률이면 “덜 탐색된 수”가 더 자주 선택됨.
- **의미**: 기력을 낮출수록 “덜 읽힌 수 = 실수에 가까운 수”를 더 골라서, 인간처럼 실수하는 느낌을 내는 용도.

---

## 3. 설정·UI 쪽

### 3.1 SettingsRepository — getMaxVisits / setMaxVisits (1~3500)

- **역할**: 사용자가 설정한 **“최대 방문수”** 값을 저장·조회할 때 쓰는 범위 제한.
- **동작**: `getMaxVisits()` 는 SharedPreferences 의 `max_visits_dynamic` 을 읽고, **1~3500** 으로 `coerceIn`. `setMaxVisits(visits)` 도 같은 범위로 잘라서 저장. 다른 화면/기능에서 “동적 max visits” 를 참고할 때 이 값을 씀.
- **의미**: 앱 전역에서 “사용자 정의 방문 상한”을 1~3500 사이로만 두어, 비정상적으로 작거나 큰 값이 들어가지 않게 함.

### 3.2 VisitsBudgeter — visitsMin.coerceAtLeast(16), CPU 시 상한 500/200

- **역할**: **목표 지연 시간**과 **ms/visit**을 이용해 “이번에 쓸 수 있는 방문 수”를 **한 번** 계산하는 유틸.
- **동작**:
  - `computeAdaptiveVisits(targetLatencyMs, lastMsPerVisit, ...)` 에서  
    `minV = visitsMin.coerceAtLeast(16)` → **최소 16** 방문은 보장.
  - CPU 모드면 `maxVisitsCap` 을 8코어 이상이면 500, 아니면 200 으로 둠. GPU면 `visitsMaxCap`(설정 상한) 그대로 사용.
  - `visitsRaw = (budget × safetyFactor) / msPerVisit` 로 계산한 뒤 `coerceIn(minV, maxV)` 로 클램프.
- **의미**: “지연 시간 목표에 맞추면서, 최소 16은 돌리고, CPU일 때는 200/500으로 상한을 낮춘다”는 **분석 예산 산정** 규칙.

### 3.3 Board — 방문수 표시·정규화 (60~99 점수)

- **역할**: 바둑판 위에 **추천별 상대 점수(60~99)** 를 그릴 때, 방문수를 비율로 바꿔서 씀.
- **동작**: 현재 보이는 추천 리스트에서 `maxVisits` = 1번 후보 방문수, `minVisits` = 마지막 후보 방문수(또는 동일 시 max와 같게).  
  각 후보 `s` 에 대해 `(s.visits - minVisits) / (maxVisits - minVisits)` 로 0~1 비율을 구한 뒤, 60~99 점으로 매핑해 “99”, “75” 같은 문자열로 그림.
- **의미**: 사용자에게 “이 수가 상대적으로 얼마나 많이 읽혔는지”를 60~99 점으로만 보여주는 **표시용** 정규화.

### 3.4 PlayBoardSection — 방문수 1000 이상일 때 참고도 팝업

- **역할**: **참고도(변화) 팝업**을 “분석이 어느 정도 돌았을 때만” 띄우기 위한 조건.
- **동작**: `isInitialAnalysisReady` 와 함께, 사실상 “분석이 준비됐다”고 볼 수 있을 때만 `VariationPopupOverlay` 를 보여줌. 주석/문맥상 **방문수가 1000 이상**일 때 준비된 것으로 보는 로직이 있으면, 그때만 참고도 팝업이 뜨고 “준비 중” 메시지와 겹치지 않게 함.
- **의미**: 너무 초반(방문 적을 때)에는 참고도 팝업을 안 띄워서, “준비 중”과 겹치는 현상을 막는 **UI 조건**.

### 3.5 ScoreUtils — normalizedVisits(visits, maxVisits)

- **역할**: **방문수를 0~1** 구간으로 정규화해, 다른 점수와 합성할 때 씀.
- **동작**: `visits / maxVisits` 를 하고, `maxVisits <= 0` 이면 분모를 1로 해서 0~1 로 `coerceIn`. `composite(wNorm, vNorm)` 에서 승률 정규값과 방문 정규값을 합쳐 60~99 점수로 바꿀 때 사용.
- **의미**: “방문수 비율”을 공통 스케일(0~1)로 만들어, 여러 UI/점수 계산에서 일관되게 쓰기 위한 유틸.

---

## 4. 네이티브 쪽

### 4.1 analyzeTop4 — jMaxVisits 1~2 이고 비헬스체크면 빈 결과

- **역할**: **분석**으로 들어온 호출인데, 방문 상한이 1~2로 들어온 경우를 “잘못된 경로”로 보고 막음.
- **동작**: `jMaxVisits` 가 1 또는 2이고 `jRequestId != -999` 이면 (헬스체크는 -999) 로그 남기고 **빈 Float 배열** 반환. 헬스체크만 2로 호출하는 것이 정상이므로, 분석 요청이 실수로 1~2로 들어오면 결과를 쓰지 못하게 함.
- **의미**: “1~2방문만 돌린 결과가 분석 결과로 쓰이는” 경로를 원천 차단.

### 4.2 analyzeTop4 — jMaxVisits=0 이면 maxVisits=1000000

- **역할**: **시간 제한만** 쓰는 분석일 때, 방문 수로는 거의 끊기지 않게 함.
- **동작**: `jMaxVisits == 0` 이면 `localParams.maxVisits = 1000000` (및 필요 시 maxPlayouts 도 그에 맞춤). 실제 종료는 `maxTime`(budgetMs 기반)으로만 일어나게 함.
- **의미**: Kotlin 에서 `maxVisits=0` 으로 “시간만 써라”고 넘긴 것과 짝을 이루는 네이티브 쪽 처리.

### 4.3 extractAnalysisData — 1짜리 필터 제거

- **역할**: (과거) 자식 노드 중 `numVisits <= 1` 인 수는 결과 배열에 안 넣었음. **(현재) 그 필터 제거** → 모든 자식 후보를 결과에 포함.
- **의미**: “1로 잘라서 안 보여주는” 동작을 없앤 부분. 인벤토리/이전 문서와 맞춤.

### 4.4 getPolicy — maxVisits=1, maxPlayouts=1

- **역할**: **정책망만** 빠르게 보고 싶을 때 쓰는 API.
- **동작**: `nativeGetPolicyHeatmap` 등에서 `localParams.maxVisits = 1`, `maxPlayouts = 1` 로 짧게만 검색. MCTS가 거의 돌지 않고 정책 출력만 가져옴.
- **의미**: “착수 분포 히트맵” 같은 것만 필요할 때, 방문 수는 1로 두고 빠르게 반환.

### 4.5 getRootLead / 소유권 — maxVisits=128 등

- **역할**: **리드(점수 차)** 나 **소유권 히트맵**을 구할 때, 검색량을 작게 제한.
- **동작**: 해당 JNI 함수들에서 `localParams.maxVisits = 128`, `maxPlayouts = 128` 등으로 설정해, 전체 분석보다 훨씬 적은 탐색만 수행.
- **의미**: “전체 수 읽기”가 아니라 “리드/소유권만 대략 보기”용이라, 128 같은 작은 상한으로 충분하다고 본 설정.

---

## 5. 설정·에셋 쪽

### 5.1 analysis.cfg — maxVisits=1600

- **역할**: **분석**용 KataGo 설정 파일의 기본 방문 상한.
- **동작**: 앱이 analysis.cfg 를 로드하면 `maxVisits=1600` 이 들어가고, `g_globalParams` 등에 반영. 다만 실제 분석 호출은 Kotlin 에서 maxVisits=0 으로 하므로, 네이티브에서 100만으로 덮어쓰고 시간 제한만 쓰는 경우가 많음.
- **의미**: “설정 파일 기본값”으로 1600이 있고, 필요 시 다른 경로(예: 다른 진입점)에서 이 값이 쓰일 수 있음.

### 5.2 play.cfg / AssetInstaller — maxVisits

- **역할**: **대국**용 설정(play.cfg)을 기기에 쓸 때, **maxVisits** 값을 넣어 줌.
- **동작**: AssetInstaller 가 play.cfg 를 생성·덮어쓸 때, 기기 성능/설정에 맞춰 `maxVisits = 100000` 등 큰 값을 넣음. 로그에 “play.cfg written: ... maxVisits=...” 형태로 남음.
- **의미**: 대국 엔진이 “최대 몇 방문까지 둘 수 있는지” 설정 파일에서 읽을 수 있게 하는 값.

---

이 문서는 위 인벤토리 항목에 대한 **기능별 설명**만 담고 있으며, 변경 이력은 VISIT_CONSTRAINTS_INVENTORY.md 를 참고하면 됨.
