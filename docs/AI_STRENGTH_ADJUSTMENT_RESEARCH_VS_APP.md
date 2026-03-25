# 인공지능 기력조절: 외부 연구 vs 본 앱 비교

## 1. 개요

바둑/Go AI의 **기력 조절(Strength Adjustment)** 은 같은 엔진으로 초보부터 프로 수준까지 난이도를 나누기 위한 기법이다.  
이 문서는 **외부 연구·문서**와 **본 앱(대국모드 봇)** 의 방식을 정리하고 비교한다.

---

## 2. 외부 연구·문서 요약

### 2.1 MCTS 기력조절 논문 (AAAI 2019)

**출처:** Wu et al., "On Strength Adjustment for MCTS-Based Programs", AAAI 2019.  
**대상:** ELF OpenGo (AlphaZero 스타일 Go 엔진).

**핵심 방법:**

1. **방문수(Simulation Count) 필터링**
   - 각 수의 **시뮬레이션(방문) 수**가 **최대 방문수의 일정 비율(threshold ratio)** 미만이면 **제외**.
   - 예: threshold ratio 0.1 → 최대 방문수의 10% 미만인 수는 후보에서 제거.
   - 이렇게 하면 **“일정 수준 이상의 품질”** 을 보장하는 수만 남는다는 이론적 분석 제시.

2. **강도 지수 z와 Softmax**
   - 필터링된 후보들에 대해 **strength index z** 를 쓴 **softmax 정책**으로 수 선택.
   - z가 클수록 더 강한 수에 가깝게, 작을수록 더 다양한(약한) 수가 선택될 수 있게 조절.

**실험 결과:**

- z와 **Elo 레이팅이 선형 관계** (z ∈ [-2, 2] 구간에서 약 **800 Elo** 범위).
- threshold ratio 0.1 기준 **회귀 오차 약 47.95 Elo**.
- 논문에서는 “조절 가능한 기력–지수 관계”를 유지하면서 넓은 기력 범위를 다루는 **state-of-the-art** 로 소개.

**특징:**

- 기력 조절이 **MCTS 결과(방문수)** 에 기반.
- **단일 지수 z** 로 기력 구간을 선형적으로 매핑.
- **품질 하한** 을 threshold ratio로 보장.

---

### 2.2 AlphaGo / AlphaZero 쪽 온도(Temperature)

**출처:** AlphaGo Zero / AlphaZero 논문, 커뮤니티 구현(Leela Zero, ELF OpenGo 등).

**역할:**

- **Temperature τ**: MCTS **방문 분포**를 \( N^{\tau} \) 형태로 변환해 수를 선택.
  - τ = 0: 항상 최다 방문 수 선택 (가장 강함).
  - τ > 0: 방문 수가 적은 수도 확률적으로 선택 → **탐험 증가, 실수/약한 수 허용**.

**구현 차이:**

- AlphaGo Zero: move 30 이후 τ → 0.
- AlphaZero: 논문에 따라 τ 유지/감소 방식이 제각각.
- ELF OpenGo: 전체 국면 τ=1 사용 등.

**정리:**  
온도는 **“방문 분포를 얼마나 날카롭게 할지”** 를 정하는 파라미터로, **높을수록 약한/다양한 수**가 나올 수 있다.

---

### 2.3 KataGo 문서 (분석 엔진 / GTP)

**출처:** KataGo 저장소 `Analysis_Engine.md`, `GTP_Extensions.md`, 이슈 등.

**관련 파라미터·개념:**

| 항목 | 설명 |
|------|------|
| **maxVisits** | 한 국면 분석 시 최대 방문 수. 많을수록 정확하지만 시간 증가. |
| **chosenMoveTemperature** | 착수 선택 시 적용되는 온도. 높을수록 방문 분포가 완만해져 다양한/약한 수 선택 가능. |
| **chosenMoveTemperatureEarly** | 초반 전용 온도 (문서에 언급). |
| **wideRootNoise** | 루트에서의 탐색 노이즈. 다양성·불확실성 증가. |
| **cpuctExploration / cpuctExplorationLog / cpuctExplorationBase** | PUCT 탐색 강도. 높을수록 다양한 수 탐색. |
| **avoidRepeatedPatternUtility / staticScoreUtilityFactor** | 반복 패턴 회피·점수 유틸리티. 인간 스타일/강도에 영향. |

