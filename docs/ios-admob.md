# iOS Integration (AdMob)

This guide shows you how to integrate the Orivia SDK with Google AdMob in iOS

## Import Dependencies

Add the Orivia specs repository, then the core SDK and AdMob mediation pod, to your `Podfile`:

```ruby
source 'https://github.com/orivia-ai/orivia-specs.git'
source 'https://cdn.cocoapods.org/'

pod 'OriviaMonetization', '1.18.0'
pod 'OriviaMonetizationAdmob', '1.18.0'
```

Run `pod install`.

## Import Package

Make sure the Orivia SDK modules are available in your files:

```swift
import OriviaMonetization
import OriviaMonetizationAdmob
import GoogleMobileAds
```

## Define Ad Unit IDs

Declare your Orivia publisher ID and ad unit IDs:

```swift
private let oriviaPublisherId = "YOUR_ORIVIA_PUBLISHER_ID"

private let interstitialAdUnitId = "YOUR_INTERSTITIAL_AD_UNIT_ID"
private let rewardedAdUnitId = "YOUR_REWARDED_AD_UNIT_ID"
private let bannerAdUnitId = "YOUR_BANNER_AD_UNIT_ID"
```

## Initialize AdMob & Orivia

Init AdMob:

```swift
MobileAds.shared.start { _ in
    // AdMob SDK initialized
}
```

Init Orivia with `.admob` mediation (any time after AdMob init, but before requesting ads):

```swift
OriviaSdk.shared.initialize(
    config: OriviaSdk.Config(
        mediation: .admob,
        publisherId: oriviaPublisherId,
        defaultBannerAdUnit: bannerAdUnitId,
        defaultInterstitialAdUnit: interstitialAdUnitId,
        defaultRewardedAdUnit: rewardedAdUnitId,
        dataCollectionOnly: false // optional, false by default
    )
) { result in
    switch result {
    case .success:
        print("Orivia SDK initialized.")
    case .failure(let error):
        print("Orivia init failed: \(error.description)")
    }
}
```

!!! info "Info"
    Orivia is ready immediately after calling `initialize`. You can load or show ads right away. If no config is loaded, default IDs are used.

!!! info "Info"
    Unlike MAX, AdMob has no runtime SDK key — its App ID is configured via `Info.plist` instead.

!!! warning "Important"
    The `completion` closure is triggered after the SDK loads its first config from the server.
    To ensure optimal ads performance, it's **highly recommended** to wait for this callback **before** starting to load any ads.

!!! warning "Important"
    Calling `initialize` multiple times **does not** reinitialize the SDK if it's already initialized or currently initializing.
    The method will only run again if the SDK has previously invoked the failure case.

    If `initialize` is called after the SDK has been initialized, the completion is invoked immediately with `.success`.
    If `initialize` is called while initialization is in progress, the newly provided completion will be notified as soon as initialization finishes, **together** with all previous completions.

If a certain ad type isn't used, just pass the constant `OriviaSdk.adUnitIdEmpty` instead of ad unit id.

## Store AdMob Ad Results

Create a single `OriviaAdmobHelper` instance at class level:

```swift
private let oriviaAdmobHelper = OriviaAdmobHelper()
```

This helper will remember the last success/failure per ad type and feed it to Orivia SDK.

## Interstitial

### Load & Show Interstitial

Use `OriviaAdmobHelper` instance to pick the ad unit id, load via AdMob, then forward the load/revenue events back to the helper:

```swift
final class InterstitialDelegate: NSObject, FullScreenContentDelegate {

    private let helper: OriviaAdmobHelper

    init(helper: OriviaAdmobHelper) {
        self.helper = helper
    }

    func adDidDismissFullScreenContent(_ ad: any FullScreenPresentingAd) {
        // Handle interstitial hidden
    }

    func ad(_ ad: any FullScreenPresentingAd, didFailToPresentFullScreenContentWithError error: Error) {
        // Handle interstitial display failed
    }
}

private var interstitialAd: InterstitialAd?
private var interstitialDelegate: InterstitialDelegate?

func loadInterstitial() {
    let adUnitId = oriviaAdmobHelper.getAdUnitId(for: .interstitial)
    InterstitialAd.load(with: adUnitId, request: Request()) { [weak self] ad, error in
        guard let self = self else { return }
        if let error = error {
            self.oriviaAdmobHelper.onAdLoadFailed(adType: .interstitial, adUnitId: adUnitId, error: error)
            // Handle your own interstitial failed logic
            return
        }
        guard let ad = ad else { return }

        let delegate = InterstitialDelegate(helper: self.oriviaAdmobHelper)
        ad.fullScreenContentDelegate = delegate
        ad.paidEventHandler = { [weak self] adValue in
            self?.oriviaAdmobHelper.onPaidEvent(adType: .interstitial, adUnitId: adUnitId, adValue: adValue, responseInfo: ad.responseInfo)
        }

        self.interstitialDelegate = delegate
        self.interstitialAd = ad
        self.oriviaAdmobHelper.onAdLoaded(adType: .interstitial, adUnitId: adUnitId, responseInfo: ad.responseInfo)
        // Handle your own interstitial loaded logic
    }
}

func showInterstitial(from presenter: UIViewController) {
    if let ad = interstitialAd {
        ad.present(from: presenter)
    }
}
```

