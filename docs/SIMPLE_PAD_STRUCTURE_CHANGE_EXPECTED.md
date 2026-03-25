# simple_pad_app 구조로 변경 시 예상되는 변화

이 문서는 현재 앱(Hope/KataGo Android)을 **simple_pad_app** 폴더 구조대로 변경했을 때 예상되는 변화를 정리한 것입니다.

---

## 1. 프로젝트·모듈 구조

| 항목 | 현재 | 변경 후 예상 |
|------|------|----------------|
| **루트 프로젝트명** | `Hope` (settings.gradle.kts) | `SimplePadApp` 등 단순 이름으로 통일 가능 |
| **모듈 구성** | `:app` + `:katago_assets` (2개) | **옵션 A:** 단일 `:app` (에셋을 app 내부로 통합)<br>**옵션 B:** `:app` + `:katago_assets` 유지 (구조만 정리) |
| **루트 build.gradle.kts** | 플러그인 7개 (application, kotlin, compose, gms, crashlytics, asset-pack) | simple_pad_app 수준이면 application + kotlin 2개만. 현재 기능 유지 시에는 그대로 두되 버전/정리만 |

**변화 요약:** 모듈을 줄이면 에셋 복사·의존성 태스크 정리가 필요하고, 유지하면 `include`와 의존성만 명확히 하면 됨.

---

## 2. 패키지·소스 디렉터리 구조

### 현재 (복잡한 계층)

```
com.katago.android/
├── MainActivity.kt, KataGoApp.kt, Constants.kt
├── ads/           (2)
├── analytics/      (1)
├── audio/          (1)
├── billing/        (1)
├── config/         (1)
├── data/repository/(4)
├── domain/usecase/ (3)
├── engine/         (24개 + rules/)
├── error/          (1)
├── ui/
│   ├── home/, main/, minigame/, offline/, play/, sandbox/
│   ├── theme/, viewmodel/
│   ├── Board.kt, BannerAd.kt, ScoreSummaryOverlay.kt, ...
└── util/           (3)
```

### 변경 후 (simple_pad_app 스타일)

```
com.simple.pad/  (또는 com.katago.android 유지 시에도 폴더만 단순화)
├── MainActivity.kt
├── AppViewModel.kt
└── engine/
    ├── PadEngine.kt      (인터페이스)
    ├── PadModels.kt      (Request/Result, Model, Strength 등)
    └── KataGoPadEngine.kt (실제 구현: 기존 NativeBridge·Service·Adapter 래핑)
```

**예상되는 구체적 변화:**

- **패키지 수:** 10개 이상 → 2개 수준 (`com.simple.pad`, `com.simple.pad.engine`).
- **ads, analytics, billing, config, error**  
  → 제거하거나, 유지 시 `engine` 옆에 `billing`, `ads` 등 최소 패키지로만 유지.
- **data/repository, domain/usecase**  
  → ViewModel/엔진 레이어로 흡수되거나, `engine` 내부 헬퍼로 통합. “Repository/UseCase” 명칭이 사라질 수 있음.
