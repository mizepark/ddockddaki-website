# 엔진/모델 이름·포맷·설치 실패 원인 정리

## 1. 이름·포맷 확인 결과 (문제 없음)

### 1.1 에셋 파일 이름

| 위치 | 기대 이름 | 실제 (katago_assets) | 일치 |
|------|------------|----------------------|------|
| 에셋 경로 | `katago/b10.bin.gz` | `katago/b10.bin.gz` | ✅ |
| 에셋 경로 | `katago/b6.txt.gz` | `katago/b6.txt.gz` | ✅ |

- 코드 상수: `AssetInstaller` — `B10_ASSET_PATH = "katago/b10.bin.gz"`, `B6_ASSET_PATH = "katago/b6.txt.gz"`.
- `requiredAssetNames = listOf("b10.bin.gz", "b6.txt.gz")` 와 `availableAssets` 비교 시 **이름 불일치 없음**.

### 1.2 설치 후 파일 이름

| 용도 | 설치 경로 | 파일명 | 비고 |
|------|-----------|--------|------|
| 분석 엔진 | `context.filesDir/katago/` | `b10.bin` | B10은 .bin |
| 대국 엔진 (0~6) | 동일 | `b10.bin` | PlayLevelModelPolicy |
| 대국 엔진 (7~30) | 동일 | `b6.txt` | B6은 .txt (`.bin` 사용 시 에러로 명시됨) |

- `PlayLevelModelPolicy`: 0~6 → `b10.bin`, 7~30 → `b6.txt`. 에셋은 `b6.txt.gz` → 압축 해제 후 `b6.txt` 로 설치. **이름·포맷 일치**.

### 1.3 빌드 시 에셋 포함

- `copyKatagoModelAssets` 태스크: `katago_assets/src/main/assets/katago` 에서 **`b10.bin.gz`, `b6.txt.gz`만** `app/src/main/assets/katago` 로 복사.
- `merge*Assets` 가 이 태스크에 의존하므로, 정상 빌드 시 **base 모듈** 에 `katago/b10.bin.gz`, `katago/b6.txt.gz` 가 포함됨.
- `katago_assets` 는 **install-time** asset pack 으로도 포함. 즉, base 복사 + asset pack 두 경로로 동일 에셋 제공.

**결론: 이름·포맷·빌드 설정은 올바르게 맞춰져 있음.**

---

## 2. 그럼에도 설치가 안 될 수 있는 다른 이유

### 2.1 앱 업데이트 시 purge → 재설치 실패

- **흐름**: 버전 마이그레이션 시 `EngineStorageMaintenance.markOutdated(this)` 호출 → 다음 실행 시 `prepareModel` 안에서 `purgeIfRequested()` 가 **`filesDir/katago` 전체를 삭제**.
- 그 다음 `forceUpdate` 등으로 압축 해제·복사가 다시 실행되는데, 이때 **재설치가 실패하면** 디렉터리는 비워진 채로 남음.
- **가능한 실패 지점**  
  - `assets.list("katago")` 가 null/빈 리스트 (에셋 노출 안 됨)  
  - `openAssetStream(...)` 예외 (에셋을 열 수 없음)  
  - 디스크 부족, 권한, I/O 예외로 복사/이동 실패  
  - `tempFile.renameTo(destFile)` 실패 후 `copyTo` 까지 실패

→ **“Model b10.bin not ready (exists=false, size=0)”** 는 **purge 로 기존 파일을 지운 뒤, 재설치가 끝나기 전이거나 재설치가 실패한 상태**에서 `isModelReady()` 를 호출했을 때 나올 수 있음.

### 2.2 에셋이 런타임에 안 보이는 경우

- **base APK 에셋**: `copyKatagoModelAssets` 가 빌드에 포함되지 않았거나, clean/build 순서 문제로 `app/src/main/assets/katago` 에 파일이 없으면 base 에는 에셋이 없음.
- **Asset pack**: `katago_assets` 는 install-time 이지만, 기기/설치 방식에 따라 **첫 실행 시점에 pack 이 아직 merge 되지 않았을 수 있음**.  
  - `context.assets.open("katago/b10.bin.gz")` 실패  
  - `AssetPackManager.getPackLocation("katago_assets")` 가 null 이거나, 해당 경로에 파일이 없음  
→ `openAssetStream` 이 세 경로(applicationContext.assets, context.assets, pack 경로 파일) 모두에서 실패하면 예외 발생 → 복사 자체가 안 됨 → 설치 실패.

### 2.3 저장 공간·권한

- `prepareModel` 내부에서 **필요 공간 ~80MB** 체크함. 이 시점에 공간이 부족하면 `IOException` 으로 실패.
- `filesDir` 은 보통 권한 문제 없지만, **내부 저장소가 가득 찬 상태에서 복사 중** 실패하거나, `renameTo`/`copyTo` 가 일부 기기에서 실패할 수 있음.

### 2.4 skipModelInstall 으로 모델 설치를 건너뛴 경우

- `ensureEngineReady(..., skipModelInstall = true)` 로 호출되면 **prepareModel 을 아예 하지 않음**.  
  - 예: 재시작/설정 변경 후 “모델 재설치는 제외하고 엔진만 다시 초기화” 하는 경로.
- 이때 **이미 설치된 모델이 없으면** (첫 설치 직후가 아니고, purge 로 지워진 뒤 등) `getModelFile()` / `isModelReady()` 가 false 를 반환할 수 있음.

### 2.5 엔진 프로세스와 컨텍스트

- `getModelRoot(context)` = `File(context.filesDir, "katago")`.  
  같은 앱이면 `filesDir` 은 동일하므로, **프로세스가 달라도 “경로” 자체는 같음**.  
  다만 **설치를 수행한 쪽과 엔진을 초기화하는 쪽이 다른 라이프사이클**이면, “방금 설치했는데 아직 파일이 안 보인다” 같은 타이밍 이슈는 가능 (파일 시스템 동기화 등).

---

## 3. 요약

- **이름·포맷**: 에셋/설치 파일명·확장자 모두 코드와 실제 에셋과 일치. **이름/포맷 문제로 인한 설치 실패 가능성은 낮음.**
- **설치가 안 되는 다른 이유**:
  1. **앱 업데이트 후 purge** 로 `katago` 디렉터리를 비운 뒤, **재설치(압축 해제·복사)가 실패**한 경우.
  2. **에셋 접근 실패**: base/assets 또는 asset pack 에서 `katago/b10.bin.gz`, `katago/b6.txt.gz` 를 열지 못하는 경우 (빌드 누락, install-time pack 미적용 등).
  3. **저장 공간 부족** 또는 **I/O/권한**으로 복사·이동 실패.
  4. **skipModelInstall = true** 로 모델 설치를 건너뛴 상태에서, 이미 설치된 모델이 없는 경우.
  5. **purge 직후·재설치 직후** 등 **타이밍**으로 `isModelReady()` 가 아직 true 가 되기 전에 엔진 초기화가 시도되는 경우.

조치 권장: **앱 데이터 삭제 후 재실행**으로 purge/재설치를 한 번 더 유도하고, 그래도 실패하면 로그에서 `openAssetStream`/`prepareModel` 예외와 `isModelReady` 로그를 확인하는 것이 좋음.
