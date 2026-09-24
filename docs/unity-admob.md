# Unity Managed AdMob

`OriviaAdmobSdk` is a high-level static wrapper over Google Mobile Ads (AdMob) that manages the full ad lifecycle — loading, retry on failure, after-fill loading — so you don't have to wire AdMob callbacks yourself.

## Import Package

Make sure the Orivia SDK namespace is available in your scripts:

```csharp
using Orivia.Monetization;
using GoogleMobileAds.Api; // for InterstitialAd / RewardedAd / Reward types
```

## Setup

### 1. Define Ad Unit IDs

Declare your Orivia publisher ID and ad unit IDs:

```csharp
private const string OriviaPublisherId = "YOUR_ORIVIA_PUBLISHER_ID";

#if UNITY_IOS
    private const string InterstitialAdUnitId = "IOS_INTERSTITIAL_ID";
    private const string RewardedAdUnitId = "IOS_REWARDED_ID";
#else
    private const string InterstitialAdUnitId = "ANDROID_INTERSTITIAL_ID";
    private const string RewardedAdUnitId = "ANDROID_REWARDED_ID";
#endif
```

### 2. Initialize

Subscribe to callbacks, then call `Init`:

```csharp
void Start()
{
    // For debug purposes only.
    OriviaAdmobSdk.SetLoggingEnabled(true);

    OriviaAdmobSdk.OnInterstitialLoaded += OnInterstitialLoaded;
    OriviaAdmobSdk.OnRewardedLoaded += OnRewardedLoaded;

    OriviaAdmobSdk.Init(
        publisherId: OriviaPublisherId,
        defaultInterstitialAdUnitId: InterstitialAdUnitId,
        defaultRewardedAdUnitId: RewardedAdUnitId,
        initListener: new InitListener()
    );
}
```

`Init` initializes Orivia SDK first, fetches remote config, then initializes Google Mobile Ads. Both must complete before ads can be loaded.

`Init` also accepts a `defaultBannerAdUnitId`. `OriviaAdmobSdk` does not load or show banners itself — manage banners directly via the Google Mobile Ads `BannerView` API — but passing it in lets the ad unit rotation logic account for banner placements too.

### 3. Implement IInitListener

```csharp
private class InitListener : OriviaAdmobSdk.IInitListener
{
    public void OnInitFinished()
    {
        // Both Orivia SDK and Google Mobile Ads are ready — safe to start loading
        OriviaAdmobSdk.LoadInterstitial();
        OriviaAdmobSdk.LoadRewarded();
    }
}
```

!!! info "Info"
    `OnInitFinished` is called after both Orivia SDK config and Google Mobile Ads have initialized successfully. It is the recommended place to trigger the first ad load.

## Loading Ads

```csharp
OriviaAdmobSdk.LoadInterstitial();
OriviaAdmobSdk.LoadRewarded();
```

**Behavior:**

- If an ad is **already ready** — fires `OnInterstitialLoaded` / `OnRewardedLoaded` immediately and returns.
- If a load is **already in progress** — skips silently.
- After a successful load, an **after-fill** load may start in parallel against a better-priced ad unit; if it wins in time, `Show*` prefers it over the original.

After a failed load, `OriviaAdmobSdk` retries automatically with backoff. You do not need to schedule retries yourself.

## Callbacks

Subscribe before calling `Init` to avoid missing the first event:

```csharp
OriviaAdmobSdk.OnInterstitialLoaded += (InterstitialAd ad) =>
{
    // Ad is ready to show. IsInterstitialReady == true.
    // Attach full-screen content callbacks on `ad` here if needed.
};

OriviaAdmobSdk.OnRewardedLoaded += (RewardedAd ad) =>
{
    // Ad is ready to show. IsRewardedReady == true.
};
```

Both callbacks pass the loaded `InterstitialAd` / `RewardedAd` instance so you can attach AdMob's full-screen content callbacks before showing. The callback fires only when the full load cycle is complete. It is safe to call `ShowInterstitial` / `ShowRewarded` directly from the callback.

## Showing Ads

```csharp
OriviaAdmobSdk.ShowInterstitial();

OriviaAdmobSdk.ShowRewarded();
OriviaAdmobSdk.ShowRewarded(reward =>
{
    // User earned a reward.
});
```

!!! warning "Important"
    `Show` logs a warning and returns early if no ad is ready. Always check `IsInterstitialReady` / `IsRewardedReady` before showing, or load first and show from the `OnLoaded` callback.

## Checking Readiness

