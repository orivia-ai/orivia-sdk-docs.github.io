# Unity Integration

This guide shows you how to integrate the Orivia SDK with AppLovin MAX in Unity

## Import Package

Make sure the Orivia SDK namespace is available in your scripts:

```csharp
using Orivia.Monetization;
```

## Define SDK Keys & Ad Unit IDs

Declare your MAX SDK key, Orivia publisher ID, and ad unit IDs:

```csharp
private const string MaxSdkKey = "YOUR_MAX_SDK_KEY";
private const string OriviaPublisherId = "YOUR_ORIVIA_PUBLISHER_ID";

#if UNITY_IOS
    private const string InterstitialAdUnitId = "IOS_INTERSTITIAL_ID";
    private const string RewardedAdUnitId = "IOS_REWARDED_ID";
    private const string BannerAdUnitId = "IOS_BANNER_ID";
#else
    private const string InterstitialAdUnitId = "ANDROID_INTERSTITIAL_ID";
    private const string RewardedAdUnitId = "ANDROID_REWARDED_ID";
    private const string BannerAdUnitId = "ANDROID_BANNER_ID";
#endif
```

## Initialize MAX & Orivia

Init MAX:

```csharp
MaxSdk.SetSdkKey(MaxSdkKey);
MaxSdk.InitializeSdk();
```

Init Orivia: (any time after MAX init, but before requesting ads):

```csharp
OriviaSdk.Init(
    publisherId: OriviaPublisherId,
    defaultBannerAdUnitId: BannerAdUnitId,
    defaultInterstitialAdUnitId: InterstitialAdUnitId,
    defaultRewardedAdUnitId: RewardedAdUnitId,
    dataCollectionOnly: false // optional, false by default
    initListener: new SimpleInitListener() // optional
);

// ...

internal class SimpleInitListener : IInitListener
{
    public void OnInitCompleted() => Debug.Log("Orivia SDK initialized.");

    public void OnInitFailed(OriviaException ex) => Debug.LogError($"Orivia init failed: {ex.ErrorReason} — {ex.Message}");
}
```

!!! info "Info"
    Orivia is ready immediately after calling `Init`. You can load or show ads right away. If no config is loaded, default adUnits are used.

!!! warning "Important"
    `OnInitCompleted()` is triggered after the SDK loads its first config from the server.  
    To ensure optimal ads performance, it's **highly recommended** to wait for this callback **before** starting to load any ads.

!!! warning "Important"
    Calling `Init` multiple times **does not** reinitialize the SDK if it's already initialized or currently initializing.
    The method will only run again if the SDK has previously invoked the `OnInitFailed` callback.
    
    If `Init` is called after the SDK has been initialized, `OnInitCompleted()` is invoked immediately for provided listener.
    If `Init` is called while initialization is in progress, the newly provided listener will be notified as soon as initialization finishes, **together** with all previous listeners.

If a certain ad type isn't used, just pass the constant `OriviaSdk.AdUnitIdEmpty` instead of ad unit id.

For example, if the application doesn't have a key for a banner:

```csharp
OriviaSdk.Init(
    publisherId: OriviaPublisherId,
    defaultBannerAdUnitId: OriviaSdk.AdUnitIdEmpty,
    defaultInterstitialAdUnitId: InterstitialAdUnitId,
    defaultRewardedAdUnitId: RewardedAdUnitId,
    dataCollectionOnly: false // optional, false by default
    initListener: new SimpleInitListener() // optional
);
```

## Store Ad Results

Create a single `OriviaMaxHelper` instance at class level:

```csharp
private OriviaMaxHelper _oriviaMaxHelper = new OriviaMaxHelper();
```

This helper will remember the last success/failure per ad type and feed it to Orivia SDK.

## Interstitial

### Hook Into MAX Interstitial Callbacks

Forward to the helper related events:

```csharp
private void InitializeInterstitialAds()
{
    MaxSdkCallbacks.Interstitial.OnAdLoadedEvent += OnInterstitialLoadedEvent;
    MaxSdkCallbacks.Interstitial.OnAdLoadFailedEvent += OnInterstitialFailedEvent;
    MaxSdkCallbacks.Interstitial.OnAdRevenuePaidEvent += OnInterstitialRevenuePaidEvent;
}

// ...

private void OnInterstitialLoadedEvent(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    _oriviaMaxHelper.OnAdLoaded(adInfo);
    // ...
}

private void OnInterstitialFailedEvent(string adUnitId, MaxSdkBase.ErrorInfo errorInfo)
{
    _oriviaMaxHelper.OnAdLoadFailed(AdType.Interstitial, adUnitId);
    // ...
}

private void OnInterstitialRevenuePaidEvent(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    _oriviaMaxHelper.OnAdRevenuePaid(adInfo);
}
```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.
    
    It's critical for the SDK to receive all events.

