# 근본 원인 분석 (수정 없음, 원인 파악만)

**목적**: 증상에 대한 당장의 땀빵이 아니라, 왜 그런 현상이 나오는지 **구조적·근본 원인 후보**만 정리한다. 수정 제안은 하지 않음.

---

## 1. 대국 모드: “해당 지점에는 둘 수 없다” (모든 착수 거절)

### 구조
- 대국(PLAY)은 **KataGoPlayService** (`:engine_play` 프로세스)에만 연결된다.
- `EngineBootstrap`에서 `playRepository = EngineRepository(app, KataGoPlayService::class.java)` 로 **별도 바인딩**.
- `ensureMoveIsLegal` → `engineFacade.isLegal(Session.PLAY, ...)` → `playController.isLegal(...)` → **Play 서비스** 원격 호출.

### 근본 원인 후보
1. **PLAY 서비스 쪽 엔진이 초기화되지 않음**  
   `ensureEngineReady`에서 `initSession(Session.PLAY, playBoardSize)`가 호출되려면 **Play 서비스가 이미 바인딩**되어 있어야 함.  
   - AppStart 시 analysisRepository 바인딩과 playRepository 바인딩이 **동시에** 진행되는데, Play 쪽 바인딩이 늦거나 실패하면 `playController.initEngine()`이 호출되지 않거나 실패할 수 있음.  
   - 로그에서 PID 2202가 반복해서 "Engine not initialized"를 찍었다면, 그 프로세스가 `:engine_play`일 가능성이 있음. 그 프로세스에서 `initEngine`이 한 번도 성공하지 않았거나, 초기화 전에 isLegal이 호출되고 있음.

2. **프로세스 분리로 인한 초기화 순서/실패 가림**  
   메인 앱은 “엔진 준비 완료”를 **두 서비스 모두** 기준으로 보지 않고, `ensureEngineReady` 한 번 성공만 보면 Ready로 전환할 수 있음.  
   - 분석(:engine)만 초기화되고 대국(:engine_play)은 바인딩 지연/실패로 init이 안 됐어도, ensureReady는 true를 반환할 수 있음.  
   - 그 결과 UI는 “준비됨”인데, 대국 탭에서만 isLegal이 항상 실패(또는 예외 → false) → “둘 수 없다”.

3. **requestId / movesHash 불일치**  
   네이티브 `isLegal`은 특정 보드 상태(requestId, movesHash)에서 초기화된 검색 객체를 전제로 함.  
   - PLAY 쪽에서 initEngine은 “빈 보드”나 특정 수순으로 한 번만 호출되고, 이후 사용자 착수에 따른 보드가 전달될 때 requestId/movesHash가 네이티브와 맞지 않으면 항상 false를 돌려줄 수 있음.  
   - (분석 쪽은 runAnalysis 시마다 같은 requestId/movesHash로 검색이 돌아가지만, PLAY는 “합법수만 검사”하는 경로라 검색 객체/상태가 어떻게 유지되는지 코드 경로 확인 필요.)

**검증 방법**  
- `:engine_play` 프로세스 로그만 따로 수집 (`adb logcat --pid=<engine_play_pid>`).  
- "KataGoPlayService bound", "initEngine requested", "Native HOME set to" 등이 있는지, 그리고 그 **이후**에 isLegal 호출이 오는지 확인.  
- Play 서비스가 한 번도 initEngine을 받지 못했거나, init 실패 직후 isLegal만 호출되는 패턴이면 1번이 근본 원인.

---

## 2. 분석 모드: 참고도/방문수가 안 올라가다가 한참 뒤에야 올라감

### 구조
- 분석은 **KataGoService** (`:engine` 프로세스)만 사용.
- 단일 이벤트 큐: `AppStart` → `ensureEngineReady` → 성공 시 `Ready` → 그 다음에야 `StartAnalysis` 이벤트가 처리됨.
- 중간 결과: 네이티브가 100ms 주기로 콜백, Kotlin에서 120ms 스로틀 + totalVisits&lt;2 드롭.

### 근본 원인 후보
1. **첫 분석이 시작되기까지의 지연**  
   - `EngineOrchestrator`는 이벤트를 **한 번에 하나씩** 처리함.  
   - `AppStart`(ensureEngineReady)가 **끝날 때까지** 다음 이벤트(StartAnalysis)는 처리되지 않음.  
   - ensureEngineReady 안에서: 모델 확인 → (필요 시) prepareModel → **analysis 쪽 서비스 바인딩 + initEngine** → **play 쪽 서비스 바인딩 + initEngine** → 헬스체크 등.  
   - 이 전체가 끝나기 전에는 분석이 **한 번도 시작되지 않음**.  
   - 따라서 “한참 기다리면 올라간다”의 한참 = **앱 시작 후 초기화가 끝나기까지의 시간**일 가능성이 큼.  
   - 근본 원인: **초기화와 첫 분석 트리거가 같은 직렬 큐에 묶여 있어서, 초기화가 길면 사용자가 보는 “첫 반응”이 그만큼 늦어짐.**

