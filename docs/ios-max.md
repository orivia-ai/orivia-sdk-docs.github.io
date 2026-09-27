# iOS Managed MAX

`OriviaMaxSdk` is a high-level wrapper over AppLovin MAX that manages the full ad lifecycle — loading, retry on failure — so you don't have to wire MAX callbacks yourself.

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

Make sure the Orivia SDK MAX mediation module is available in your files:

```swift
import OriviaMonetization
import OriviaMonetizationMax
```

## Setup

### 1. Define SDK Keys & Ad Unit IDs

Declare your MAX SDK key, Orivia publisher ID, and ad unit IDs:

```swift
private let maxSdkKey = "YOUR_MAX_SDK_KEY"
private let oriviaPublisherId = "YOUR_ORIVIA_PUBLISHER_ID"

private let interstitialAdUnitId = "YOUR_INTERSTITIAL_AD_UNIT_ID"
private let rewardedAdUnitId = "YOUR_REWARDED_AD_UNIT_ID"
```

### 2. Initialize

Subscribe to callbacks, then call `initialize`:

```swift
func start() {
    // For debug purposes only.
    OriviaMaxSdk.shared.setLoggingEnabled(true)

    OriviaMaxSdk.shared.onInterstitialLoaded = {
        // Ad is ready to show. isInterstitialReady() == true.
    }
    OriviaMaxSdk.shared.onRewardedLoaded = {
        // Ad is ready to show. isRewardedReady() == true.
    }

    OriviaMaxSdk.shared.initialize(
        sdkKey: maxSdkKey,
        publisherId: oriviaPublisherId,
        defaultInterstitialAdUnitId: interstitialAdUnitId,
        defaultRewardedAdUnitId: rewardedAdUnitId
    ) {
        // Both Orivia SDK and MAX are ready — safe to start loading
        OriviaMaxSdk.shared.loadInterstitial()
        OriviaMaxSdk.shared.loadRewarded()
    }
}
```

`initialize` initializes Orivia SDK first, fetches remote config, then initializes AppLovin MAX. Both must complete before ads can be loaded.

`initialize` also accepts a `defaultBannerAdUnitId`. `OriviaMaxSdk` does not load or show banners itself — manage banners directly via the raw MAX API — but passing it in lets the ad unit rotation logic account for banner placements too.

!!! warning "Important"
    You may register a raw MAX delegate via `addInterstitialAdDelegate` / `addRewardedAdDelegate` directly for informational purposes (e.g., analytics), but all ad display operations must go through `OriviaMaxSdk`.

### 3. Handle Initialization Completion

```swift
OriviaMaxSdk.shared.initialize(
    sdkKey: maxSdkKey,
    publisherId: oriviaPublisherId,
    defaultInterstitialAdUnitId: interstitialAdUnitId,
    defaultRewardedAdUnitId: rewardedAdUnitId
) {
    // Both Orivia SDK and MAX are ready — safe to start loading
    OriviaMaxSdk.shared.loadInterstitial()
    OriviaMaxSdk.shared.loadRewarded()
}
```

!!! info "Info"
    The `completion` closure is called after both Orivia SDK config and AppLovin MAX have initialized successfully. It is the recommended place to trigger the first ad load. It fires on success only — a failed `OriviaSdk` initialization is only logged internally.

## Loading Ads

```swift
OriviaMaxSdk.shared.loadInterstitial()
OriviaMaxSdk.shared.loadRewarded()
```

**Behavior:**

- If an ad is **already ready** — fires `onInterstitialLoaded` / `onRewardedLoaded` immediately and returns.
- If a load is **already in progress** — skips silently.

After a failed load, `OriviaMaxSdk` retries automatically. You do not need to schedule retries yourself.

## Callbacks

Subscribe before calling `initialize` to avoid missing the first event:

```swift
OriviaMaxSdk.shared.onInterstitialLoaded = {
    // Ad is ready to show. isInterstitialReady() == true.
}

OriviaMaxSdk.shared.onRewardedLoaded = {
    // Ad is ready to show. isRewardedReady() == true.
}
```

The callback fires only when the full load cycle is complete. It is safe to call `showInterstitial` / `showRewarded` directly from the callback.

### Observing Raw MAX Events

iOS's AppLovin MAX SDK has no global callback bus (unlike Unity's static `MaxSdkCallbacks.*`) — `OriviaMaxSdk` owns the ad instance and its delegate internally, so this is the only way to see raw ad events (display, hide, click, load failure, display failure, revenue) for your own analytics:

```swift
final class AdEventsObserver: NSObject, MAAdDelegate, MAAdRevenueDelegate, MARewardedAdDelegate {
    func didLoad(_ ad: MAAd) { /* ... */ }
    func didFailToLoadAd(forAdUnitIdentifier adUnitIdentifier: String, withError error: MAError) { /* ... */ }
    func didDisplay(_ ad: MAAd) { /* ... */ }
    func didHide(_ ad: MAAd) { /* ... */ }
    func didClick(_ ad: MAAd) { /* ... */ }
    func didFail(toDisplay ad: MAAd, withError error: MAError) { /* ... */ }
    func didRewardUser(for ad: MAAd, with reward: MAReward) { /* ... */ }
    func didPayRevenue(for ad: MAAd) { /* ... */ }
}

// OriviaMaxSdk holds the delegate weakly, so keep a strong reference to it yourself
// for as long as you want to keep observing events.
let observer = AdEventsObserver()
OriviaMaxSdk.shared.addInterstitialAdDelegate(observer)
OriviaMaxSdk.shared.addInterstitialAdRevenueDelegate(observer)
OriviaMaxSdk.shared.addRewardedAdDelegate(observer)
OriviaMaxSdk.shared.addRewardedAdRevenueDelegate(observer)
```

