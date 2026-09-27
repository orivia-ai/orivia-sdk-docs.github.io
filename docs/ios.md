# iOS Integration

This guide shows you how to integrate the Orivia SDK with AppLovin MAX in iOS

## Import Dependencies

Add the Orivia specs repository, then the core SDK and MAX mediation pod, to your `Podfile`:

```ruby
source 'https://github.com/orivia-ai/orivia-specs.git'
source 'https://cdn.cocoapods.org/'

pod 'OriviaMonetization', '1.15.0'
pod 'OriviaMonetizationMax', '1.15.0'
```

Run `pod install`.

## Import Package

Make sure the Orivia SDK modules are available in your files:

```swift
import OriviaMonetization
import OriviaMonetizationMax
import AppLovinSDK
```

## Define SDK Keys & Ad Unit IDs

Declare your MAX SDK key, Orivia publisher ID, and ad unit IDs:

```swift
private let maxSdkKey = "YOUR_MAX_SDK_KEY"
private let oriviaPublisherId = "YOUR_ORIVIA_PUBLISHER_ID"

private let interstitialAdUnitId = "YOUR_INTERSTITIAL_AD_UNIT_ID"
private let rewardedAdUnitId = "YOUR_REWARDED_AD_UNIT_ID"
private let bannerAdUnitId = "YOUR_BANNER_AD_UNIT_ID"
```

## Initialize MAX & Orivia

Init MAX:

```swift
let initConfig = ALSdkInitializationConfiguration(sdkKey: maxSdkKey) { builder in
    builder.mediationProvider = ALMediationProviderMAX
}
ALSdk.shared().initialize(with: initConfig) { _ in
    // MAX SDK initialized
}
```

Init Orivia (any time after MAX init, but before requesting ads):

```swift
OriviaSdk.shared.initialize(
    config: OriviaSdk.Config(
        mediation: .max,
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

!!! warning "Important"
    The `completion` closure is triggered after the SDK loads its first config from the server.
    To ensure optimal ads performance, it's **highly recommended** to wait for this callback **before** starting to load any ads.

!!! warning "Important"
    Calling `initialize` multiple times **does not** reinitialize the SDK if it's already initialized or currently initializing.
    The method will only run again if the SDK has previously invoked the failure case.

    If `initialize` is called after the SDK has been initialized, the completion is invoked immediately with `.success`.
    If `initialize` is called while initialization is in progress, the newly provided completion will be notified as soon as initialization finishes, **together** with all previous completions.

If a certain ad type isn't used, just pass the constant `OriviaSdk.adUnitIdEmpty` instead of ad unit id.

For example, if the application doesn't have a key for a banner:

```swift
OriviaSdk.shared.initialize(
    config: OriviaSdk.Config(
        mediation: .max,
        publisherId: oriviaPublisherId,
        defaultBannerAdUnit: OriviaSdk.adUnitIdEmpty,
        defaultInterstitialAdUnit: interstitialAdUnitId,
        defaultRewardedAdUnit: rewardedAdUnitId,
        dataCollectionOnly: false // optional, false by default
    )
)
```

## Store Ad Results

Create a single `OriviaMaxHelper` instance at class level:

```swift
private let oriviaMaxHelper = OriviaMaxHelper()
```

This helper will remember the last success/failure per ad type and feed it to Orivia SDK.

## Interstitial

### Hook Into MAX Interstitial Callbacks

Forward to the helper related events:

```swift
final class InterstitialDelegate: NSObject, MAAdDelegate, MAAdRevenueDelegate {

    private let helper: OriviaMaxHelper

    init(helper: OriviaMaxHelper) {
        self.helper = helper
    }

    func didLoad(_ ad: MAAd) {
        helper.onAdLoaded(ad)
        // Handle your own interstitial loaded logic
    }

    func didFailToLoadAd(forAdUnitIdentifier adUnitIdentifier: String, withError error: MAError) {
        helper.onAdLoadFailed(adType: .interstitial, adUnitId: adUnitIdentifier)
        // Handle your own interstitial failed logic
    }

    func didDisplay(_ ad: MAAd) {
        // Handle interstitial displayed
    }

    func didHide(_ ad: MAAd) {
        // Handle interstitial hidden
    }

