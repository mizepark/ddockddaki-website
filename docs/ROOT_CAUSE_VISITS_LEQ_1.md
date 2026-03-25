# 방문수 1 이하가 남발되는 근본 원인

**목표**: 추천에 쓰는 자식 노드는 `numVisits > 1` 이어야 함. 그런데 자주 totalVisits=2, 8 등으로 "zero suggestions"가 나오는 이유를 정리함.

---

## 1. 중간 콜백(인터미디어트)이 초반에만 나감

- **네이티브**: `analyzeTop4` 내부 모니터 스레드가 **100ms마다** `extractAnalysisData`를 호출해 `onIntermediateResult`로 Java에 전달함.
- **초반 0.3~1초**: 루트 totalVisits가 2, 8, 20 … 수준이라, 자식 노드 방문이 거의 안 쌓여 **전부 numVisits ≤ 1** → `extractAnalysisData`에서 추천 0개 → **"zero suggestions"** 로그만 반복.
- 즉, **방문이 2 이상 쌓이기 전 상태**가 100ms 간격으로 계속 전달되는 것이 "방문수 1 이하가 남발"처럼 보이는 한 축임.

---

## 2. 앱 쪽 중간 결과 처리

- **EngineViewModel**: `onIntermediateResult` → `handleAnalysisResult(..., isIntermediate = true)`.
- 중간 결과에서 **suggestions가 비어 있으면** `suggestionsToShow = state.analysis.topSuggestions` 로 **기존 추천을 유지**하므로, 중간이 빈 걸로 **덮어쓰진 않음**.
- 대신 **처음부터** 중간만 오면 topSuggestions는 계속 비어 있고, **최종 결과**가 와야만 추천이 채워짐.

---

## 3. 최종 결과가 적용되지 않는 경우 (가능 원인)

- **AnalysisResultGate.shouldProcessResult** 에서 `runEpoch`, `lastSubmittedEpoch`, `resultAnalyzedRequestId` 등으로 결과를 걸러냄.
- 분석을 **시작한 직후**에 `RESTORE_SAVED_GAME`, `INIT_ENGINE` 등으로 **의도/epoch가 바뀌면**, 나중에 도착하는 **최종 결과(133 visits)** 의 runEpoch가 "이전 세대"로 취급돼 **폐기**될 수 있음.
- 그러면 사용자 입장에서는 "한 번도 추천이 안 뜬다"가 됨. (실제로는 최종이 왔지만 게이트에서 버려진 경우.)

---

## 4. 분석이 일찍 끊겨서 방문이 안 쌓이는 경우

- **stopAnalysis** 가 호출되면 `g_stopFlag`로 검색이 중단됨.
- 세션 전환, 메모리 정리(trim), onCleared, Binder 끊김 등에서 stop이 불리면, 2~3초 채우기 전에 끊겨서 **totalVisits가 2, 8 수준에서 멈춤** → 계속 zero suggestions.
- **Play 엔진(engine_play)** 이 한 번 shutdown 되면, 그 프로세스로 가는 분석은 엔진이 비어 있어서 제대로 안 돌거나, 아주 짧게만 돌 수 있음.

---

## 5. 요약: "방문수 1 이하가 남발"되는 이유

| 원인 | 설명 |
|------|------|
| **중간 콜백** | 100ms마다 초반 상태(totalVisits=2,8,…)를 보내서, 자식이 전부 ≤1인 구간이 반복적으로 전달됨. |
| **최종 결과 폐기** | 의도/epoch 변경으로 최종(133 visits) 결과가 게이트에서 걸러지면, 추천은 한 번도 안 뜸. |
| **분석 조기 중단** | stopAnalysis·엔진 shutdown 등으로 검색이 빨리 끊기면, 방문이 2 이상 쌓이기 전에 끝남. |
| **헬스체크/짧은 실행** | 0.5초 등 짧은 예산으로 돌리는 경로는 애초에 2방문 수준에서 끝나서 zero suggestions만 나옴. |

---

## 6. 권장 대응 (추가 제약 없이)

1. **네이티브**: 시간 제한만 쓸 때(jMaxVisits==0) **maxPlayouts가 1로 남지 않도록** 최소 100만 보장 (설정된 파라미터대로 검색되게).
2. **앱**: 분석 시작 직후 **불필요한 RESTORE_SAVED_GAME / INIT_ENGINE 제출**이 최종 결과를 게이트에서 폐기하지 않도록 검토.
3. **stopAnalysis / 엔진 shutdown**: 불필요하게 자주 호출되지 않도록 호출 조건 정리.

로직은 설정된 파라미터에 의해 방문된 후보들 중에서만 착수하게 하고, **방문수 1짜리만 제거**할 뿐 그 외 제약은 두지 않음.
