# 로그로 확인한 현재 대국 봇 정보 (실제 기기 로그 기준)

아래는 **실제 adb logcat으로 수집한 com.katago.android 로그**에서 추출한 내용입니다.

---

## 1. 엔진 / 모델

```
Engine session=PLAY (PLAY=대국엔진 ANALYSIS=분석엔진) model=b10.bin
```

- **세션**: PLAY (대국엔진)
- **모델 파일**: **b10.bin**

---

## 2. 적용 중인 파라미터 (play.cfg)

대국 시작 시(레벨 28 기준) 기록된 값:

```
play.cfg written: chosenMoveTemp=1.0 wideRootNoise=0.192 cpuct=9.45 cpuctLog=9.45 cpuctBase=3720.0 avoidRepeatUtility=1.8840001 staticScoreUtility=0.82500005 maxVisits=100000
```

| 파라미터 | 값 |
|----------|-----|
| chosenMoveTemperature | 1.0 |
| wideRootNoise | 0.192 |
| cpuctExploration | 9.45 |
| cpuctExplorationLog | 9.45 |
| cpuctExplorationBase | 3720.0 |
| avoidRepeatedPatternUtility | 1.8840001 |
| staticScoreUtilityFactor | 0.82500005 |
| maxVisits | 100000 |

---

## 3. 대국 시작 시 로그 (레벨/시간/기력 관련)

```
startGame: levelIndex=28 levelLabel=초보 4 humanColor=WHITE botIsBlack=true timeRange=1.0521739s~1.0869565s randTemp=2.0900002 avoidUtility=1.8840001 scoreUtility=0.82500005
```

- **레벨 인덱스**: 28
- **표시 라벨**: 초보 4
- **제한 시간 구간**: 약 1.05s ~ 1.09s
- **randTemp (chosenMoveTemperature 적용 전 보간값 등)**  
  - 로그 상 randTemp=2.09, 실제 play.cfg에는 1.0으로 기록됨 (레벨별 적용 로직 참고)

---

## 4. 방문수 (분석 단위)

`AnalysisUseCase: Analysis Metrics` 로그에서 확인된 방문수 예:

| 시각 | visits | 소요 시간 | vps |
|------|--------|-----------|-----|
| 19:36:49 | 130 | 3.17s | 41.0 |
| 19:36:54 | 104 | 1.56s | 66.8 |
| 19:36:56 | 78 | 2.22s | 35.2 |
| 19:37:00 | 134 | 3.03s | 44.2 |
| 19:37:04 | 83 | 2.32s | 35.8 |
| 19:37:07 | 134 | 2.75s | 48.7 |
| 19:39:12 | 88 | 2.27s | 38.7 |
| 19:39:15 | 134 | 2.89s | 46.4 |

---

## 5. 실제 착수한 수의 승률·방문수 (봇이 선택한 수)

`selectBotMove` 로그 — 봇이 **실제로 선택해 둔 수**의 승률·방문수:

| 시각 | level | tier | pool | **pick winrate** | **visits** |
|------|-------|------|------|------------------|------------|
| 19:36:52 | 28 | 고수1~ | 6 | **0.456** | **23** |
| 19:36:56 | 28 | 고수1~ | 9 | **0.462** | **7** |
| 19:37:04 | 28 | 고수1~ | 7 | **0.480** | **11** |
| 19:39:12 | 28 | 고수1~ | 7 | **0.489** | **7** |

- **winrate**: 흑 기준 승률 (0~1). 예: 0.456 = 45.6%.
- **visits**: 그 수가 선택될 때의 (해당 후보의) 방문수.

---

## 6. TopSuggestion (1수) 승률

`TopSuggestion winrate` 로그 — 해당 국면에서 **1수로 선택된 좌표**의 승률:

| moves | side | coord | black(winrate) | display |
|-------|------|-------|----------------|---------|
| 0 | B | (15,3) | 0.467 | 0.467 |
| 3 | W | (4,16) | 0.456 | 0.544 |
| 5 | W | (15,15) | 0.493 | 0.507 |
| 7 | W | (4,16) | 0.472 | 0.528 |

---

## 7. 이후 로그로 확인하는 방법

1. **앱 PID 확인**  
   `adb shell pidof com.katago.android`

2. **해당 앱 로그만 저장**  
   `adb logcat -d --pid=<PID> -t 5000 > app_log.txt`

3. **찾으면 되는 키워드**
   - 엔진/모델: `Engine session=PLAY` `model=`
   - 파라미터: `play.cfg written:`
   - 레벨/시간: `startGame: levelIndex=`
   - 봇 착수 승률/방문수: `selectBotMove: ... winrate= ... visits=`
   - 1수 승률: `TopSuggestion winrate:`
   - 분석 방문수: `Analysis Metrics: visits=`

이 문서는 위와 같은 **실제 로그 라인**을 기준으로 작성되었습니다.
