# 대국모드 기력 조절 진단 요약

현재 앱의 기력 조절이 **제대로 되어 있는지** 코드·설계 기준으로 정리한 문서입니다.

---

## 1. 결론 요약

| 질문 | 답변 |
|------|------|
| **기력 조절이 되고 있나?** | **예.** 레벨에 따라 play.cfg 파라미터와 후처리 선택 로직이 구간별로 다르게 적용된다. |
| **모든 레벨이 의도대로 구분되나?** | **대체로 예.** 다만 후보 수가 적거나 분석 시간이 짧을 때는 레벨 차이가 잘 안 날 수 있다. |
| **버그나 잘못된 경로가 있나?** | 설계상 명확한 버그는 없음. 다만 **설정 적용 순서**와 **후보 수가 적을 때의 수렴**은 개선 여지가 있다. |

---

## 2. 잘 되어 있는 부분

- **프로(0)**: 항상 최고 승률 1개 선택. `play.cfg`도 온도·노이즈 0으로 고정.
- **최저기력(30)**: 자살수 제외 후 최하승률 5% 풀에서 무작위. 로그에서도 `pick=lowest`, 낮은 winrate로 확인 가능.
- **레벨 1~6 (초고수)**: `pickRapidStrength` — best 확률(레벨 1에서 높음 ~ 6에서 낮음) + 승률 마진 내에서만 선택. 레벨이 올라갈수록 best에 수렴.
- **레벨 7~29 (고수~초보)**: `pickGradualStrength` — 하위 비율(tail) 풀 + 가중 샘플링. 레벨 숫자가 작을수록(7에 가까울수록) 고승률 쪽으로 선택.
- **play.cfg**: `startGame()`에서 레벨별로 `chosenMoveTemperature`, `wideRootNoise`, `cpuct*`, `avoidRepeatedPatternUtility`, `staticScoreUtilityFactor`를 앵커 보간해 설정하고, `ensureConfigFile(..., writePlayConfig = true)`로 기록.
- **착수당 시간**: 레벨별 `setMaxTime()` → `getMaxTime()` → `playBudgetMs`로 분석 엔진에 전달됨.
- **합법·자살 필터**: `selectBotMove`에서 `isLegalMove` 및 `MIN_WINRATE_NON_SUICIDE`로 불법/자살수 제거 후 선택.
- **모델 분리**: 레벨 0~6 → B10, 7~30 → B6. `ensurePlayEngineReadyForLevel(levelIndex)`에서 세션·모델 준비.

---

## 3. 한계·주의점 (기력이 덜 구분되거나 불안정해 보일 수 있는 경우)

### 3.1 후보 수가 적을 때

- **원인**: `pickGradualStrength`, `pickRapidStrength` 모두 **후보 리스트(suggestions)** 크기에 의존한다.
- **동작**:
  - `suggestions.size`가 1~2개면 `lowPool`/`candidates`가 best 하나만 포함하거나, `pool`이 비어서 **결과적으로 best를 그대로 반환**한다.
  - 분석 시간이 짧은 레벨(예: 최하 1초)이나 초반 국면에서는 엔진이 적은 수만 후보로 돌려줘서, **레벨 7~29·1~6이라도 best에 가깝게 수렴**할 수 있다.
- **정리**: “레벨을 내렸는데도 생각보다 강하게 둔다”고 느껴지면, 그 국면에서는 후보가 적어서 그럴 수 있다.

### 3.2 방문수가 매우 적은 후보

- **원인**: `BOT_MIN_SUGGESTION_VISITS = 1`이라 방문수 1~2인 후보도 풀에 들어가고, 방문수 품질 필터(예: 최대 방문수의 10% 미만 제외)는 없다.
- **동작**: 승률 추정이 노이즈가 큰 후보도 선택 대상이 되어, **같은 레벨이라도 국면·타이밍에 따라 선택이 들쭉날쭉**할 수 있다.
- **참고**: `docs/LOW_VISIT_COUNT_ANALYSIS.md`.

### 3.3 play.cfg 기록 시점

- **현재 순서**: `startGame()` 내부에서  
  1) `ensurePlayEngineReadyForLevel(selectedLevelIndex)`  
  2) `ensureConfigFile(application, writePlayConfig = true)`  
  즉, **엔진을 먼저 준비한 뒤** play.cfg를 쓴다.
- **영향**: PLAY 엔진이 이미 기동 중이면, 그때까지는 **이전 게임의 play.cfg**를 읽고 있을 수 있다. 레벨을 바꾼 직후 **첫 수만** 예전 레벨 설정으로 나갈 가능성이 있다.
- **개선안**: `ensureConfigFile(writePlayConfig = true)`를 `ensurePlayEngineReadyForLevel` **앞**에 두거나, 설정 기록 후 PLAY 엔진을 한 번 재초기화하는 방식 검토.

### 3.4 레벨 간 체감·Elo

- 레벨–Elo 실측이 없어, “레벨 14 ≈ 몇 Elo” 같은 보장은 없다.  
- 대략 추정표는 `docs/AI_STRENGTH_ADJUSTMENT_RESEARCH_VS_APP.md` §6 참고.

---

## 4. 점검 방법 (실기로 확인할 때)

1. **로그**
   - `adb logcat -s PlayMode:*` 등으로  
     `startGame: levelIndex=... strength01=... strengthZ=... timeRange=...`  
     `selectBotMove: level=... strength01=... strengthZ=... tier=... pick winrate=... visits=...`  
     확인.
2. **레벨 0 vs 30**
   - 프로: `pick=best`, winrate가 보통 높게 나옴.  
   - 최저기력: `tier=최저기력`, `pick=lowest`, winrate가 상대적으로 낮게 나옴.
3. **레벨 변경 후 첫 수**
   - 레벨을 바꾼 뒤 바로 새 게임 시작하고, 첫 봇 수의 로그에서 `levelIndex`와 `play.cfg written`(이전에 찍힌) 값이 일치하는지 확인.

---

## 5. 요약 한 줄

- **기력 조절 자체는 설계대로 적용되어 있고**, 프로/최저기력 구분과 레벨별 선택 로직·파라미터는 의도대로 동작한다.
- **후보가 적거나 시간이 짧은 국면**에서는 레벨 차이가 작게 느껴질 수 있고, **레벨 변경 직후 한 수**는 예전 설정일 수 있으니, 위 한계를 알고 있으면 “제대로 되어 있는지” 판단에 도움이 된다.