`getAdUnitId(for:)` returns the AdMob ad unit ID to use for the ad request.

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.

    It's critical for the SDK to receive all events.

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Rewarded

### Load & Show Rewarded

Use `OriviaAdmobHelper` instance to pick the ad unit id, load via AdMob, then forward the load/revenue events back to the helper:

```swift
final class RewardedDelegate: NSObject, FullScreenContentDelegate {

    private let helper: OriviaAdmobHelper

    init(helper: OriviaAdmobHelper) {
        self.helper = helper
    }

    func adDidDismissFullScreenContent(_ ad: any FullScreenPresentingAd) {
        // Handle rewarded hidden
    }

    func ad(_ ad: any FullScreenPresentingAd, didFailToPresentFullScreenContentWithError error: Error) {
        // Handle rewarded display failed
    }
}

private var rewardedAd: RewardedAd?
private var rewardedDelegate: RewardedDelegate?

func loadRewarded() {
    let adUnitId = oriviaAdmobHelper.getAdUnitId(for: .rewarded)
    RewardedAd.load(with: adUnitId, request: Request()) { [weak self] ad, error in
        guard let self = self else { return }
        if let error = error {
            self.oriviaAdmobHelper.onAdLoadFailed(adType: .rewarded, adUnitId: adUnitId, error: error)
            // Handle your own rewarded failed logic
            return
        }
        guard let ad = ad else { return }

        let delegate = RewardedDelegate(helper: self.oriviaAdmobHelper)
        ad.fullScreenContentDelegate = delegate
        ad.paidEventHandler = { [weak self] adValue in
            self?.oriviaAdmobHelper.onPaidEvent(adType: .rewarded, adUnitId: adUnitId, adValue: adValue, responseInfo: ad.responseInfo)
        }

        self.rewardedDelegate = delegate
        self.rewardedAd = ad
        self.oriviaAdmobHelper.onAdLoaded(adType: .rewarded, adUnitId: adUnitId, responseInfo: ad.responseInfo)
        // Handle your own rewarded loaded logic
    }
}

func showRewarded(from presenter: UIViewController) {
    if let ad = rewardedAd {
        ad.present(from: presenter) { /* reward received */ }
    }
}
```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.

    It's critical for the SDK to receive all events.

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Extra Value

Returns a server-configured string value for the given ad type and key.

```swift
let value: String? = OriviaSdk.shared.getExtraValue(adType: .interstitial, key: "my_key")
```

## Ad Config Update Listener

Get notified whenever the server-driven ad config for a given ad type changes:

```swift
class AdConfigObserver: OriviaSdk.AdConfigUpdateListener {
    func onAdConfigUpdated(adType: AdType) {
        // ad config for adType was refreshed
    }
}

let observer = AdConfigObserver()
OriviaSdk.shared.addAdConfigUpdateListener(adType: .interstitial, listener: observer)

// Later, when no longer needed:
OriviaSdk.shared.removeAdConfigUpdateListener(adType: .interstitial, listener: observer)
```

!!! warning "Important"
    The listener fires on every config refresh, not only when a value actually changed, and is held with a strong reference until removed — keep a reference to `observer` yourself and always call `removeAdConfigUpdateListener` to avoid leaking.

## Client Parameters

To pass client parameters to the server:

```swift
let nested = ValueMap.Builder().put("nested", "value").build()
let clientParams = ValueMap.Builder()
    .put("str", "value")
    .put("int", 12)
    .put("float", 1.3)
    .put("bool", true)
    .put("nested", nested)
    .build()
OriviaSdk.shared.setClientParams(clientParams)
```

If the parameters should be passed in the init request, the `setClientParams` method must be called before invoking the `initialize` method.

!!! warning "Important"
    Passing `nil` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

```swift
OriviaSdk.shared.setLoggingEnabled(true)
```

Logs appear in the **Xcode Console**.

## Privacy

To pass GDPR Applies there are 2 options.

Pass it through CMP [IABTCF_gdprApplies](https://github.com/InteractiveAdvertisingBureau/GDPR-Transparency-and-Consent-Framework/blob/aa079f33b574bf5fe48594719cbe7e99d848bcd4/TCFv2/IAB%20Tech%20Lab%20-%20CMP%20API%20v2.md#in-app-details) property or using SDK method:

```swift
OriviaSdk.shared.setGdprApplies(true) // or false, or nil for unknown
```

To pass COPPA:

```swift
OriviaSdk.shared.setCoppa(true) // or false, or nil for unknown
```

Null values reset properties.