Unregister with the matching `remove*` method (e.g. `removeInterstitialAdDelegate`) once you no longer need the events — for example in `deinit`. Has no effect when `dataCollectionOnly == true`, since you already own the raw ad instance and set your own delegate on it directly.

## Showing Ads

`placement` and `customData` are passed directly to AppLovin MAX and behave identically to `MAInterstitialAd.show` / `MARewardedAd.show`.

```swift
OriviaMaxSdk.shared.showInterstitial()                          // no placement
OriviaMaxSdk.shared.showInterstitial(placement: "game_over")    // with placement
OriviaMaxSdk.shared.showInterstitial(placement: "level_end", customData: "my_data") // with placement + custom data

OriviaMaxSdk.shared.showRewarded()
OriviaMaxSdk.shared.showRewarded(placement: "extra_life")
OriviaMaxSdk.shared.showRewarded(placement: "extra_life", customData: "my_data") // with placement + custom data
```

!!! warning "Important"
    `show` logs a warning and returns early if no ad is ready. Always check `isInterstitialReady()` / `isRewardedReady()` before showing, or load first and show from the loaded callback.

## Checking Readiness

```swift
if OriviaMaxSdk.shared.isInterstitialReady() {
    OriviaMaxSdk.shared.showInterstitial(placement: "placement")
}

if OriviaMaxSdk.shared.isRewardedReady() {
    OriviaMaxSdk.shared.showRewarded(placement: "placement")
}
```

## Data Collection Only Mode

In this mode `OriviaMaxSdk` only tracks events — it does not create, load, or show any MAX ad instance itself. The publisher initializes AppLovin MAX, creates and owns the raw `MAInterstitialAd` / `MARewardedAd` instances, and drives loading and display directly.

Pass `dataCollectionOnly: true` to `initialize`, and report every load/failure/revenue event back to Orivia by calling `onAdLoaded`, `onAdLoadFailed`, and `onAdRevenuePaid` from your own delegate — the same contract `OriviaMaxHelper` already exposes, mirrored here for a single entry point:

```swift
OriviaMaxSdk.shared.initialize(
    sdkKey: maxSdkKey, // unused in this mode, but still required
    publisherId: oriviaPublisherId,
    defaultInterstitialAdUnitId: interstitialAdUnitId,
    defaultRewardedAdUnitId: rewardedAdUnitId,
    dataCollectionOnly: true
) {
    // Orivia config is ready — MAX init is not awaited
}

// The publisher initializes MAX and loads/shows ads as usual:
final class InterstitialDelegate: NSObject, MAAdDelegate, MAAdRevenueDelegate {
    func didLoad(_ ad: MAAd) {
        OriviaMaxSdk.shared.onAdLoaded(ad)
    }

    func didFailToLoadAd(forAdUnitIdentifier adUnitIdentifier: String, withError error: MAError) {
        OriviaMaxSdk.shared.onAdLoadFailed(adType: .interstitial, adUnitId: adUnitIdentifier)
    }

    func didPayRevenue(for ad: MAAd) {
        OriviaMaxSdk.shared.onAdRevenuePaid(ad)
    }

    // ... other MAAdDelegate callbacks
}

let interstitialAd = MAInterstitialAd(adUnitIdentifier: interstitialAdUnitId)
let delegate = InterstitialDelegate()
interstitialAd.delegate = delegate
interstitialAd.revenueDelegate = delegate
interstitialAd.load()
```

`loadInterstitial`, `showInterstitial`, `loadRewarded`, and `showRewarded` are disabled in this mode and log a warning if called. `isInterstitialReady` / `isRewardedReady` always return `false` — track readiness yourself. `addInterstitialAdDelegate` and related registration methods have no effect in this mode, since you already own the raw ad instance and its delegate.

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

If the parameters should be passed in the init request, call `setClientParams` before `OriviaMaxSdk.initialize`.

!!! warning "Important"
    Passing `nil` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

```swift
OriviaMaxSdk.shared.setLoggingEnabled(true)
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

## Full Example

```swift
import OriviaMonetization
import OriviaMonetizationMax
import AppLovinSDK

final class AdsManager: NSObject, ObservableObject {

    override init() {
        super.init()

        // For debug purposes only.
        OriviaMaxSdk.shared.setLoggingEnabled(true)

        OriviaMaxSdk.shared.onInterstitialLoaded = {
            // Ad is ready — show it when appropriate
        }
        OriviaMaxSdk.shared.onRewardedLoaded = {
            // Ad is ready — show it when appropriate
        }

        OriviaMaxSdk.shared.initialize(
            sdkKey: "YOUR_MAX_SDK_KEY",
            publisherId: "YOUR_PUBLISHER_ID",
            defaultInterstitialAdUnitId: "YOUR_INTERSTITIAL_AD_UNIT_ID",
            defaultRewardedAdUnitId: "YOUR_REWARDED_AD_UNIT_ID"
        ) {
            OriviaMaxSdk.shared.loadInterstitial()
            OriviaMaxSdk.shared.loadRewarded()
        }
    }

    func showInterstitial() {
        if OriviaMaxSdk.shared.isInterstitialReady() {
            OriviaMaxSdk.shared.showInterstitial(placement: "my_placement")
        }
    }

    func showRewarded() {
        if OriviaMaxSdk.shared.isRewardedReady() {
            OriviaMaxSdk.shared.showRewarded(placement: "my_placement")
        }
    }
}
```
