# Android Integration

This guide shows you how to integrate the Orivia SDK with AppLovin MAX in Android

## Import Dependencies

Add the Orivia SDK to your project (currently, only manual addition of AAR artifacts is available).

## Define SDK Keys & Ad Unit IDs

Declare your MAX SDK key, Orivia publisher ID, and ad unit IDs:

=== "Kotlin"
    ```kotlin
    private const val MAX_SDK_KEY = "YOUR_MAX_SDK_KEY"
    private const val ORIVIA_PUBLISHER_ID = "YOUR_ORIVIA_PUBLISHER_ID"

    private const val INTERSTITIAL_AD_UNIT_ID = "YOUR_INTERSTITIAL_AD_UNIT_ID"
    private const val REWARDED_AD_UNIT_ID = "YOUR_REWARDED_AD_UNIT_ID"
    private const val BANNER_AD_UNIT_ID = "YOUR_BANNER_AD_UNIT_ID"
    ```

=== "Java"
    ```java
    private static final String MAX_SDK_KEY = "YOUR_MAX_SDK_KEY";
    private static final String ORIVIA_PUBLISHER_ID = "YOUR_ORIVIA_PUBLISHER_ID";

    private static final String INTERSTITIAL_AD_UNIT_ID = "YOUR_INTERSTITIAL_AD_UNIT_ID";
    private static final String REWARDED_AD_UNIT_ID = "YOUR_REWARDED_AD_UNIT_ID";
    private static final String BANNER_AD_UNIT_ID = "YOUR_BANNER_AD_UNIT_ID";
    ```

## Initialize MAX & Orivia

Init MAX:

=== "Kotlin"
    ```kotlin
    val initConfig = AppLovinSdkInitializationConfiguration.builder(MAX_SDK_KEY)
        .setMediationProvider(AppLovinMediationProvider.MAX)
        .build()

    AppLovinSdk.getInstance(context).initialize(initConfig) {
        // MAX SDK initialized
    }
    ```

=== "Java"
    ```java
    AppLovinSdkInitializationConfiguration initConfig = AppLovinSdkInitializationConfiguration.builder(MAX_SDK_KEY)
        .setMediationProvider(AppLovinMediationProvider.MAX)
        .build();

    AppLovinSdk.getInstance(context).initialize(initConfig, config -> {
        // MAX SDK initialized
    });
    ```

Init Orivia (any time after MAX init, but before requesting ads):

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).init(
        OriviaSdk.Config(
            publisherId = ORIVIA_PUBLISHER_ID,
            defaultBannerAdUnit = BANNER_AD_UNIT_ID,
            defaultInterstitialAdUnit = INTERSTITIAL_AD_UNIT_ID,
            defaultRewardedAdUnit = REWARDED_AD_UNIT_ID,
            dataCollectionOnly = false // optional, false by default
        ),
        initListener = object : OriviaSdk.InitListener {
            override fun onInitCompleted() {
                Log.d("OriviaSDK", "Orivia SDK initialized.")
            }

            override fun onInitFailed(exception: OriviaException) {
                Log.e("OriviaSDK", "Orivia init failed: ${exception.reason}")
            }
        }
    )
    ```

=== "Java"
    ```java
    OriviaSdk.getInstance(context).init(
        new OriviaSdk.Config(
            ORIVIA_PUBLISHER_ID,
            BANNER_AD_UNIT_ID,
            INTERSTITIAL_AD_UNIT_ID,
            REWARDED_AD_UNIT_ID,
            false // dataCollectionOnly
        ),
        new OriviaSdk.InitListener() {
            @Override
            public void onInitCompleted() {
                Log.d("OriviaSDK", "Orivia SDK initialized.");
            }

            @Override
            public void onInitFailed(@NonNull OriviaException exception) {
                Log.e("OriviaSDK", "Orivia init failed: " + exception.getReason());
            }
        }
    );
    ```

!!! info "Info"
    Orivia is ready immediately after calling `init`. You can load or show ads right away. If no config is loaded, default IDs are used.

!!! warning "Important"
    `onInitCompleted()` is triggered after the SDK loads its first config from the server.  
    To ensure optimal ads performance, it's **highly recommended** to wait for this callback **before** starting to load any ads.

!!! warning "Important"
    Calling `init` multiple times **does not** reinitialize the SDK if it's already initialized or currently initializing.
    The method will only run again if the SDK has previously invoked the `onInitFailed` callback.
    If `init` is called after the SDK has been initialized, `onInitCompleted()` is invoked immediately for provided listener.
    If `init` is called while initialization is in progress, the newly provided listener will be notified as soon as initialization finishes, **together** with all previous listeners.

If a certain ad type isn't used, just pass the constant `OriviaSdk.AD_UNIT_ID_EMPTY` instead of ad unit id.

For example, if the application doesn't have a key for a banner:

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).init(
        OriviaSdk.Config(
            publisherId = ORIVIA_PUBLISHER_ID,
            defaultBannerAdUnit = OriviaSdk.AD_UNIT_ID_EMPTY,
            defaultInterstitialAdUnit = INTERSTITIAL_AD_UNIT_ID,
            defaultRewardedAdUnit = REWARDED_AD_UNIT_ID,
            dataCollectionOnly = false
        )
    )
    ```