**Human-style / 약한 플레이 관련:**

- **Human-style 모델**(예: g170) + **staticScoreUtilityFactor**, **chosenMoveTemperature** 조절로 “인간다운”/다양한 수 유도.
- **chosenMoveTemperatureOnlyBelowProb = 1.0** 으로 두고 **chosenMoveTemperature** 를 낮추면, 더 결정론적·강한 플레이에 가깝게 조절 가능.
- 핸디캡(5수 이상): 고핸디캡 훈련 데이터 부족 등으로 별도 개선 논의 있음(Issue #39 등).

**정리:**  
KataGo는 **설정 파일(play.cfg 등)** 로 온도·노이즈·cpuct·유틸리티 등을 바꿔 **검색 단계에서부터** 기력/스타일을 조절한다.

---

### 2.4 요약: 외부에서 쓰는 기력조절 축

| 축 | 방식 | 대표 사례 |
|----|------|-----------|
| **방문수 필터** | 최대 방문수의 비율 미만 수 제외 | AAAI 논문 (threshold ratio) |
| **단일 지수 + Softmax** | z와 선형 관계로 Elo/기력 매핑 | ELF OpenGo (z ∈ [-2,2]) |
| **온도** | 방문 분포를 \( N^{\tau} \) 로 스무딩 | AlphaGo/Zero, KataGo chosenMoveTemperature |
| **검색 파라미터** | cpuct, wideRootNoise, 유틸리티 등 | KataGo play.cfg |
| **시간/방문 상한** | maxTime, maxVisits 제한 | KataGo, 일반 MCTS 엔진 |

---

## 3. 본 앱의 기력조절 방식

### 3.1 이중 구조: “검색 파라미터 + 후처리 선택”

앱은 두 단계를 쓴다.

1. **검색 단계 (KataGo play.cfg)**  
   레벨에 따라 `chosenMoveTemperature`, `wideRootNoise`, `cpuct*`, `avoidRepeatedPatternUtility`, `staticScoreUtilityFactor` 등을 **앵커 보간**으로 설정해, **엔진이 이미 약한/다양한 수를 더 탐색하도록** 만든다.

2. **후처리 단계 (앱 Kotlin)**  
   엔진이 준 `topSuggestions`(후보 수 목록)에서 **승률·방문수**를 이용해 한 수를 고른다.  
   - **레벨 0 (프로):** 항상 최고 승률 1개.  
   - **레벨 1~6:** `pickRapidStrength` — best 확률 + 승률 마진 내에서 샘플링.  
   - **레벨 7~29:** `pickGradualStrength` — 하위 비율(tail) 풀에서 승률·방문수 가중 샘플링.  
   - **레벨 30:** `pickLowestWinrateExcludingSuicide` — 자살수 제외 후 **최저 승률 5% 풀** 에서 무작위.

자세한 수식·상수는 `PLAY_MODE_BOT_SPEC.md`, `PlayViewModel.kt` 참고.

### 3.2 앱에서 쓰는 “축” 정리

| 축 | 앱에서의 사용 |
|----|----------------|
| **온도** | `chosenMoveTemperature`: 레벨 1→0.2, 10→1.1, 30→2.2 등 앵커 보간. |
| **검색 다양성** | `wideRootNoise`, `cpuct*` 레벨별 증가 → 약한 레벨일수록 탐색 더 분산. |
| **스타일/유틸리티** | `avoidRepeatedPatternUtility`, `staticScoreUtilityFactor` 레벨별 증가. |
| **시간 예산** | 레벨별 `maxTime` (프로 3~6초, 최하 1초 등). |
| **후처리: 승률/방문수** | 최고승률 1개 고정(프로), 구간별 tail 비율·최소승률·가중샘플링(7~29), 최저승률 5% 풀(30). |

**방문수 필터:**  
앱은 **“최대 방문수의 threshold ratio 미만 제외”** 같은 **엔진 출력 필터**는 쓰지 않는다.  
대신 **후보 풀을 승률 기준으로 잘라** (tail 비율, 최저 5% 등) 그 안에서만 선택한다.

---

## 4. 비교

### 4.1 유사점

- **MCTS 결과(방문/승률) 활용**  
  외부 연구와 앱 모두 “검색이 만든 후보” 위에서 동작한다.
- **온도 개념**  
  KataGo `chosenMoveTemperature` 와 AlphaGo/Zero의 τ는 역할이 비슷하다. 앱은 이 값을 레벨별로 보간해 넣는다.
- **“약한 수”를 고의로 선택**  
  논문의 softmax(z), KataGo의 온도·노이즈, 앱의 pickGradual/pickLowest 모두 “최선만 고르지 않고 의도적으로 약한/다양한 수를 택함”으로 기력을 낮춘다.
- **단일 축이 아닌 다중 파라미터**  
  KataGo처럼 온도·노이즈·cpuct·유틸리티를 함께 쓰고, 앱은 여기에 **후처리(승률 구간·가중치)** 를 더해 세밀하게 구간을 나눈다.

### 4.2 차이점

| 항목 | 외부 (AAAI 등) | 본 앱 |
|------|----------------|--------|
| **방문수 threshold** | 최대 방문수의 비율 미만 수 **제외** (품질 하한 보장) | 사용 안 함. 대신 **승률 기준** tail/최저 5% 풀 사용. |
| **기력 지수** | **단일 z**, Elo와 선형 관계로 보정·예측 가능 | **레벨 0~30** 이나, z 같은 “연속 지수”는 노출하지 않음. |
| **Elo/정량 보정** | z ↔ Elo 회귀, 오차 수십 Elo 수준 보고 | 레벨–Elo 보정·보고 없음. |
| **조절 위치** | 주로 **선택 정책**(필터 + softmax) | **검색 설정(play.cfg) + 선택 로직** 둘 다 사용. |
| **최저 기력** | 필터로 “너무 나쁜 수” 차단 | **최저승률 풀 + 자살수 제외** 로 “너무 나쁜 수”만 막음. |

### 4.3 앱 방식의 장단점

**장점**

- **레벨 수가 많고 구간별로 다르게** 동작할 수 있음 (프로 / 초고수 / 고수~초보).
- **Human-style 모델(g170)** 과 **play.cfg** 조합으로 “인간다운” 실수·스타일을 검색 단계에서부터 유도.
- **후처리** 로 같은 엔진 출력이라도 레벨별로 확실히 다른 선택(최선만 / tail / 최저승률 풀)을 할 수 있음.
- 구현이 **엔진 출력(승률·방문수)** 만 있으면 되어, 엔진 내부 수정 없이 적용 가능.

**단점 / 한계**

- **방문수 품질 필터가 없음**  
  논문처럼 “최대 방문수의 10% 미만 수 제외”를 하지 않아, 방문수가 매우 적은 수도 풀에 들어갈 수 있음.  
  (실제로는 `minVisits` 등으로 어느 정도 걸러질 수 있으나, “threshold ratio”와 같은 이론적 하한은 없음.)
- **레벨–Elo 보정 없음**  
  “레벨 14 ≈ 몇 Elo” 같은 보정·실험 데이터가 없어, 난이도가 사용자 체감에만 의존.
- **시간 예산이 짧은 구간**  
  최하 레벨에서 1초 등으로 짧으면 방문수가 적어, 후보 풀 자체가 불안정할 수 있음.

---

## 5. 정리 및 제안

- **외부 연구**는 주로  
  - **방문수 기반 필터(threshold ratio)**  
  - **단일 strength index z + softmax**  
  - **Elo와의 선형 관계**  
  로 “조절 가능하고 예측 가능한” 기력을 강조한다.
- **본 앱**은  
  - **KataGo 검색 파라미터(온도·노이즈·cpuct·유틸리티)**  
  - **승률 구간·tail·최저승률 풀** 을 이용한 후처리 선택  
  으로 **다단계 레벨**과 **Human-style** 을 구현한다.

**적용한 개선 (§6):**  
- **내부 강도 지수**: 레벨 → strength01(0~1), strengthZ(-2~2) 매핑을 코드·로그에 반영.  
- **레벨–Elo 보정**: 레벨별 대략 Elo 추정표를 두고, 추후 측정으로 갱신 가능하도록 문서화.

**미적용 아이디어 (참고용):**

1. **방문수 필터 도입**  
   후보 풀을 만들 때 “최대 방문수의 θ(예: 0.1) 미만인 수는 제외”를 옵션으로 두면, 논문과 비슷한 **품질 하한**을 줄 수 있다.
2. **레벨–Elo 보정**  
   레벨별로 대국 로그/시뮬레이션으로 대략적인 Elo를 추정해 두면, “레벨 14 ≈ 1200 Elo” 같은 설명이 가능해진다.
3. **단일 “강도 지수” 노출**  
   내부적으로 레벨을 연속 지수(예: 0~1)로 매핑하고, 필요하면 “이 레벨 ≈ z=…” 형태로 문서화하면, 외부 연구와 비교·튜닝이 쉬워진다.

이 문서는 위와 같이 **외부 연구·문서**와 **앱의 현재 설계**를 나란히 두고, 같은 목표(기력 조절)에 대해 어떤 선택이 이루어졌는지 비교한 것이다.

---

## 6. 적용: 내부 강도 지수 및 레벨–Elo 보정

### 6.1 내부 강도 지수 (strength01, strengthZ)

레벨을 **연속 지수**로 매핑해 로그·튜닝·외부 연구와 비교할 수 있도록 했다.

| 지수 | 범위 | 정의 | 용도 |
|------|------|------|------|
| **strength01** | 0 ~ 1 | 1 − (levelIndex / 30). 레벨 0 → 1, 레벨 30 → 0. | 앱 내부·문서용. 1=최강, 0=최하. |
| **strengthZ** | −2 ~ 2 | 2 − (levelIndex / 30)×4. 레벨 0 → 2, 레벨 30 → −2. | ELF OpenGo 논문의 strength index z와 대응. |

**구현 위치:** `PlayViewModel.kt`  
- `playStrengthIndex01(levelIndex)`, `playStrengthIndexZ(levelIndex)` (private).  
- `startGame` 로그 및 `selectBotMove` 로그에 `strength01=…`, `strengthZ=…` 출력.

**로그 예:**
```
startGame: levelIndex=14 strength01=0.533 strengthZ=0.133 ...
selectBotMove: level=14 strength01=0.533 strengthZ=0.133 tier=고수1~ pool=... pick winrate=... visits=...
```

### 6.2 레벨–Elo 보정표

레벨별 **대략적인 Elo** 추정표. 실제 Elo는 대국/시뮬레이션 측정 후 갱신해야 한다.  
아래 표의 Elo는 **참고용 선형 보간**이며(프로 ≈ 2000, 최하 ≈ 1200, 구간 800), **미측정** 상태이다.

| 레벨 | 라벨 | strength01 | strengthZ | Approx Elo (참고·미측정) |
|------|------|------------|-----------|--------------------------|
| 0 | 프로 | 1.00 | 2.00 | ~2000 |
| 1~6 | 초고수 1~6 | 0.97~0.80 | 1.87~1.20 | ~1997~1980 |
| 7~12 | 고수 1~6 | 0.77~0.60 | 1.07~0.40 | ~1977~1960 |
| 13~18 | 중수 1~6 | 0.57~0.40 | 0.27~−0.40 | ~1957~1940 |
| 19~24 | 하수 1~6 | 0.37~0.20 | −0.53~−1.20 | ~1937~1920 |
| 25~30 | 초보 1~6 | 0.17~0.00 | −1.33~−2.00 | ~1917~1200 |

**Elo 참고 공식 (미측정 시 placeholder):**  
`Approx Elo ≈ 2000 − (levelIndex / 30) × 800` (레벨 0 → 2000, 레벨 30 → 1200).  
실제로는 기기·모델·시간 제한·대국 수에 따라 달라지므로, **실측 후 표를 갱신**하는 것을 권장한다.

---

## 7. 참고 문헌·링크

- Wu, I.-C., Wu, T.-R., Liu, A.-J., Guei, H., & Wei, T. (2019). **On Strength Adjustment for MCTS-Based Programs.** AAAI, 33(01), 1222–1229.  
  https://doi.org/10.1609/aaai.v33i01.33011222  
- KataGo: Handicap / strength 관련 이슈 및 문서  
  - e.g. [How to improve KataGo at high handicap?](https://github.com/lightvector/KataGo/issues/39)  
  - `Analysis_Engine.md`, `GTP_Extensions.md` (chosenMoveTemperature, human-style 등)
- AlphaGo Zero / AlphaZero: policy temperature, MCTS visit distribution.
- ELF OpenGo: strength index z, threshold ratio, Elo correlation.
- 본 앱: `docs/PLAY_MODE_BOT_SPEC.md`, `PlayViewModel.kt` (pickRapidStrength, pickGradualStrength, pickLowestWinrateExcludingSuicide).
