# Orivia SDK Documentation

Orivia SDK provides ML-driven ad unit optimization for AppLovin MAX. The SDK dynamically selects the most effective ad unit for each impression based on user context and behavior.

## 📱 Supported Platforms

| Platform | Status | Documentation |
|----------|--------|---------------|
| **Unity** | ✅ Available | [Unity Integration](unity.md) |
| **Android** | ✅ Available | [Android Integration](android.md) |
| **iOS** | 🚧 Coming Soon | [iOS Integration](ios.md) |

## 🔧 Quick Integration

=== "Unity (C#)"
    ```csharp
    OriviaSdk.Init(
        publisherId: OriviaPublisherId,
        defaultBannerAdUnitId: BannerAdUnitId,
        defaultInterstitialAdUnitId: InterstitialAdUnitId,
        defaultRewardedAdUnitId: RewardedAdUnitId,
        dataCollectionOnly: false
    );
    ```

=== "Android (Kotlin)"
    ```kotlin
    OriviaSdk.getInstance(context).init(
        OriviaSdk.Config(
            publisherId = ORIVIA_PUBLISHER_ID,
            defaultBannerAdUnit = BANNER_AD_UNIT_ID,
            defaultInterstitialAdUnit = INTERSTITIAL_AD_UNIT_ID,
            defaultRewardedAdUnit = REWARDED_AD_UNIT_ID,
            dataCollectionOnly = false
        )
    )
    ```

## 🚀 How It Works

1. **Initialize MAX** - Set up your AppLovin MAX SDK
2. **Initialize Orivia** - Add the optimization layer
3. **Request Ad Units** - Use `OriviaMaxHelper` to get optimized ad unit IDs
4. **Load/Show Ads** - Continue using standard MAX methods

## 🎯 Key Features

- **Dynamic Ad Unit Selection** - Automatically chooses optimal ad units per user
- **Fallback Support** - Uses default ad unit IDs when optimization is unavailable
- **Revenue Tracking** - Integrates with MAX revenue callbacks
- **Client Parameters** - Support for custom user context data

## 📚 Getting Started

Choose your platform and follow the integration guide:

- [Unity Integration Guide](unity.md) - Complete Unity setup and implementation
- [Android Integration Guide](android.md) - Android/Kotlin/Java implementation
- [iOS Integration Guide](ios.md) - iOS implementation (coming soon)