### Load & Show Interstitial

Use `OriviaMaxHelper` instance to pick the ad unit, then call MAX:

```csharp
private OriviaMaxHelper.AdUnit? _currentInterstitialAdUnit;

void LoadInterstitial()
{
    // store returned ad unit for later use in show
    _currentInterstitialAdUnit = _oriviaMaxHelper.GetAdUnit(AdType.Interstitial);
    // disable MAX's auto-retry via extra parameter
    MaxSdk.SetInterstitialExtraParameter(_currentInterstitialAdUnit.MaxId, "disable_auto_retries", "true");
    MaxSdk.LoadInterstitial(_currentInterstitialAdUnit.MaxId);
}

void ShowInterstitial()
{
    if (_currentInterstitialAdUnit != null && MaxSdk.IsInterstitialReady(_currentInterstitialAdUnit.MaxId))
    {
        MaxSdk.ShowInterstitial(_currentInterstitialAdUnit.MaxId);
    }
}
```

`GetAdUnit` returns an `OriviaMaxHelper.AdUnit` object with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `OriviaId` | `string` | Internal Orivia ad unit identifier |
| `MaxId` | `string` | AppLovin MAX ad unit ID to use for ad requests |
| `PriceFloor` | `double` | Price floor for the ad request; `0` if not set |
| `Name` | `string` | Ad unit name; empty string if not available |

!!! warning "Important"
    `GetAdUnit` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Rewarded

### Hook Into MAX Rewarded Callbacks

Forward to the helper related events:

```csharp
private void InitializeRewardedAds()
{
    MaxSdkCallbacks.Rewarded.OnAdLoadedEvent += OnRewardedAdLoadedEvent;
    MaxSdkCallbacks.Rewarded.OnAdLoadFailedEvent += OnRewardedAdFailedEvent;
    MaxSdkCallbacks.Rewarded.OnAdRevenuePaidEvent += OnRewardedAdRevenuePaidEvent;
}

// ...

private void OnRewardedAdLoadedEvent(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    _oriviaMaxHelper.OnAdLoaded(adInfo);
    // ...
}

private void OnRewardedAdFailedEvent(string adUnitId, MaxSdkBase.ErrorInfo errorInfo)
{
    _oriviaMaxHelper.OnAdLoadFailed(AdType.Rewarded, adUnitId);
    // ...
}

private void OnRewardedAdRevenuePaidEvent(string adUnitId, MaxSdkBase.AdInfo adInfo)
{
    _oriviaMaxHelper.OnAdRevenuePaid(adInfo);
}
```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.
    
    It's critical for the SDK to receive all events.

### Load & Show Rewarded

Use `OriviaMaxHelper` instance to pick the ad unit, then call MAX:

```csharp
private OriviaMaxHelper.AdUnit? _currentRewardedAdUnit;

void LoadRewardedAd()
{
    // store returned ad unit for later use in show
    _currentRewardedAdUnit = _oriviaMaxHelper.GetAdUnit(AdType.Rewarded);
    // disable MAX's auto-retry via extra parameter
    MaxSdk.SetRewardedAdExtraParameter(_currentRewardedAdUnit.MaxId, "disable_auto_retries", "true");
    MaxSdk.LoadRewardedAd(_currentRewardedAdUnit.MaxId);
}

void ShowRewardedAd()
{
    if (_currentRewardedAdUnit != null && MaxSdk.IsRewardedAdReady(_currentRewardedAdUnit.MaxId))
    {
        MaxSdk.ShowRewardedAd(_currentRewardedAdUnit.MaxId);
    }
}
```

!!! warning "Important"
    `GetAdUnit` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Extra Value

Returns a server-configured string value for the given ad type and key.

```csharp
string? value = OriviaSdk.GetExtraValue(AdType.Interstitial, "my_key");
```

## Features Data

`GetFeaturesData()` provides server-configured feature data, including AppLovin MAX specific settings.

```csharp
var featuresData = OriviaSdk.GetFeaturesData();
Dictionary<string, string>? adUnitNames = featuresData.MaxFeatureData?.AdUnitNames;
```

`MaxFeatureData?.AdUnitNames` is a map of MAX ad unit ID → ad unit name configured on the server.

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

If the parameters should be passed in the init request, the `SetClientParams` method must be called before invoking the `Init` method.

!!! warning "Important"
    Passing `null` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

```csharp
OriviaSdk.SetLoggingEnabled(boolean);
```

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
