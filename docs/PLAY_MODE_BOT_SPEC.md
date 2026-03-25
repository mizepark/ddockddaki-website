# 대국모드(Play Mode) 봇 사양 요약

## 1. 기력(레벨) 구조

| 인덱스 | 라벨 (한국어) | 영문 | 모델 | 비고 |
|--------|----------------|------|------|------|
| 0 | 프로 | Pro | **B10** | 최강 |
| 1~6 | 초고수 1~6 | Super Expert 1~6 | **B10** | |
| 7~30 | 고수~초보 1~24 | Expert~Novice | **B10** | 단일 모델, 최하(30)=최저기력 |

- **전 레벨**: **B10** 단일 모델 (분석·대국 공용). CPU 전용 빌드 기준.

---

## 2. 엔진 종류

- 대국모드 전용 엔진: **Session.PLAY**.
- 분석 모드와 분리된 `play.cfg` + `playHomeDir` 사용.
- 로그 태그 예: `Engine session=PLAY (PLAY=대국엔진 ANALYSIS=분석엔진) model=b10.bin`.

---

## 3. 적용 파라미터 (play.cfg / 레벨별)

게임 시작 시 `startGame()`에서 레벨에 따라 설정한 뒤 `AssetInstaller.ensureConfigFile(..., writePlayConfig = true)`로 **play.cfg**에 기록됩니다.

### 3.1 프로(레벨 0) 고정값

| 파라미터 | 값 |
|----------|-----|
| chosenMoveTemperature | 0.0 |
| avoidRepeatedPatternUtility | 0.0 |
| staticScoreUtilityFactor | 0.0 |
| wideRootNoise | 0.0 |
| cpuctExploration | 1.0 |
| cpuctExplorationLog | 0.45 |
| cpuctExplorationBase | 500.0 |

### 3.2 레벨 1~30 (앵커 보간)

앵커 구간은 다음과 같이 선형 보간됩니다.

| 파라미터 | 앵커 (index → value) | 적용 범위 |
|----------|----------------------|-----------|
| chosenMoveTemperature | 1→0.20, 10→1.10, 30→2.20 | 0.05~2.2 |
| avoidRepeatedPatternUtility | 1→0.20, 10→1.20, 30→1.96 | 0~2.0 |
| staticScoreUtilityFactor | 1→0.20, 10→0.60, 30→0.85 | 0~1.0 |
| wideRootNoise | 1→0.03, 10→0.12, 30→0.20 | 0~0.2 |
| cpuctExploration | 1→0.80, 10→4.50, 30→10.00 | 0~10 |
| cpuctExplorationLog | 1→0.80, 10→4.50, 30→10.00 | 0~10 |
| cpuctExplorationBase | 1→200, 10→1200, 30→4000 | 10~100000 |

- avoidRepeatedPattern = true 고정.

### 3.3 공통(모든 레벨)

- **maxTime** = 30.0 (play.cfg에 고정)
- **maxVisits** = 100000
- winLossUtilityFactor = 1.0, dynamicScoreUtilityFactor = 0.3

### 3.4 실제 분석 예산(착수당)

- 앱이 요청할 때 사용하는 시간: `SettingsRepository.getMaxTime()` (초) → 3~6초로 클램프되어 **playBudgetMs**로 전달.
- `startGame()`에서 레벨별로 `setMaxTime(maxTimeSec)` 호출:
  - 프로(0): 3~6초
  - 초고수6(6): 2~3초
  - 고수1(7): 1.6~2초
  - 최하(30): 1~1초

즉, **착수당 분석 시간**은 레벨에 따라 위 구간의 **max** 값이 저장되고, 그 값이 3~6초로 제한되어 엔진 요청에 사용됩니다.

---

## 4. 착수 선택 로직 요약

- **프로(0)**: 항상 최고 승률 후보 1개 선택.
- **초고수1~6(1~6)**: `pickRapidStrength` — 승률 마진 내 후보 중에서 기력에 따라 선택.
- **고수1~최하 근접(7~29)**: `pickGradualStrength` — 완만한 승률 상승 곡선.
- **최저기력(30)**: `pickLowestWinrateExcludingSuicide` — 자살수 제외 후 **최저 승률** 후보 중 무작위.

---

## 5. 실제 착수한 수의 승률

- **봇 착수 시**: `selectBotMove()` 안에서 선택된 수의 `winrate`, `visits`가 **로그**로만 출력됩니다.  
  - 태그: `PlayMode`  
  - 예: `selectBotMove: level=... tier=... pick winrate=0.52 visits=1200`
- **사용자(사람) 착수**: 해당 수가 이전 국면 분석의 후보였을 때 그 수의 승률을 따로 표시하거나 로그하는 코드는 **없음**.
- **화면 표시**: 현재 국면의 **topSuggestions**(다음 수 후보) 승률만 표시됩니다. “방금 둔 수의 승률”을 보여주는 UI/로깅은 없습니다.

요약하면, **실제로 착수한 후보수의 승률**은  
- 봇이 둔 수: 로그에서만 확인 가능  
- 사람이 둔 수: 앱에서는 확인 불가 (추가하려면 분석 결과에서 해당 좌표의 승률을 찾아 표시/로깅하는 기능이 필요).

---

## 6. 로그로 확인하는 방법

1. **엔진·모델**: `Engine session=PLAY ... model=` 로그 라인.
2. **play.cfg 기록 시 파라미터**:  
   `play.cfg written: chosenMoveTemp=... wideRootNoise=... cpuct=... cpuctLog=... cpuctBase=... avoidRepeatUtility=... staticScoreUtility=... maxVisits=...`
3. **레벨·시간·기력 관련**:  
   `startGame: levelIndex=... levelLabel=... timeRange=... randTemp=... avoidUtility=... scoreUtility=...`
4. **봇이 선택한 수의 승률/방문수**:  
   `selectBotMove: level=... strength01=... strengthZ=... tier=... pick winrate=... visits=...`
5. **내부 강도 지수·레벨–Elo 보정**:  
   레벨별 strength01(0~1), strengthZ(-2~2), 대략 Elo 추정표는 `docs/AI_STRENGTH_ADJUSTMENT_RESEARCH_VS_APP.md` §6 참고.

필터 예: `adb logcat -s PlayMode:* Timber:*` 또는 앱 패키지명으로 필터.
