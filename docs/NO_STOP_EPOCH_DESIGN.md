# 중단 제거 + Epoch 기반 결과 무효화 설계

**목적**: “중복 간섭/중단” 구조를 제거하고, 세대(epoch) 승계로만 결과를 제어한다.

---

## 1. 문제 요약

- **STOP_BEFORE**: 새 의도마다 기존 분석을 먼저 중단 → 방문수 0/빈 결과 가능성 증가.
- **명시적 stopAnalysis**: 변형 종료, 오버레이, 착수/Undo/Jump에서 호출 → 분석이 충분히 진행되기 전에 끊김.
- **결과**: “방문 0”과 “중단/레이스”가 구조적으로 연결됨.

---

## 2. 구조적 대안 (적용 방향)

| 항목 | 내용 |
|------|------|
| **단일 분석 루프 + 단일 입력 스트림** | UI는 “새 스냅샷(의도)”만 전달. 기존 분석은 **중단하지 않고**, 결과만 epoch로 무효화. |
| **Epoch 승계** | 각 의도에 `epoch` 부여. 결과 수신 시 **마지막 epoch만 반영**, 이전 결과는 게이트에서 폐기. |
| **StopPolicy 제거** | STOP_BEFORE 제거, 항상 NO_STOP. “의도 갱신 = stop” 경로 제거. |
| **ViewModel에서 stopAnalysis 제거** | 착수/Undo/점프/변형/오버레이에서는 **stopAnalysis 호출 없이** 새 의도만 발행. |
| **엔진 측** | 필요 시 Facade에서 “새 분석 시작 시에만” 내부적으로 이전 정리(soft). ViewModel은 stop 노출하지 않음. |

---

## 3. 구현 요약

- **AnalysisIntent**: `epoch: Long` 추가. `stopPolicy` 제거(또는 항상 NO_STOP).
- **ViewModel**: `currentEpoch` 증가 → 의도 제출 시 `intent.epoch = currentEpoch`. `runAnalysisInternal`에서 `stopBefore` 분기 제거, `lastRunEpoch` 저장.
- **결과 게이트**: `shouldProcessAnalysisResult`에 `runEpoch`, `lastSubmittedEpoch` 추가 → `runEpoch < lastSubmittedEpoch`이면 반영 안 함.
- **stopAnalysis 제거**: playMove/stopVariation/오버레이/Undo/Jump 등에서 `stopAnalysis` 호출 제거; 새 의도만 발행.

---

## 4. 참조

- AnalysisSnapshotValidator, AnalysisStateMachine, EngineViewModel runAnalysisInternal / handleAnalysisResult.

---

## 5. 구현 완료 요약

| 항목 | 적용 내용 |
|------|-----------|
| **AnalysisIntent.epoch** | 추가. 제출 시 currentEpoch 증가, 동일 위치 스킵 시에는 증가 안 함. |
| **StopPolicy 기본값** | STOP_BEFORE → NO_STOP. runAnalysisInternal에서 stopBefore 분기 제거. |
| **결과 게이트** | shouldProcessAnalysisResult에 runEpoch, lastSubmittedEpoch 추가. runEpoch < lastSubmittedEpoch이면 폐기. |
| **ViewModel stopAnalysis 제거** | playMove, stopVariation, 소유권 오버레이, jumpToStart/End, jumpBack5, jumpForward5, redo, loadSgf, setTargetLatency, triggerHintAnalysis에서 제거. |
| **HINT/DEEP 목표 도달** | 목표 도달 시 stopAllAnalysis 호출 제거. 결과 채택 조건(shouldShowSuggestions)으로만 처리. |
| **유지** | onCleared 시 stopAllAnalysis()만 호출. 세션 전환/재시작/종료/복구는 EngineOrchestrator 등 기존 경로 유지. |
