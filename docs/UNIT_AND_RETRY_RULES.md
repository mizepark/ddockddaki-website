# 제한/재시도 단위 규칙 (Unit and Retry Rules)

**핵심 규칙**: "~당 N번" 제한/재시도를 넣을 때, **코드에서 세는 단위(키)가 의도한 "~"와 반드시 일치**해야 한다. 단위가 어긋나면 재시도가 무한히 돌거나, 제한이 전혀 동작하지 않는다.

---

## 금지 패턴 (절대 하지 말 것)

| 잘못된 예 | 이유 |
|-----------|------|
| "한 번의 분석 요청당 1번만 재시도"인데 **requestId**로 횟수 세기 | 재시도할 때마다 `runAnalysisInternal`에서 **새 requestId**가 부여됨 → 매번 "첫 실패"로 인식되어 무한 재시도. |
| "세션당 1회"인데 **매 요청마다 새로 부여되는 ID**로 세기 | 동일. 키가 매번 바뀌면 "당"이 의미 없음. |
| "이 화면에서 1번만"인데 **매번 새로 생성되는 객체/ID**로 세기 | 동일. |

**규칙**: 제한/재시도의 **단위가 바뀌지 않는 키**를 써야 한다.  
- "같은 국면당" → `movesHash` (국면이 바뀔 때만 바뀜)  
- "이 호출당" → `repeat(N)` 루프 자체 (별도 키 없음, 루프가 한 "호출")  
- "이 컨트롤러 인스턴스당" → 인스턴스 필드로 카운트 (생명주기가 같음)

---

## 코드 적용 체크리스트

제한/재시도 로직을 추가·수정할 때 반드시 확인:

1. **의도한 단위**: "____ 당 N번"에서 빈칸에 들어가는 것이 뭔지 명시.
2. **실제 키**: 코드에서 횟수를 세는 데 사용하는 변수/식이 그 단위와 **같은 생명주기**를 가지는지 확인.
3. **키가 매번 바뀌는 값이면** (requestId, 새 객체 등) 사용 금지. 단위에 맞는 키로 교체.

---

## 전역 인벤토리 (제한/재시도 사용처)

| 위치 | 제한/재시도 내용 | 단위(per what) | 사용 키 | 상태 |
|------|------------------|----------------|---------|------|
| `EngineViewModel` | 빈 분석 결과 재시도 | **동일 국면(movesHash)당** MAX_EMPTY_RETRY회 | `emptyFinalRetryMovesHash`, `emptyFinalRetryCount` | ✅ movesHash 기준 |
| `AnalysisStateMachine` / `AnalysisIntentFilter` | 같은 의도 중복 실행 스킵 | **동일 position당** 1회 실행 | `samePosition(session, mode, movesHash, retryToken)` | ✅ position 정의 명확 |
| `AnalysisScheduler` | 분석 보호 구간 | **한 번 분석 시작 후** minAnalysisProtectMs 동안 추가 실행 방지 | `analysisStartTimeMs`, `isAnalysisRunning` | ✅ 실행 시작 시점 기준 |
| `EngineCalls.executeWithRetry` | AIDL 호출 실패 시 재시도 | **이번 호출(invocation)당** maxAttempts회 | 루프 `repeat(maxAttempts)` (별도 키 없음) | ✅ 호출당 1회 실행 범위 |
| `PlayViewModel` | 봇 착수 실패 시 재시도 | **이번 봇 턴(한 번의 selectBotMoveAndPlay)당** BOT_MOVE_MAX_ATTEMPTS회 | `repeat(BOT_MOVE_MAX_ATTEMPTS)` 루프 | ✅ 턴당 1회 범위 |
| `RewardedAdController` | 광고 로드 실패 시 재시도 | **이 인스턴스의 로드 시도 세션당** MAX_RETRY_COUNT회 | `retryCount` (성공 시 0으로 리셋) | ✅ 인스턴스·로드 세션 기준 |
| `BannerAd` | 배너 로드 재시도 | 로드 시도당 (명시적 상한 없을 수 있음) | `retryDelayMs` 백오프만 있음 | ⚠ 필요 시 최대 횟수+단위 명시 |
| `EngineInitCoordinator` | ensureReady 재시도 | **한 번의 초기화 시도당** | 루프 내 retry 1회 | ✅ 호출당 |
| `AssetInstaller.prepareModelWithRetry` | 모델 준비 재시도 | **한 번의 prepare 호출당** | 루프 | ✅ 호출당 |

---

## SafetyLimits 상수와 단위

- `EMPTY_FINAL_RETRY_MAX`: **동일 국면(movesHash)당** 빈 분석 결과 재시도 최대 횟수. **requestId 사용 금지.**
- `BOT_MOVE_MAX_ATTEMPTS`: **한 봇 턴당** 착수 시도 최대 횟수.
- 기타 재시도/타임아웃 상수도 "어떤 단위당"인지 주석으로 명시.

---

이 문서는 새 제한/재시도 로직 추가 시 참고하고, 리뷰 시 "단위-키 일치" 여부를 반드시 확인하기 위한 것이다.