=== "Java"
    ```java
    OriviaSdk.getInstance(context).init(
        new OriviaSdk.Config(
            ORIVIA_PUBLISHER_ID,
            OriviaSdk.AD_UNIT_ID_EMPTY,
            INTERSTITIAL_AD_UNIT_ID,
            REWARDED_AD_UNIT_ID,
            false
        )
    );
    ```

## Store Ad Results

Create a single `OriviaMaxHelper` instance at class level:

=== "Kotlin"
    ```kotlin
    private val oriviaMaxHelper: OriviaMaxHelper by lazy {
        OriviaMaxHelper(context)
    }
    ```

=== "Java"
    ```java
    private OriviaMaxHelper oriviaMaxHelper = new OriviaMaxHelper(context);
    ```

This helper will remember the last success/failure per ad type and feed it to Orivia SDK.

## Interstitial

### Hook Into MAX Interstitial Callbacks

Forward to the helper related events:

=== "Kotlin"
    ```kotlin
    private fun setupInterstitialCallbacks(interstitialAd: MaxInterstitialAd) {
        interstitialAd.setListener(object : MaxAdListener {
            override fun onAdLoaded(maxAd: MaxAd) {
                oriviaMaxHelper.onAdLoaded(maxAd)
                // Handle your own interstitial loaded logic
            }

            override fun onAdLoadFailed(adUnitId: String, error: MaxError) {
                oriviaMaxHelper.onAdLoadFailed(AdType.INTERSTITIAL, adUnitId)
                // Handle your own interstitial failed logic
            }

            override fun onAdDisplayed(maxAd: MaxAd) {
                // Handle interstitial displayed
            }

            override fun onAdHidden(maxAd: MaxAd) {
                // Handle interstitial hidden
            }

            override fun onAdClicked(maxAd: MaxAd) {
                // Handle interstitial clicked
            }

            override fun onAdDisplayFailed(maxAd: MaxAd, error: MaxError) {
                // Handle interstitial display failed
            }
        })

        interstitialAd.setRevenueListener { maxAd ->
            oriviaMaxHelper.onAdRevenuePaid(maxAd)
        }
    }
    ```

