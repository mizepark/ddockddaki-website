# 대국모드 MCTS 탐색 특성 조절

MCTS(몬테카를로 트리 탐색)의 **탐색 폭·특성**을 조절할 수 있는지, 그리고 앱에서 어떻게 쓰는지 정리한 문서입니다.

---

## 1. 조절 가능 여부: **가능**

KataGo의 SearchParams에는 탐색을 넓히거나 좁히는 파라미터가 있으며, **play.cfg**에 넣은 값이 대국모드 검색에 반영되도록 수정했습니다.

---

## 2. 관련 파라미터 (play.cfg → MCTS)

| 파라미터 | 역할 | 레벨별 앱 설정 (PLAY_MODE_BOT_SPEC) |
|----------|------|-------------------------------------|
| **cpuctExploration** | PUCT 탐색 상수. **높을수록** 덜 탐색된 수도 더 자주 선택 → 방문이 여러 수에 퍼짐. | 레벨 1→0.8, 10→4.5, 30→10.0 |
| **cpuctExplorationLog** | PUCT 로그 항. 역시 높을수록 탐색 폭 증가. | 레벨 1→0.8, 10→4.5, 30→10.0 |
| **cpuctExplorationBase** | PUCT 베이스. | 레벨 1→200, 10→1200, 30→4000 |
| **wideRootNoise** | 루트에 더하는 노이즈. **높을수록** 다양한 수가 탐색됨. | 레벨 1→0.03, 10→0.12, 30→0.20 |
| **rootNoiseEnabled** | 루트 노이즈 on/off (play.cfg에서 설정 시). | (앱은 wideRootNoise만 레벨별 설정) |
| **fpuReductionMax** / **rootFpuReductionMax** | FPU(First Play Urgency). 미방문 수에 주는 보정. | config 기본값 사용 |

- **프로(레벨 0)**: cpuct·wideRootNoise 등은 0 또는 낮은 고정값으로, “최선만 골라서” 강하게 둠.
- **레벨이 낮을수록(30에 가까울수록)**: cpuct·wideRootNoise를 크게 해서 **탐색을 넓히고**, 그 결과 **방문수 2 이상인 후보가 더 많이** 나오도록 설계됨.

---

## 3. 수정 내용: analyzeTop4에서 config 존중

**문제**:  
`katago_jni.cpp`의 `analyzeTop4()`에서 예전에 **cpuctExploration=1.0**, **cpuctExplorationLog=0.45**, **fpuReductionMax=0.2**, **rootFpuReductionMax=0.1**을 **고정으로 덮어쓰고** 있어서, play.cfg에 넣은 레벨별 cpuct·wideRootNoise가 **검색에 전혀 반영되지 않았음**.

**수정**:  
위 탐색 관련 덮어쓰기를 제거하고, **g_globalParams**(엔진 초기화 시 play.cfg에서 읽은 값)를 그대로 쓰도록 변경.

- **유지한 것**: `maxTime`, `maxVisits`, `maxPlayouts`, `lagBuffer`, `useLcbForSelection`, `lcbStdevs`, `numVirtualLossesPerThread` (시간 제어·안정성용).
- **제거한 덮어쓰기**: `cpuctExploration`, `cpuctExplorationLog`, `fpuReductionMax`, `rootFpuReductionMax` → config(play.cfg) 값 사용.

그래서 **대국모드에서는 play.cfg의 MCTS 파라미터로 탐색 특성을 조절할 수 있습니다.**

---

## 4. 기대 효과

- **레벨이 낮을수록** cpuct·wideRootNoise가 커지므로, 같은 시간에도 **방문이 여러 수에 더 퍼져** “방문수 > 1”인 후보가 2~3개보다 많아질 수 있음.
- 그에 따라 `pickGradualStrength` / `pickRapidStrength`가 **best 한 개만** 고르는 상황이 줄어들 수 있음.
- 기력 구간별로 “탐색 폭”이 다르게 들어가서, 레벨에 따른 난이도 차이가 더 잘 나올 수 있음.

---

## 5. 참고

- 후보 수가 적었던 이유: `docs/WHY_FEW_SUGGESTIONS.md`
- 레벨별 play.cfg 값: `docs/PLAY_MODE_BOT_SPEC.md` §3
- KataGo SearchParams: `katago_src/cpp/search/searchparams.cpp`, `setup.cpp` (loadSingleParams에서 cfg 읽음)