2. **세션/탭과 표시되는 분석 결과의 불일치**  
   - 사용자가 “분석” 탭에 있는데, 내부적으로 analyzingSession이 PLAY로 잡혀 있거나, 분석 결과가 ANALYSIS 세션으로 오는데 UI는 PLAY 세션 기준으로 방문수를 안 그리도록 되어 있으면 “안 올라가는 것처럼” 보일 수 있음.  
   - 또는 첫 분석 intent가 ANALYSIS가 아니라 PLAY로 나가고, PLAY 쪽 서비스가 미초기화라 결과가 비어 오면 방문수 0으로만 보일 수 있음.  
   - (이건 “근본 원인”보다는 상태 머신/세션 전환 버그에 가까우며, 로그로 intent/session/requestId를 추적해야 함.)

3. **중간 콜백이 오기 전까지의 빈 구간**  
   - 네이티브는 검색이 돌아가기 시작한 뒤 **100ms마다** 중간 결과를 보냄.  
   - Kotlin은 totalVisits&lt;2면 UI에 반영하지 않음.  
   - 따라서 “방문수가 올라간다”고 느끼는 시점 = 최소 2 이상의 totalVisits가 오는 첫 콜백 시점.  
   - 이 자체가 “근본 원인”이라기보다는, 1번(초기화 지연)과 겹치면 “아무 반응 없음 → 한참 뒤 갑자기 올라감”으로 체감됨.

**검증 방법**  
- 로그에 `EngineOrchestrator process state=Idle event=AppStart` 시각과 `process state=Ready event=StartAnalysis` 시각의 차이 = 사용자가 체감하는 “한참”.  
- 그 사이에 prepareModel, 바인딩, initEngine(ANALYSIS), initEngine(PLAY) 등이 모두 포함되어 있는지 확인.

---

## 3. 리소스 오류 (Invalid resource ID 0x00000000, No package ID 6a)

### 관찰
- 로그: `E .katago.android: Invalid resource ID 0x00000000`, `No package ID 6a found for resource ID 0x6a0b0013`.
- 0x6a = 패키지 인덱스 106 등, 다른 패키지(라이브러리) 리소스를 앱 컨텍스트에서 참조하려다 실패한 형태로 해석됨.

### 근본 원인 후보
- **어딘가에서 리소스 ID를 0으로 넘기거나**, 빌드/병합 과정에서 해당 ID가 앱 패키지에 없이 참조되고 있음.  
- 또는 **동적 로딩된 모듈(AdMob/GMS 등)** 이 자신의 패키지 리소스를 쓰는데, 호출 컨텍스트가 메인 앱이라 패키지가 6a로 해석되는 등, 컨텍스트/테마 불일치.  
- “아무것도 안 된다”가 UI가 빈 화면이거나 특정 화면에서만 크래시라면, 이 리소스 오류가 그 화면 그리기 경로에서 발생했을 가능성 있음.

**검증 방법**  
- 0x00000000을 사용하는 코드 경로 검색 (setImageResource(0) 등).  
- 0x6a0b0013이 어떤 리소스인지 (라이브러리 심볼/리소스 맵) 확인하고, 그 리소스를 참조하는 호출부가 메인 앱인지 라이브러리인지 구분.

---

## 4. 요약 표 (근본 원인 후보만)

| 증상 | 근본 원인 후보 | 검증 방향 |
|-----|----------------|-----------|
| 대국 모드 모든 착수 “둘 수 없다” | (1) :engine_play 쪽 initEngine 미호출/실패. (2) PLAY 바인딩이 늦거나 실패해도 ensureEngineReady는 성공으로 처리됨. (3) isLegal 시 requestId/movesHash 불일치. | engine_play PID 로그에서 bound/initEngine/isLegal 순서와 실패 유무 확인. |
| 분석 방문수 한참 뒤에야 올라감 | (1) 단일 이벤트 큐에서 AppStart(초기화)가 끝나기 전엔 StartAnalysis가 처리되지 않음 → 초기화 길이 = 체감 “한참”. (2) 첫 중간 결과가 totalVisits&lt;2로 드롭되는 구간. | AppStart 완료 시각 ~ 첫 StartAnalysis 처리 시각 차이, 및 첫 유효 콜백 시각 측정. |
| 리소스 오류 / UI 개판 | 리소스 ID 0 또는 다른 패키지 리소스(6a)를 잘못된 컨텍스트에서 참조. | 0 사용처 검색, 6a 리소스 참조 경로 확인. |

---

이 문서는 **원인 파악용**이며, 위 검증으로 근본 원인을 좁힌 뒤에만 구조적 수정을 하는 것이 좋음.