=== "Java"
    ```java
    private void setupInterstitialCallbacks(MaxInterstitialAd interstitialAd) {
        interstitialAd.setListener(new MaxAdListener() {
            @Override
            public void onAdLoaded(MaxAd maxAd) {
                oriviaMaxHelper.onAdLoaded(maxAd);
                // Handle your own interstitial loaded logic
            }

            @Override
            public void onAdLoadFailed(String adUnitId, MaxError error) {
                oriviaMaxHelper.onAdLoadFailed(AdType.INTERSTITIAL, adUnitId);
                // Handle your own interstitial failed logic
            }

            @Override
            public void onAdDisplayed(MaxAd maxAd) {
                // Handle interstitial displayed
            }

            @Override
            public void onAdHidden(MaxAd maxAd) {
                // Handle interstitial hidden
            }

            @Override
            public void onAdClicked(MaxAd maxAd) {
                // Handle interstitial clicked
            }

            @Override
            public void onAdDisplayFailed(MaxAd maxAd, MaxError error) {
                // Handle interstitial display failed
            }
        });

        interstitialAd.setRevenueListener(maxAd -> oriviaMaxHelper.onAdRevenuePaid(maxAd));
    }
    ```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.
    
    It's critical for the SDK to receive all events.

### Load & Show Interstitial

Use `OriviaMaxHelper` instance to pick the ad unit id, then call MAX:

=== "Kotlin"
    ```kotlin
    private fun loadInterstitial() {
        val adUnitId = oriviaMaxHelper.getAdUnitId(AdType.INTERSTITIAL)
        val interstitialAd = MaxInterstitialAd(adUnitId)
        
        // Disable MAX's auto-retry via extra parameter
        interstitialAd.setExtraParameter("disable_auto_retries", "true")
        
        setupInterstitialCallbacks(interstitialAd)
        interstitialAd.loadAd()
    }

    private fun showInterstitial() {
        if (interstitialAd?.isReady == true) {
            interstitialAd.showAd(activity)
        }
    }
    ```

=== "Java"
    ```java
    private void loadInterstitial() {
        String adUnitId = oriviaMaxHelper.getAdUnitId(AdType.INTERSTITIAL);
        MaxInterstitialAd interstitialAd = new MaxInterstitialAd(adUnitId);
        
        // Disable MAX's auto-retry via extra parameter
        interstitialAd.setExtraParameter("disable_auto_retries", "true");
        
        setupInterstitialCallbacks(interstitialAd);
        interstitialAd.loadAd();
    }

    private void showInterstitial() {
        if (interstitialAd != null && interstitialAd.isReady()) {
            interstitialAd.showAd(activity);
        }
    }
    ```

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Rewarded

### Hook Into MAX Rewarded Callbacks

Forward to the helper related events:

=== "Kotlin"
    ```kotlin
    private fun setupRewardedCallbacks(rewardedAd: MaxRewardedAd) {
        rewardedAd.setListener(object : MaxRewardedAdListener {
            override fun onAdLoaded(maxAd: MaxAd) {
                oriviaMaxHelper.onAdLoaded(maxAd)
                // Handle your own rewarded loaded logic
            }

            override fun onAdLoadFailed(adUnitId: String, error: MaxError) {
                oriviaMaxHelper.onAdLoadFailed(AdType.REWARDED, adUnitId)
                // Handle your own rewarded failed logic
            }

            override fun onAdDisplayed(maxAd: MaxAd) {
                // Handle rewarded displayed
            }

            override fun onAdHidden(maxAd: MaxAd) {
                // Handle rewarded hidden
            }

            override fun onAdClicked(maxAd: MaxAd) {
                // Handle rewarded clicked
            }

            override fun onAdDisplayFailed(maxAd: MaxAd, error: MaxError) {
                // Handle rewarded display failed
            }

            override fun onUserRewarded(maxAd: MaxAd, maxReward: MaxReward) {
                // Handle user rewarded
            }
        })

        rewardedAd.setRevenueListener { maxAd ->
            oriviaMaxHelper.onAdRevenuePaid(maxAd)
        }
    }
    ```