```csharp
if (OriviaAdmobSdk.IsInterstitialReady)
    OriviaAdmobSdk.ShowInterstitial();

if (OriviaAdmobSdk.IsRewardedReady)
    OriviaAdmobSdk.ShowRewarded();
```

`IsInterstitialReady` / `IsRewardedReady` return `false` while the primary or after-fill ad is still loading.

## Data Collection Only Mode

In this mode `OriviaAdmobSdk` only initializes Orivia SDK for config/tracking — it does **not** initialize Google Mobile Ads, and it does not subscribe to any AdMob callbacks on your behalf.

Pass `dataCollectionOnly: true` to `Init`:

```csharp
OriviaAdmobSdk.Init(
    publisherId: OriviaPublisherId,
    defaultInterstitialAdUnitId: InterstitialAdUnitId,
    defaultRewardedAdUnitId: RewardedAdUnitId,
    dataCollectionOnly: true,
    initListener: new InitListener()
);
```

`OnInitFinished` fires as soon as the Orivia SDK config is ready — Google Mobile Ads initialization is not awaited.

!!! warning "Important"
    The publisher drives `MobileAds.Initialize` and all ad loading/showing directly. To keep tracking data, report events manually via `OriviaSdk.TrackAdLoaded`, `OriviaSdk.TrackAdImpression`, and `OriviaSdk.TrackAdLoadFailed`.

`OriviaAdmobSdk.LoadInterstitial`, `ShowInterstitial`, `LoadRewarded`, and `ShowRewarded` are disabled in this mode and log a warning if called.

## Client Parameters

To pass client parameters to the server:

```csharp
var nested = new ValueMap.Builder().Put("nested", "value").Build();
var clientParams = new ValueMap.Builder()
    .Put("str", "value")
    .Put("int", UnityEngine.Random.Range(1, 1000))
    .Put("float", 3.43)
    .Put("bool", true)
    .Put("nested", nested)
    .Build();
OriviaSdk.SetClientParams(clientParams);
```

If the parameters should be passed in the init request, call `SetClientParams` before `OriviaAdmobSdk.Init`.

!!! warning "Important"
    Passing `null` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

```csharp
OriviaAdmobSdk.SetLoggingEnabled(true);
```

Logs appear in **logcat** (Android) or **Xcode Console** (iOS).

## Privacy

To pass GDPR Applies there are 2 options.

Pass it through CMP [IABTCF_gdprApplies](https://github.com/InteractiveAdvertisingBureau/GDPR-Transparency-and-Consent-Framework/blob/aa079f33b574bf5fe48594719cbe7e99d848bcd4/TCFv2/IAB%20Tech%20Lab%20-%20CMP%20API%20v2.md#in-app-details) property or using SDK method:

```csharp
OriviaSdk.SetGdprApplies(boolean?)
```

To pass COPPA:

```csharp
OriviaSdk.SetCoppa(boolean?)
```

Null values reset properties.

## Full Example

```csharp
using Orivia.Monetization;
using GoogleMobileAds.Api;
using UnityEngine;

public class AdsManager : MonoBehaviour
{
    void Start()
    {
        // For debug purposes only.
        OriviaAdmobSdk.SetLoggingEnabled(true);

        OriviaAdmobSdk.OnInterstitialLoaded += OnInterstitialLoaded;
        OriviaAdmobSdk.OnRewardedLoaded += OnRewardedLoaded;

        OriviaAdmobSdk.Init(
            publisherId: "YOUR_PUBLISHER_ID",
            defaultInterstitialAdUnitId: "YOUR_INTERSTITIAL_AD_UNIT_ID",
            defaultRewardedAdUnitId: "YOUR_REWARDED_AD_UNIT_ID",
            initListener: new InitListener()
        );
    }

    private void OnInterstitialLoaded(InterstitialAd ad)
    {
        // Ad is ready — show it when appropriate
    }

    private void OnRewardedLoaded(RewardedAd ad)
    {
        // Ad is ready — show it when appropriate
    }

    public void ShowInterstitial()
    {
        if (OriviaAdmobSdk.IsInterstitialReady)
            OriviaAdmobSdk.ShowInterstitial();
    }

    public void ShowRewarded()
    {
        if (OriviaAdmobSdk.IsRewardedReady)
        {
            OriviaAdmobSdk.ShowRewarded(reward =>
            {
                Debug.Log($"User earned reward — {reward.Type} x{reward.Amount}");
            });
        }
    }

    private class InitListener : OriviaAdmobSdk.IInitListener
    {
        public void OnInitFinished()
        {
            OriviaAdmobSdk.LoadInterstitial();
            OriviaAdmobSdk.LoadRewarded();
        }
    }
}
```
