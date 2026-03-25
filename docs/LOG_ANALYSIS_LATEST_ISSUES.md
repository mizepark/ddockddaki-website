# 최신 로그 분석 — 추천수/착수 안 되는 원인

캡처 로그(`device_log_latest.txt`) 기준으로 정리한 내용입니다.

---

## 1. 착수 불가: isLegal_failed 반복

- **로그**: `EngineViewModel: isLegal_failed` 가 16:08 구간에 **수십 번** 연속으로 찍힘.
- **위치**: `ensureMoveIsLegal` → `engineFacade.isLegal` → 네이티브/서비스 호출에서 **예외** 발생 시 `getOrElse { false }` 로 false 반환.
- **의미**: 착수 시 합법성 검사를 엔진(다른 프로세스)에 요청하는데, **IPC/타임아웃/서비스 비정상** 등으로 호출이 실패하고 있어, 모든 착수가 "해당 지점에는 둘 수 없습니다"로 이어짐.

---

## 2. 추천수 안 뜸: extractAnalysisData zero suggestions

- **로그**: `extractAnalysisData: zero suggestions after filter n=64 totalVisits=... maxVisits=1 <=1=64` 반복.
- **의미**: 분석은 끝나지만(Search finished visits=134 등) **자식 노드가 모두 numVisits ≤ 1** 이라, `numVisits > 1` 필터를 통과하는 추천이 0개 → 화면에 추천수 없음.
- **과거 빌드**: `maxVisits=128` 로 제한돼 있어 128번 플레이아웃이 많은 수에 나뉘어 자식당 1 이하로만 쌓임.
- **최신 빌드(17:34~)**: `triggerAnalysis start mode=QUICK budget=... (time-only)`, `analyze start ... maxVisits=0` 로 **시간만 사용**하는 빌드가 동작 중.

---

## 3. 최신 빌드(17:34) 동작 요약

- **17:34:40**  
  - 분석 엔진(프로세스 18524) init 성공 → Warmup, Main Search initialized.  
  - 곧바로 `analyzeTop4: Search finished visits=2` (워밍업 수준), `extractAnalysisData: zero suggestions` (totalVisits=2라 당연히 0개).  
  - `EngineFacade: health check passed`, `session=ANALYSIS model=b10.bin ready`.
- **17:34:40.252**  
  - 대국 엔진 프로세스 시작: `Start proc 18654:com.katago.android:engine_play`.
- **17:34:41~17:35:50**  
  - 앱(18304)에서 **triggerAnalysis (time-only)** 가 **1~2초 간격으로 매우 빈번**하게 호출됨.  
  - `analyze start budgetMs=1826 maxVisits=0` 등으로 분석 요청은 계속 나감.
- **17:34:57**  
  - `NativeBridge_shutdown: Engine state cleared` (프로세스 **18654** = engine_play).  
  - **대국 쪽 네이티브 엔진만** 리셋된 상태.

정리하면:

- **분석 쪽(18524)** 은 17:34:40에 정상 초기화되었고, 이후 로그에선 이 프로세스가 shutdown 되었다는 기록은 없음.
- **대국 쪽(18654)** 은 시작 직후 약 17초 만에 shutdown 되어, 이후 이 프로세스로의 분석/legal 요청은 **엔진이 비어 있는 상태**로 가거나 실패할 수 있음.
- 그와 별개로 **분석 요청이 1~2초마다 반복**되면, 한 번 돌아가던 분석이 다음 요청/상태 전이에 의해 **중단되거나 결과가 쓰이기 전에 덮일** 가능성이 있음.

---

## 4. 추정 원인 정리

| 증상 | 로그상 원인 |
|------|-------------|
| **착수 안 됨** | `isLegal_failed` 다수 → 엔진 쪽 `isLegal` 호출이 **예외/IPC 실패**로 끝나 false 반환. (재시도 추가한 최신 빌드에서도, 기기 로그가 이전 빌드일 수 있음.) |
| **추천수 안 뜸** | (1) **zero suggestions after filter**: 루트 방문은 쌓이지만 자식별로 1 이하만 쌓여 추천 0개. (2) **분석 요청 과다**: triggerAnalysis가 1~2초마다 반복되면 분석이 끝나기 전에 취소되거나, 결과가 반영되기 전에 다음 요청으로 덮일 수 있음. |
| **대국 모드까지 동일** | 대국 엔진(18654)이 **시작 후 약 17초 만에 shutdown** 되어, 대국 탭에서의 분석/legal이 **엔진 없는 프로세스**를 향해 가거나 실패하는 상태. |

---

## 5. 제안 사항

1. **isLegal 실패**  
   - 최신 빌드(재시도 + time-only)가 **실제로 이 기기에 설치**돼 있는지 확인.  
   - 설치가 최신이면: **엔진 서비스가 살아 있는지**, **Binder 타임아웃/재시도** 로그를 추가해, 실패가 “엔진 미초기화”인지 “IPC/타임아웃”인지 구분하는 것이 좋음.

2. **분석 요청 과다**  
   - `triggerAnalysis` / `intent_update` 가 **같은 국면에서 1~2초마다** 나가지 않도록, **보호 구간·디바운스** 등으로 요청 빈도를 줄이면, 한 번 돌린 분석이 끝까지 돌고 결과가 UI에 반영될 가능성이 높아짐.

3. **engine_play shutdown**  
   - **17:34:57** 에 **18654(engine_play)** 만 `NativeBridge_shutdown` 이 호출된 이유(세션 전환, 설정 변경, 메모리 정리 등)를 코드/로그로 확인.  
   - 불필요하게 PLAY 엔진만 죽는 경로가 있다면, 그 호출을 줄이거나 조건을 완화하는 것이 좋음.

4. **추천 0개 필터**  
   - 단기 대응으로, **numVisits > 1** 대신 **numVisits >= 1** (또는 0 제외) 등으로 완화해, 최소 1번이라도 방문된 수는 보이게 할 수 있음. (정책에 따라 선택.)

이 문서는 **원인 분석과 대응 방향**만 정리한 것이며, 실제 수정은 별도 작업이 필요합니다.
