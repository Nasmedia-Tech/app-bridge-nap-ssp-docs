# NapSSP App Bridge Guide

> **NapSSP SDK** × **App-WebView 하이브리드 브릿지** 연동 가이드 및 통합 패키지

![Platform](https://img.shields.io/badge/platform-Android%20%7C%20iOS-blue)
![Android](https://img.shields.io/badge/Android-Kotlin%20%7C%20Jetpack%20Compose-brightgreen)
![iOS](https://img.shields.io/badge/iOS-Swift%20%7C%20SwiftUI-lightgrey)
![JS](https://img.shields.io/badge/Web-Vanilla%20JS-yellow)
![License](https://img.shields.io/badge/license-MIT-green)

---

## 개요

웹(WebView) 화면에서 JavaScript 한 줄로 NapSSP 광고를 요청하고, 네이티브 앱(Android / iOS)이 실제 광고를 렌더링하는 **하이브리드 브릿지 패턴**의 레퍼런스 구현체입니다.

```
┌──────────────────────────────────────────┐
│               WebView (JS)               │
│  NapSspBridge.loadAd('rewardVideo')  ──► │──► Native Bridge
│  NapSspBridge.onEvent = fn(…)        ◄── │◄── SDK 이벤트
└──────────────────────────────────────────┘
```

### 지원 광고 포맷

| 포맷 | 설명 |
|---|---|
| `banner` | 배너 광고 (기본 320×100, size 파라미터로 변경 가능) |
| `native` | 네이티브 광고 |
| `video` | 인스트림 비디오 |
| `rewardVideo` | 보상형 비디오 |
| `interstitialVideo` | 전면 비디오 |
| `interstitialBanner` | 전면 배너 |

#### 배너 사이즈 — 동적 처리 (하드코딩 불필요)

NapSSP SDK는 배너 사이즈를 **Ad Unit ID 기준으로 서버에서 결정**합니다. `size` 파라미터를 `"WxH"` 형식으로 전달하면 Native가 높이를 자동으로 파싱합니다. **신규 사이즈 추가 시 코드 변경이 필요 없습니다.**

```javascript
callNative('loadAd', {
  format: 'banner',
  adUnitId: '발급받은_해당_사이즈_AD_UNIT_ID',
  size: '360x230'  // "WxH" 형식 — Native에서 높이 파싱
});
```

```kotlin
// Android: 파싱으로 높이 결정 (하드코딩 없음)
val h = params.optString("size").split("x").getOrNull(1)?.toIntOrNull() ?: 100
adHeight = h.dp
```

> NaverAdManager는 360×230, Kakao AdFit은 360×210 사이즈를 해당 어댑터 전용으로 지원합니다. 각 사이즈별 전용 Ad Unit ID를 파트너 사이트에서 별도 발급받으세요.

---

## 빠른 시작

### 1. WebView에 JS 브릿지 포함

```html
<script src="napssp-bridge.js"></script>
```

### 2. SDK 초기화

```js
NapSspBridge.init();
```

### 3. 광고 요청

```js
NapSspBridge.loadAd('banner');           // 기본 배너
NapSspBridge.loadAd('rewardVideo');      // 보상형 비디오
NapSspBridge.loadAd('banner', '104704'); // Ad Unit ID 직접 지정
```

### 4. 이벤트 수신

```js
NapSspBridge.onEvent = function(event, format, detail) {
    if (event === 'rewarded') {
        // 보상 지급 로직
    }
};
```

---

## 파일 구성

```
integration-package/
├── android/
│   ├── NapSspConfig.kt              Android 키 설정
│   ├── NapSspSdkIntegration.kt      Android SDK 엔진 (6개 포맷)
│   └── HybridWebViewScreen.kt       Compose WebView 화면
├── ios/
│   ├── NapSspConfig.swift           iOS 키 설정
│   ├── NapSspSdkIntegration.swift   iOS SDK 엔진 (6개 포맷)
│   ├── HybridWebViewScreen.swift    SwiftUI WKWebView 화면
│   └── Bridge/
│       ├── NapSspAdEventBridge.swift
│       ├── InterstitialModule.swift
│       └── RewardedModule.swift
└── web/
    ├── napssp-bridge.js             JS 브릿지 유틸리티
    └── sample.html                  광고 테스트 페이지
```

---

## Android 연동 요약

**1. `settings.gradle.kts`에 Maven 저장소 추가**

```kotlin
// AdFit / Pangle 미디에이션 사용 시 필요
maven { url = uri("https://devrepo.kakao.com/nexus/content/groups/public/") }
maven { url = uri("https://artifact.bytedance.com/repository/pangle/") }
```

**2. `build.gradle.kts`에 SDK 의존성 추가**

```kotlin
// Core (필수)
implementation("io.github.nasmedia-tech:admixer-ssp:1.0.23")
implementation("com.google.android.gms:play-services-ads-identifier:18.9.0")
// 미디에이션 어댑터 (사용할 것만 선택)
implementation("io.github.nasmedia-tech:admixer-admanager:1.0.14")
implementation("io.github.nasmedia-tech:admixer-adfit:1.0.10")
implementation("io.github.nasmedia-tech:admixer-pangle:1.0.10")
implementation("io.github.nasmedia-tech:admixer-applovin:1.0.8")
implementation("io.github.nasmedia-tech:admixer-unity:1.0.6")
```

**3. `integration-package/android/` 파일 3개를 프로젝트에 복사**

**4. `NapSspConfig.kt`에서 Ad Unit ID 설정**

```kotlin
const val BANNER_AD_UNIT_ID   = 0  // TODO: 실제 발급된 ID로 교체
const val NATIVE_AD_UNIT_ID   = 0
const val VIDEO_AD_UNIT_ID    = 0
const val REWARD_AD_UNIT_ID   = 0
const val PANGLE_APP_ID       = "YOUR_PANGLE_APP_ID"
```

**5. `HybridWebViewScreen`을 WebView 화면으로 사용**

---

## iOS 연동 요약

**1-A. SPM 방식** — Xcode → File → Add Package Dependencies

```
# Core (필수)
https://github.com/Nasmedia-Tech/iOS-SSP-SPM.git           (최신: 1.1.5)
https://github.com/Nasmedia-Tech/iOS-SSP-Mediation-SPM.git (최신: 2.3.3)

# 미디에이션 어댑터 (사용할 것만 선택)
https://github.com/Nasmedia-Tech/iOS-SSP-GAM-SPM.git
https://github.com/Nasmedia-Tech/iOS-SSP-AdFit-SPM.git
https://github.com/Nasmedia-Tech/iOS-SSP-Pangle-SPM.git
https://github.com/Nasmedia-Tech/iOS-SSP-AppLovin-SPM.git
https://github.com/Nasmedia-Tech/iOS-SSP-UnityAds-SPM.git
```

**1-B. CocoaPods 방식** — `Podfile`

```ruby
platform :ios, '13.0'
target 'YourApp' do
  use_frameworks!
  pod 'AdMixerMediation'
  pod 'AdMixerMediationGAM'       # GAM 사용 시
  pod 'AdMixerMediationAdFit'     # AdFit 사용 시
  pod 'AdMixerMediationPangle'    # Pangle 사용 시
  pod 'AdMixerMediationAppLovin'  # AppLovin 사용 시
  pod 'AdMixerMediationUnityAds'  # Unity Ads 사용 시
end
```

**2. `Info.plist`에 필수 키 추가**

```xml
<key>NSAppTransportSecurity</key>
<dict><key>NSAllowsArbitraryLoads</key><true/></dict>
<!-- GAM 사용 시 -->
<key>GADApplicationIdentifier</key>
<string>ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX</string>
<!-- ATT (iOS 14.5+) -->
<key>NSUserTrackingUsageDescription</key>
<string>맞춤 광고 제공을 위해 광고 추적 권한이 필요합니다.</string>
```

**3. `integration-package/ios/` 파일들을 프로젝트에 추가**

**4. `NapSspConfig.swift`에서 Ad Unit ID 설정**

```swift
static var bannerAdUnitId:   Int { 0 }  // TODO: 실제 ID로 교체
static var nativeAdUnitId:   Int { 0 }
static var videoAdUnitId:    Int { 0 }
static var rewardAdUnitId:   Int { 0 }
```

**5. `HybridWebViewScreen`을 화면으로 사용**

---

## 상세 가이드

전체 연동 가이드 (아키텍처, 코드 설명, FAQ 포함):

👉 **[guide.md](./guide.md)**

---

## 브릿지 메시지 스펙

### JS → Native

```json
{ "action": "loadAd", "params": { "format": "banner", "adUnitId": "104704" } }
```

| action | params | 설명 |
|---|---|---|
| `init` | — | SDK 초기화 |
| `loadAd` | `format`, `adUnitId?` | 광고 로드 요청 |
| `clearAds` | — | 표시 중인 광고 해제 |

### Native → JS (`window.onNapSspMessage`)

```json
{ "action": "event", "status": "success", "data": "[banner] loaded: 104704" }
```

| action | data 예시 | 설명 |
|---|---|---|
| `init` | `"SDK Initialized"` | 초기화 응답 |
| `loadAd` | `"Accepted banner"` | 요청 수신 ACK |
| `event` | `"[banner] loaded: 104704"` | SDK 광고 이벤트 |
| `error` | `"Unknown action"` | 오류 응답 |

### 이벤트 종류

`loaded` · `displayed` · `clicked` · `rewarded` · `completed` · `skipped` · `closed` · `failed`

---

## 주의사항

- `loadAd` 응답은 **ACK** (요청 수신 확인)이며, 광고 로드 성공을 의미하지 않습니다.
- 실제 로드 성공/실패는 `event` action의 `loaded` / `failed` 이벤트로 전달됩니다.
- 클라이언트(JS) 600ms, 네이티브 500ms **디바운스** 가 적용되어 연속 요청이 차단됩니다.
- `adUnitId`는 Android에서 `Int`, iOS에서 `Int` 타입이므로 JS에서 숫자 문자열로 전달하세요.

---

## 문의

- **Nasmedia 기술지원**: [https://github.com/Nasmedia-Tech](https://github.com/Nasmedia-Tech)
