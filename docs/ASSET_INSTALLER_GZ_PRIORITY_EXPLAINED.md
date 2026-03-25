# AssetInstaller ".gz 우선" 로직이 무슨 뜻인지

## 1. "엔진 파일이 제멋대로 있다"는 말이 맞나?

**아니요.** 엔진/모델 파일이 아무렇게나 들어가 있는 게 아닙니다.

- **의도된 구조**: `katago_assets`에는 **b10.bin.gz**, **b6.txt.gz** 만 넣고, 빌드 시 `copyKatagoModelAssets`가 이 두 파일만 `app/src/main/assets/katago`로 복사합니다.
- 즉, **정상적으로는 에셋에는 .gz 압축 파일만** 있어야 합니다. 압축을 푼 `b10.bin`을 에셋 소스에 넣는 구성이 아닙니다.

그런데도 **"b10.bin과 b10.bin.gz가 둘 다 있다"**는 로그가 나올 수 있는 경우는 예를 들면:

- **에셋이 두 경로에서 합쳐질 때**: base APK용으로 복사한 `app/.../assets/katago`와, asset pack `katago_assets`가 같이 쓰이면서, 기기/빌드에 따라 `assets.list("katago")` 결과에 **.gz와 .bin이 둘 다** 들어갈 수 있음.
- 또는 예전 빌드/다른 브랜치에서 **압축 푼 b10.bin을 assets에 넣어둔** 적이 있거나, 다른 태스크가 .bin을 만들어 둔 경우.

그래서 "파일이 제멋대로 있다"기보다는, **일부 환경에서는 에셋 목록에 .gz와 .bin이 둘 다 나올 수 있다**고 가정하고 있는 것입니다.

---

## 2. 그때 "어떤 파일을 쓸지"만 정하는 거다

- **.gz를 우선**: `b10.bin.gz`가 목록에 있으면 **무조건 이걸 선택**하고, `GZIPInputStream`으로 압축을 풀어서 설치합니다.
- **.bin만 있으면**: `b10.bin.gz`가 없을 때만 `b10.bin`을 골라서, 압축 해제 없이 그대로 복사합니다.

즉, **설치할 “원본”을 고를 때의 우선순위**를 바꾼 것이지,  
에셋 폴더 안에 어떤 파일을 넣을지(빌드/에셋 구조)를 막 바꾼 게 아닙니다.

---

## 3. 왜 .gz를 우선하게 했나

- **.gz**: 우리가 패키징한 **원본 압축 파일**이라, 이걸 풀면 형식이 맞는 모델이 나옵니다.
- **.bin**: 이미 풀린 파일인데,  
  - 어디서 생성됐는지(다른 빌드/다른 경로),  
  - 이중 압축이었는지,  
  - 손상/잘못된 파일이 섞였는지  
  알기 어렵습니다. 이걸 그대로 쓰면 엔진이 읽지 못해 "Zombie State (Empty Analysis)"처럼 동작할 수 있습니다.

그래서 **“에셋 목록에 둘 다 있을 수 있으면, 항상 .gz를 쓰자”**로 통일한 것입니다.  
엔진 파일이 제멋대로 있다는 뜻이 아니라, **그런 애매한 상황에서도 항상 검증된 .gz 기준으로 설치하자**는 의미입니다.

---

## 4. 현재 코드 상태

지금 `AssetInstaller.kt`는 이미 **.gz를 먼저 보는** 순서입니다.

```kotlin
// simpleName = "b10.bin.gz", simpleNameNoGz = "b10.bin"
if (availableAssets.contains(simpleName)) {   // .gz 있으면
     finalAssetPath = "katago/$simpleName"
     isGzip = true
} else if (availableAssets.contains(simpleNameNoGz)) {  // 없을 때만 .bin
     finalAssetPath = "katago/$simpleNameNoGz"
     isGzip = false
}
```

그래서 "순서를 바꿔서 .gz를 우선 사용하도록 수정했다"는 설명은,  
**예전에 반대 순서(.bin 우선)였던 것을 고친 것**이거나, **지금 적용된 동작을 설명한 것**으로 보면 됩니다.  
엔진 파일이 제멋대로 있는 게 아니라, **에셋 목록에 .gz와 .bin이 둘 다 나올 수 있는 경우를 대비해, 설치 시 항상 .gz를 우선 쓰도록 한 것**이라고 이해하면 됩니다.
