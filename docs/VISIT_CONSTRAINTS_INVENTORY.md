# 앱 내 방문수(visits) 제약 사항 인벤토리

앱 전역에서 **방문수(visits)와 관련된 제한·필터·상한/하한**을 조사한 목록.  
(방문수 1짜리 제한은 제거된 상태 기준.)

---

## 1. Kotlin — 분석·추천

| 위치 | 내용 | 역할 |
|------|------|------|
| **AnalysisUseCase.kt** | `maxVisits: Int = 0` (analyze 인자) | 0이면 시간 제한만 사용, 네이티브에 그대로 전달. |
| **AnalysisUseCase.kt** | `minVisits = (maxVisitsValue * minShare).toInt()` | 추천 후보 필터: `s.visits >= minVisits` 만 통과. minShare는 레벨/구간별. **coerceAtLeast(2) 제거됨** → minVisits 는 0 이상. |
| **AnalysisUseCase.kt** | `maxVisitsValue in 1..499` 시 minShare 보정 | 방문수 구간별 minShare 계산. |
| **EngineViewModel.kt** | `maxVisits = 0` (runAnalysisInternal) | 분석 호출 시 항상 0 (시간 제한만). |
| **EngineViewModel.kt** | `result.totalVisits >= 1000` 등 | 중간 결과 표시/목표 도달 판단에 사용. |
| **EngineViewModel.kt** | `visits >= 1000` 시 "분석 완료" 문구 | UI 메시지 분기. |
| **EngineTypes.kt** | `visitsMin: Int = 16`, `visitsMax: Int = 2000` | 엔진 튜닝/상태용 기본값. |

---

## 2. Kotlin — 대국(봇)

| 위치 | 내용 | 역할 |
|------|------|------|
| **PlayViewModel.kt** | `BOT_MIN_SUGGESTION_VISITS = 1` | 봇 후보 필터: `it.visits >= BOT_MIN_SUGGESTION_VISITS` (최소 1방문만 있으면 사용). |
| **PlayViewModel.kt** | `BOT_RECOVERY_MIN_VISITS = 5` | 복구 시 최소 방문수 기준 (관련 로직에서 사용). |
| **PlayViewModel.kt** | `(1f / (s.visits + 1f)).pow(visitPower)` | 기력별 가중치 계산에 방문수 사용. |

---

## 3. Kotlin — 설정·저장·UI

| 위치 | 내용 | 역할 |
|------|------|------|
| **SettingsRepository.kt** | `getMaxVisits(): coerceIn(1, 3500)` | 저장된 max_visits_dynamic 범위 제한. |
| **SettingsRepository.kt** | `setMaxVisits(visits): coerceIn(1, 3500)` | 동일. |
| **SettingsRepository.kt** | `engine_visits_base`, `engine_visits_max` (기본 128, 512) | 엔진 방문수 설정. |
| **VisitsBudgeter.kt** | `visitsMin.coerceAtLeast(16)` | computeAdaptiveVisits 내부: 최소 16. |
| **VisitsBudgeter.kt** | CPU 시 `maxVisitsCap` 500/200 | CPU 모드일 때 방문 수 상한. |
| **Board.kt** | `rankedSuggestions` 의 minVisits/maxVisits | 추천별 상대 점수(60~99) 계산용. |
| **PlayBoardSection.kt** | 방문수 1000 이상일 때 참고도 팝업 | "준비 중"과 겹치지 않도록 조건. |
| **ScoreUtils.kt** | `normalizedVisits(visits, maxVisits)` | 방문수 정규화 (0~1). |

---

## 4. 네이티브 (katago_jni.cpp)

| 위치 | 내용 | 역할 |
|------|------|------|
| **analyzeTop4** | `jMaxVisits` 1~2 이고 `requestId != -999` → 빈 배열 반환 | 헬스체크가 아닌 분석 호출에 maxVisits 1~2 들어오면 빈 결과 (잘못된 경로 차단). |
| **analyzeTop4** | `jMaxVisits == 0` → `localParams.maxVisits = 1000000` | 시간 제한만 쓸 때 방문 상한 완화. |
| **analyzeTop4** | `localParams.maxPlayouts = max(localParams.maxVisits, g_globalParams.maxPlayouts)` | 검색이 1에서 끊기지 않도록 상한 맞춤. |
| **extractAnalysisData** | **제거됨**: `numVisits <= 1` 필터 없음 | 모든 자식 후보를 결과 배열에 포함. |
| **initEngine 워밍업** | `wParams.maxVisits = 1` (초기) → 5 (의미 있는 워밍업) | 초기화 시 짧은 검색. |
| **stoppingParams** | `maxVisits = 1`, `maxTime = 0.0001` | 중단용 파라미터. |
| **nativeGetPolicyHeatmap** | `localParams.maxVisits = 1`, `maxPlayouts = 1` | 정책만 빠르게. |
| **getRootLead** | `localParams.maxVisits = 128`, `maxPlayouts = 128` | 리드 추정용. |
| **getOwnershipHeatmap** 등 | `maxVisits = 128` | 소유권/히트맵용. |

---

## 5. 설정·에셋

| 위치 | 내용 | 역할 |
|------|------|------|
| **assets/katago/analysis.cfg** | `maxVisits = 1600` | 분석용 기본 상한 (config 로드). |
| **AssetInstaller.kt** | play.cfg 에 `maxVisits = 100000` 등 | 대국용 설정 작성. |
| **SettingsRepository.kt** | config 파싱 시 `maxVisits` 정규식 | 기기 설정에서 visits 추출. |

---

## 6. 제거된 항목 (이번 수정)

- **AnalysisUseCase.kt**: 파싱 시 `visits <= 1` 이면 추천에서 제외하던 로직 **제거**.
- **AnalysisUseCase.kt**: `minVisits` 계산 시 `.coerceAtLeast(2)` **제거**.
- **katago_jni.cpp extractAnalysisData**: `data[i].numVisits <= 1` 이면 결과에 넣지 않던 필터 **제거** (후보 전부 포함).

---

이 문서는 조사 시점 기준 인벤토리이며, 방문수 1짜리 제한 제거 반영됨.