- **ui/**  
  → 화면이 한 개면 `MainActivity` + `AppViewModel`만 두고, Compose UI는 같은 패키지 또는 `ui/` 한 단계만 둠.  
  → 여러 화면 유지 시에는 `ui/main`, `ui/play` 등은 유지하되 `com.simple.pad.ui` 아래로 단순화.

---

## 3. 엔진 레이어

### 현재

- **AIDL:** `IKataGoEngine`, `IKataGoCallback` (네이티브 프로세스와 통신).
- **서비스:** `KataGoService` (엔진 프로세스 생명주기).
- **Kotlin:** `NativeBridge`, `EngineController`, `EngineAdapter`, `EngineViewModel`, `EngineLifecycleCoordinator`, `Session`, `GameStateUseCase` 등 20여 개.
- **네이티브:** `src/main/cpp` (KataGo C++, JNI).

### 변경 후 예상

- **외부에 노출되는 API:** `PadEngine` 인터페이스 하나.
  - `suspend fun analyze(request): AnalysisResult`
  - `suspend fun hint(request): HintResult`
  - `suspend fun play(request): PlayResult`
- **구현체:** `KataGoPadEngine` (또는 유지 시 이름만 정리한 하나의 진입점).
  - 내부에서 기존 `NativeBridge` ↔ `KataGoService` ↔ AIDL 호출을 그대로 사용하거나, 한 겹으로 래핑.
  - `EngineController`, `EngineAdapter`, `Session` 등은 **내부 구현 디테일**이 되어 외부에서는 보이지 않음.
- **모델:** `PadModels.kt`처럼 `AnalysisRequest`, `PlayRequest`, `Move`, `AnalysisResult` 등을 한 곳에서 정의.  
  기존 `EngineTypes.kt`, `Move.kt` 등은 이 모델로 매핑되거나 대체됨.

**변화 요약:**  
엔진을 “한 인터페이스 + 한 구현 진입점 + 공통 모델”로 정리. AIDL/네이티브/JNI는 그대로 두되, 호출 경로가 `AppViewModel` → `PadEngine` → (기존 코드)로 단순화됨.

---

## 4. UI·화면

### 현재

- **Activity:** `MainActivity` (Compose, 네비게이션/화면 전환).
- **화면:** Home, Main(보드/분석), Play(대국), Sandbox, Offline, MiniGame, 토큰/설정 등.
- **ViewModel:** `PlayViewModel`, `EngineViewModel`, `AnalysisUiViewModel`, `OfflineViewModel`, `SandboxViewModel`, `MiniGameViewModel`, `TokenUiViewModel` 등 다수.
- **Contract:** `PlayScreenContracts`, `MainScreenContracts` 등.

### simple_pad_app 수준으로 단순화할 경우

- **Activity:** 1개 (`MainActivity`).
- **화면:** 1개. 모드(분석/대국), 모델, 기력/분석타입, 수 입력, 실행 버튼, 결과/기록을 한 화면에.
- **ViewModel:** 1개 (`AppViewModel`). 모든 UI 상태와 분석/힌트/대국 요청 보유.
- **Contract:** 제거 또는 최소화.

**기능 유지하면서 구조만 맞출 경우**

- 화면은 그대로 두되, **진입점을 단순화:** `MainActivity` + `AppViewModel`만 루트에 두고, 기존 화면들은 `com.simple.pad.ui.*` 아래로 이동.
- 각 화면 ViewModel은 유지하되, **엔진 호출은 모두 `PadEngine` 인터페이스를 통해서만** 하도록 정리.

---

## 5. 빌드·의존성

### 현재 (app/build.gradle.kts)

- **Android:** namespace, NDK, CMake, externalNativeBuild, assetPacks, signingConfigs, productFlavors, buildTypes, AIDL, ProGuard 등.
- **의존성:** Compose BOM, billing, play-services-ads, Firebase (crashlytics, analytics), Play Review/Asset Delivery, Timber, 보안 등.

### 변경 후 예상

- **simple_pad_app과 동일하게 가져가려면:**  
  NDK/CMake/AIDL/에셋팩/Flavor/ProGuard 제거 → **현재 앱의 KataGo·결제·광고 기능을 포기**해야 함.
- **현재 기능을 유지하면서 “구조만” 맞추려면:**  
  빌드 설정은 거의 그대로 두고, **모듈/소스 경로/패키지명**만 simple_pad_app 스타일로 정리.  
  의존성은 유지.

---

## 6. 리소스·에셋·네이티브

| 항목 | 현재 | 구조 변경 시 예상 |
|------|------|-------------------|
| **res/** | drawable, mipmap, values, raw, xml, 다국어 | 유지. 필요 시 경로만 정리. |
| **assets** | katago(모델·설정), gnugo, ui 이미지, privacy_policy | 단일 모듈이면 app 내부로 통합. |
| **jniLibs** | libgnugo_android.so 등 | 유지. |
| **cpp/** | KataGo 네이티브 빌드 | 유지. 엔진 구조만 Kotlin 쪽에서 단순화. |
| **aidl/** | IKataGoEngine, IKataGoCallback | 유지. PadEngine이 이걸 감싸는 형태. |

**변화:** “폴더 구조”만 맞추는 경우 리소스/에셋/네이티브는 **위치·이름만 정리**하고 기능은 유지.  
완전히 simple_pad_app처럼 만든다면 네이티브·AIDL·대용량 에셋 제거로 **앱이 목업/데모 수준**으로 바뀜.

---

## 7. 제거·축소될 수 있는 것 (완전 단순화 시)

- **모듈:** `katago_assets` (app으로 통합 시).
- **패키지/클래스:** ads, analytics, billing, config, domain/usecase, data/repository 대부분, error 전용 처리, 다수 ViewModel/Contract.
- **빌드:** productFlavors, ProGuard 규칙 상당 부분, Firebase/Crashlytics, Billing, Ads 관련 설정.
- **화면:** Home, Sandbox, Offline, MiniGame, 토큰 구매 등 → 한 화면으로 통합 또는 제거.

**구조만 맞추고 기능 유지 시:** 위 항목은 **제거되지 않고** 패키지/폴더 위치만 `com.simple.pad.*` 스타일로 재배치됨.

---

## 8. 요약 표

| 영역 | “구조만” simple_pad_app 스타일 | “기능까지” simple_pad_app 수준 |
|------|--------------------------------|--------------------------------|
| **모듈** | 1~2개 유지, 경로·이름 정리 | 단일 `:app` |
| **패키지** | 2~3단계로 단순화 (pad, pad.engine) | 2단계 (pad, pad.engine) |
| **엔진** | PadEngine 인터페이스 + 1개 구현체로 진입 통일 | 동일 + 내부는 목업 가능 |
| **UI** | 화면/ViewModel 유지, 루트만 단순화 | 1 Activity, 1 ViewModel, 1 화면 |
| **의존성** | 유지 | 최소 (Compose, lifecycle 등) |
| **네이티브/AIDL** | 유지 | 제거(목업만) 또는 유지 선택 |
| **예상 작업량** | 중간 (리팩터·이동·의존성 정리) | 매우 큼 (기능 제거·통합·테스트) |

---

## 9. 결론

- **simple_pad_app “폴더 구조”로 변경했을 때 예상되는 변화**는  
  - **프로젝트/모듈/패키지/엔진 진입점/루트 UI**가 단순해지고,  
  - **엔진은 PadEngine 한 겹 뒤로 숨고**,  
  - **화면/ViewModel을 한 개로 줄이면** simple_pad_app과 비슷한 형태가 됨.
- **기능을 유지하면서 구조만 맞추면**  
  - 디렉터리·패키지·엔진 추상화·빌드 정리 위주로 변화하고,  
  - 네이티브·에셋·부가 기능은 그대로 두고 **호출 경로와 진입점만 단순화**된다고 보면 됨.

이 문서는 “예상되는 변화”만 설명하며, 실제 변경 작업 범위는 “구조만” vs “기능 단순화까지” 선택에 따라 크게 달라집니다.