=== "Java"
    ```java
    private void setupRewardedCallbacks(MaxRewardedAd rewardedAd) {
        rewardedAd.setListener(new MaxRewardedAdListener() {
            @Override
            public void onAdLoaded(MaxAd maxAd) {
                oriviaMaxHelper.onAdLoaded(maxAd);
                // Handle your own rewarded loaded logic
            }

            @Override
            public void onAdLoadFailed(String adUnitId, MaxError error) {
                oriviaMaxHelper.onAdLoadFailed(AdType.REWARDED, adUnitId);
                // Handle your own rewarded failed logic
            }

            @Override
            public void onAdDisplayed(MaxAd maxAd) {
                // Handle rewarded displayed
            }

            @Override
            public void onAdHidden(MaxAd maxAd) {
                // Handle rewarded hidden
            }

            @Override
            public void onAdClicked(MaxAd maxAd) {
                // Handle rewarded clicked
            }

            @Override
            public void onAdDisplayFailed(MaxAd maxAd, MaxError error) {
                // Handle rewarded display failed
            }

            @Override
            public void onUserRewarded(MaxAd maxAd, MaxReward maxReward) {
                // Handle user rewarded
            }
        });

        rewardedAd.setRevenueListener(maxAd -> oriviaMaxHelper.onAdRevenuePaid(maxAd));
    }
    ```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.
    
    It's critical for the SDK to receive all events.

### Load & Show Rewarded

Use `OriviaMaxHelper` instance to pick the ad unit id, then call MAX:

=== "Kotlin"
    ```kotlin
    private fun loadRewarded() {
        val adUnitId = oriviaMaxHelper.getAdUnitId(AdType.REWARDED)
        val rewardedAd = MaxRewardedAd.getInstance(adUnitId)
        
        // Disable MAX's auto-retry via extra parameter
        rewardedAd.setExtraParameter("disable_auto_retries", "true")
        
        setupRewardedCallbacks(rewardedAd)
        rewardedAd.loadAd()
    }

    private fun showRewarded() {
        if (rewardedAd?.isReady == true) {
            rewardedAd.showAd(activity)
        }
    }
    ```

=== "Java"
    ```java
    private void loadRewarded() {
        String adUnitId = oriviaMaxHelper.getAdUnitId(AdType.REWARDED);
        MaxRewardedAd rewardedAd = MaxRewardedAd.getInstance(adUnitId);
        
        // Disable MAX's auto-retry via extra parameter
        rewardedAd.setExtraParameter("disable_auto_retries", "true");
        
        setupRewardedCallbacks(rewardedAd);
        rewardedAd.loadAd();
    }

    private void showRewarded() {
        if (rewardedAd != null && rewardedAd.isReady()) {
            rewardedAd.showAd(activity);
        }
    }
    ```

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Client Parameters

To pass client parameters to the server:

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).setClientParams(
        valueMap {
            put("str", "value")
            put("int", 12)
            put("float", 1.3f)
            put("bool", true)
            put("nested", valueMap {
                put("str", "value")
            })
        }
    )
    ```

=== "Java"
    ```java
    OriviaSdk.getInstance(context).setClientParams(
        new ValueMap.Builder()
            .put("str", "value")
            .put("int", 12)
            .put("float", 1.3f)
            .put("bool", true)
            .put("nested", new ValueMap.Builder()
                .put("str", "value")
                .build()
            ).build()
    );
    ```

If the parameters should be passed in the init request, the `setClientParams` method must be called before invoking the `init` method.

!!! warning "Important"
    Passing `null` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).setLoggingEnabled(true)
    ```

=== "Java"
    ```java
    OriviaSdk.getInstance(context).setLoggingEnabled(true);
    ```

## Privacy

To pass GDPR Applies there are 2 options.

Pass it through CMP [IABTCF_gdprApplies](https://github.com/InteractiveAdvertisingBureau/GDPR-Transparency-and-Consent-Framework/blob/aa079f33b574bf5fe48594719cbe7e99d848bcd4/TCFv2/IAB%20Tech%20Lab%20-%20CMP%20API%20v2.md#in-app-details) property or using SDK method:

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).setGdprApplies(true) // or false, or null for unknown
    ```

=== "Java"
    ```java
    OriviaSdk.getInstance(context).setGdprApplies(true); // or false, or null for unknown
    ```

To pass COPPA:

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).setCoppa(true) // or false, or null for unknown
    ```

=== "Java"
    ```java
    OriviaSdk.getInstance(context).setCoppa(true); // or false, or null for unknown
    ```

Null values reset properties.
