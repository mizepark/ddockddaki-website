# 현재 대국모드 봇 로그 분석 결과 (실제 기기 로그 기준)

캡처 시각: 02-04 00:49~00:51 (앱 PID 22295)

---

## 1. 기력(레벨)

| 항목 | 값 |
|------|-----|
| **levelIndex** | **30** |
| **levelLabel** | **초보 6** (최저기력 구간) |
| **humanColor** | WHITE (사람 백, 봇 흑) |
| **botIsBlack** | true |

---

## 2. 엔진 종류

```
Engine session=PLAY (PLAY=대국엔진 ANALYSIS=분석엔진) model=b10.bin
```

- **세션**: PLAY (대국엔진)
- **모델 파일**: **b10.bin**

---

## 3. 적용 파라미터 (play.cfg — 대국 시작 후 기록값)

```
play.cfg written: chosenMoveTemp=1.0 wideRootNoise=0.2 cpuct=10.0 cpuctLog=10.0 cpuctBase=4000.0 avoidRepeatUtility=1.96 staticScoreUtility=0.85 maxVisits=100000
```

| 파라미터 | 값 |
|----------|-----|
| chosenMoveTemperature | 1.0 |
| wideRootNoise | 0.2 |
| cpuctExploration | 10.0 |
| cpuctExplorationLog | 10.0 |
| cpuctExplorationBase | 4000.0 |
| avoidRepeatedPatternUtility | 1.96 |
| staticScoreUtilityFactor | 0.85 |
| maxVisits | 100000 |

---

## 4. 대국 시작 시 로그 (timeRange, randTemp 등)

```
startGame: levelIndex=30 levelLabel=초보 6 humanColor=WHITE botIsBlack=true timeRange=1.0s~1.0s randTemp=2.2 avoidUtility=1.96 scoreUtility=0.85
```

- **제한 시간**: 1.0s ~ 1.0s (착수당 약 1초)
- **randTemp** (레벨 보간값): 2.2
- **avoidUtility**: 1.96
- **scoreUtility**: 0.85

---

## 5. 낮은 승률 선택 적용 여부

**적용됨 (레벨 30 = 최저기력)**

로그 라인 예:
```
selectBotMove: level=30 tier=최저기력 pool=1 pick=lowest winrate=... visits=...
```

- **tier**: 최저기력
- **pick**: lowest (최저 승률 후보 선택)
- **pool**: 1 (해당 국면에서 최하승률 5% 풀에 들어간 후보가 1개였음)

---

## 6. 방문수 (Analysis Metrics)

로그에 찍힌 구간별 방문수 예:

| 시각 | visits | time | vps |
|------|--------|------|-----|
| 00:49:32 | 162 | 3.02s | 53.6 |
| 00:49:39 | 172 | 3.14s | 54.8 |
| 00:49:42 | 113 | 2.22s | 50.9 |
| 00:49:44 | 106 | 2.33s | 45.5 |
| 00:49:47 | 101 | 2.14s | 47.3 |
| 00:49:51 | 79 | 1.78s | 44.5 |
| … | 68~134 | 0.9~3.1s | 42~150 |

- **범위**: 약 68 ~ 172 (국면·시간 제한에 따라 변동)

---

## 7. 실제 착수한 수의 승률·방문수 (봇이 선택한 수)

selectBotMove `pick=lowest` 로그에서 추출한 **실제 착수 수**의 winrate / visits:

| 시각 | pick winrate | visits |
|------|----------------|--------|
| 00:49:36 | 0.435 (43.5%) | 2 |
| 00:49:42 | 0.433 (43.3%) | 2 |
| 00:49:47 | 0.394 (39.4%) | 2 |
| 00:49:51 | 0.386 (38.6%) | 4 |
| 00:49:57 | 0.626 (62.6%) | 2 |
| 00:50:13 | 0.600 (60.0%) | 7 |
| 00:50:19 | 0.561 (56.1%) | 15 |
| 00:50:24 | 0.454 (45.4%) | 2 |
| 00:50:29 | 0.402 (40.2%) | 2 |
| 00:50:36 | 0.322 (32.2%) | 2 |
| 00:50:42 | 0.265 (26.5%) | 5 |
| 00:50:46 | 0.182 (18.2%) | 2 |
| 00:51:00 | 0.264 (26.4%) | 12 |
| 00:51:05 | 0.097 (9.7%) | 2 |
| 00:51:10 | 0.085 (8.5%) | 133 |

- 레벨 30이므로 **항상 최저 승률 후보**를 골라 둔 것이며, 국면에 따라 8.5% ~ 62.6% 구간의 수가 선택됨.

---

## 8. 요약

| 항목 | 내용 |
|------|------|
| **기력** | 레벨 30 (초보 6, 최저기력) |
| **엔진** | PLAY, model=b10.bin |
| **파라미터** | chosenMoveTemp=1.0, wideRootNoise=0.2, cpuct=10, cpuctLog=10, cpuctBase=4000, avoidRepeatUtility=1.96, staticScoreUtility=0.85, maxVisits=100000 |
| **제한 시간** | 1.0s ~ 1.0s |
| **방문수** | 국면당 약 68~172 (로그 구간 기준) |
| **낮은 승률 선택** | **적용됨** (tier=최저기력, pick=lowest) |
| **실제 착수** | 매 수 최하승률 후보 1개 풀에서 선택, winrate 8.5%~62.6%, visits 2~133 |

이 문서는 위 로그 라인들을 기준으로 작성되었습니다.