    func didClick(_ ad: MAAd) {
        // Handle interstitial clicked
    }

    func didFail(toDisplay ad: MAAd, withError error: MAError) {
        // Handle interstitial display failed
    }

    func didPayRevenue(for ad: MAAd) {
        helper.onAdRevenuePaid(ad)
    }
}
```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.

    It's critical for the SDK to receive all events.

### Load & Show Interstitial

Use `OriviaMaxHelper` instance to pick the ad unit id, then call MAX:

```swift
private var interstitialAd: MAInterstitialAd?
private var interstitialDelegate: InterstitialDelegate?

func loadInterstitial() {
    let adUnitId = oriviaMaxHelper.getAdUnitId(for: .interstitial)
    let ad = MAInterstitialAd(adUnitIdentifier: adUnitId)

    // Disable MAX's auto-retry via extra parameter
    ad.setExtraParameterForKey("disable_auto_retries", value: "true")

    let delegate = InterstitialDelegate(helper: oriviaMaxHelper)
    ad.delegate = delegate
    ad.revenueDelegate = delegate

    interstitialDelegate = delegate
    interstitialAd = ad
    ad.load()
}

func showInterstitial() {
    if let ad = interstitialAd, ad.isReady {
        ad.show()
    }
}
```

`getAdUnitId(for:)` returns the AppLovin MAX ad unit ID to use for the ad request.

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Rewarded

### Hook Into MAX Rewarded Callbacks

Forward to the helper related events:

```swift
final class RewardedDelegate: NSObject, MARewardedAdDelegate, MAAdRevenueDelegate {

    private let helper: OriviaMaxHelper

    init(helper: OriviaMaxHelper) {
        self.helper = helper
    }

    func didLoad(_ ad: MAAd) {
        helper.onAdLoaded(ad)
        // Handle your own rewarded loaded logic
    }

    func didFailToLoadAd(forAdUnitIdentifier adUnitIdentifier: String, withError error: MAError) {
        helper.onAdLoadFailed(adType: .rewarded, adUnitId: adUnitIdentifier)
        // Handle your own rewarded failed logic
    }

    func didDisplay(_ ad: MAAd) {
        // Handle rewarded displayed
    }

    func didHide(_ ad: MAAd) {
        // Handle rewarded hidden
    }

    func didClick(_ ad: MAAd) {
        // Handle rewarded clicked
    }

    func didFail(toDisplay ad: MAAd, withError error: MAError) {
        // Handle rewarded display failed
    }

    func didRewardUser(for ad: MAAd, with reward: MAReward) {
        // Handle user rewarded
    }

    func didPayRevenue(for ad: MAAd) {
        helper.onAdRevenuePaid(ad)
    }
}
```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.

    It's critical for the SDK to receive all events.

### Load & Show Rewarded

Use `OriviaMaxHelper` instance to pick the ad unit id, then call MAX:

```swift
private var rewardedAd: MARewardedAd?
private var rewardedDelegate: RewardedDelegate?

func loadRewarded() {
    let adUnitId = oriviaMaxHelper.getAdUnitId(for: .rewarded)
    let ad = MARewardedAd.shared(withAdUnitIdentifier: adUnitId)

    // Disable MAX's auto-retry via extra parameter
    ad.setExtraParameterForKey("disable_auto_retries", value: "true")

    let delegate = RewardedDelegate(helper: oriviaMaxHelper)
    ad.delegate = delegate
    ad.revenueDelegate = delegate

    rewardedDelegate = delegate
    rewardedAd = ad
    ad.load()
}

func showRewarded() {
    if let ad = rewardedAd, ad.isReady {
        ad.show()
    }
}
```

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Extra Value

Returns a server-configured string value for the given ad type and key.

```swift
let value: String? = OriviaSdk.shared.getExtraValue(adType: .interstitial, key: "my_key")
```

## Features Data

`getFeaturesData()` provides server-configured feature data, including AppLovin MAX specific settings.

```swift
let featuresData = OriviaSdk.shared.getFeaturesData()
let adUnitNames: [String: String]? = featuresData.maxFeatureData?.adUnitNames
```

`maxFeatureData?.adUnitNames` is a map of MAX ad unit ID → ad unit name configured on the server.

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
