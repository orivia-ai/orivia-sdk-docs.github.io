# Unity Managed MAX

`OriviaMaxSdk` is a high-level static wrapper over AppLovin MAX that manages the full ad lifecycle — loading, retry on failure — so you don't have to wire MAX callbacks yourself.

## Import Package

Make sure the Orivia SDK namespace is available in your scripts:

```csharp
using Orivia.Monetization;
```

## Setup

### 1. Define SDK Keys & Ad Unit IDs

Declare your MAX SDK key, Orivia publisher ID, and ad unit IDs:

```csharp
private const string MaxSdkKey = "YOUR_MAX_SDK_KEY";
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

Subscribe to callbacks, set the MAX SDK key, then call `Init`:

```csharp
void Start()
{
    // For debug purposes only.
    OriviaMaxSdk.SetLoggingEnabled(true);

    OriviaMaxSdk.OnInterstitialLoaded += OnInterstitialLoaded;
    OriviaMaxSdk.OnRewardedLoaded += OnRewardedLoaded;

    MaxSdk.SetSdkKey(MaxSdkKey);
    OriviaMaxSdk.Init(
        publisherId: OriviaPublisherId,
        defaultInterstitialAdUnitId: InterstitialAdUnitId,
        defaultRewardedAdUnitId: RewardedAdUnitId,
        initListener: new InitListener()
    );
}
```

`Init` initializes Orivia SDK first, fetches remote config, then initializes AppLovin MAX. Both must complete before ads can be loaded.

!!! warning "Important"
    You may subscribe to AppLovin MAX callbacks directly for informational purposes (e.g., analytics), but all ad display operations must go through `OriviaMaxSdk`.

### 3. Implement IInitListener

```csharp
private class InitListener : OriviaMaxSdk.IInitListener
{
    public void OnInitFinished()
    {
        // Both Orivia SDK and MAX are ready — safe to start loading
        OriviaMaxSdk.LoadInterstitial();
        OriviaMaxSdk.LoadRewarded();
    }
}
```

!!! info "Info"
    `OnInitFinished` is called after both Orivia SDK config and AppLovin MAX have initialized successfully. It is the recommended place to trigger the first ad load.

## Loading Ads

```csharp
OriviaMaxSdk.LoadInterstitial();
OriviaMaxSdk.LoadRewarded();
```

**Behavior:**

- If an ad is **already ready** — fires `OnInterstitialLoaded` / `OnRewardedLoaded` immediately and returns.
- If a load is **already in progress** — skips silently.

After a failed load, `OriviaMaxSdk` retries automatically. You do not need to schedule retries yourself.

## Callbacks

Subscribe before calling `Init` to avoid missing the first event:

```csharp
OriviaMaxSdk.OnInterstitialLoaded += () =>
{
    // Ad is ready to show. IsInterstitialReady == true.
};

OriviaMaxSdk.OnRewardedLoaded += () =>
{
    // Ad is ready to show. IsRewardedReady == true.
};
```

The callback fires only when the full load cycle is complete. It is safe to call `ShowInterstitial` / `ShowRewarded` directly from the callback.

## Showing Ads

`placement` and `customData` are passed directly to AppLovin MAX and behave identically to `MaxSdk.ShowInterstitial` / `MaxSdk.ShowRewardedAd`.

```csharp
OriviaMaxSdk.ShowInterstitial();                        // no placement
OriviaMaxSdk.ShowInterstitial("game_over");             // with placement
OriviaMaxSdk.ShowInterstitial("level_end", "my_data"); // with placement + custom data

OriviaMaxSdk.ShowRewarded();
OriviaMaxSdk.ShowRewarded("extra_life");
OriviaMaxSdk.ShowRewarded("extra_life", "my_data");    // with placement + custom data
```

!!! warning "Important"
    `Show` logs a warning and returns early if no ad is ready. Always check `IsInterstitialReady` / `IsRewardedReady` before showing, or load first and show from the `OnLoaded` callback.

## Checking Readiness

```csharp
if (OriviaMaxSdk.IsInterstitialReady)
    OriviaMaxSdk.ShowInterstitial("placement");

if (OriviaMaxSdk.IsRewardedReady)
    OriviaMaxSdk.ShowRewarded("placement");
```

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

If the parameters should be passed in the init request, call `SetClientParams` before `OriviaMaxSdk.Init`.

!!! warning "Important"
    Passing `null` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

```csharp
OriviaMaxSdk.SetLoggingEnabled(true);
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
using UnityEngine;

public class AdsManager : MonoBehaviour
{
    void Start()
    {
        // For debug purposes only.
        OriviaMaxSdk.SetLoggingEnabled(true);

        OriviaMaxSdk.OnInterstitialLoaded += OnInterstitialLoaded;
        OriviaMaxSdk.OnRewardedLoaded += OnRewardedLoaded;

        MaxSdk.SetSdkKey("YOUR_MAX_SDK_KEY");
        OriviaMaxSdk.Init(
            publisherId: "YOUR_PUBLISHER_ID",
            defaultInterstitialAdUnitId: "YOUR_INTERSTITIAL_AD_UNIT_ID",
            defaultRewardedAdUnitId: "YOUR_REWARDED_AD_UNIT_ID",
            initListener: new InitListener()
        );
    }

    private void OnInterstitialLoaded()
    {
        // Ad is ready — show it when appropriate
    }

    private void OnRewardedLoaded()
    {
        // Ad is ready — show it when appropriate
    }

    public void ShowInterstitial()
    {
        if (OriviaMaxSdk.IsInterstitialReady)
            OriviaMaxSdk.ShowInterstitial("my_placement");
    }

    public void ShowRewarded()
    {
        if (OriviaMaxSdk.IsRewardedReady)
            OriviaMaxSdk.ShowRewarded("my_placement");
    }

    private class InitListener : OriviaMaxSdk.IInitListener
    {
        public void OnInitFinished()
        {
            OriviaMaxSdk.LoadInterstitial();
            OriviaMaxSdk.LoadRewarded();
        }
    }
}
```
