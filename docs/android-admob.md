# Android Integration (AdMob)

This guide shows you how to integrate the Orivia SDK with Google AdMob in Android

## Import Dependencies

Add the Orivia Maven repository, then the core SDK and AdMob mediation module:

=== "Kotlin DSL"
    ```kotlin
    repositories {
        maven { url = uri("https://raw.githubusercontent.com/orivia-ai/orivia-maven/main") }
    }

    dependencies {
        implementation("ai.orivia:monetization:1.18.0")
        implementation("ai.orivia:mediation-admob:1.18.0")
    }
    ```

=== "Groovy DSL"
    ```groovy
    repositories {
        maven { url "https://raw.githubusercontent.com/orivia-ai/orivia-maven/main" }
    }

    dependencies {
        implementation "ai.orivia:monetization:1.18.0"
        implementation "ai.orivia:mediation-admob:1.18.0"
    }
    ```

## Define Ad Unit IDs

Declare your Orivia publisher ID and ad unit IDs:

=== "Kotlin"
    ```kotlin
    private const val ORIVIA_PUBLISHER_ID = "YOUR_ORIVIA_PUBLISHER_ID"

    private const val INTERSTITIAL_AD_UNIT_ID = "YOUR_INTERSTITIAL_AD_UNIT_ID"
    private const val REWARDED_AD_UNIT_ID = "YOUR_REWARDED_AD_UNIT_ID"
    private const val BANNER_AD_UNIT_ID = "YOUR_BANNER_AD_UNIT_ID"
    ```

=== "Java"
    ```java
    private static final String ORIVIA_PUBLISHER_ID = "YOUR_ORIVIA_PUBLISHER_ID";

    private static final String INTERSTITIAL_AD_UNIT_ID = "YOUR_INTERSTITIAL_AD_UNIT_ID";
    private static final String REWARDED_AD_UNIT_ID = "YOUR_REWARDED_AD_UNIT_ID";
    private static final String BANNER_AD_UNIT_ID = "YOUR_BANNER_AD_UNIT_ID";
    ```

## Initialize AdMob & Orivia

Init AdMob:

=== "Kotlin"
    ```kotlin
    MobileAds.initialize(context) {
        // AdMob SDK initialized
    }
    ```

=== "Java"
    ```java
    MobileAds.initialize(context, initializationStatus -> {
        // AdMob SDK initialized
    });
    ```

Init Orivia with `Mediation.ADMOB` (any time after AdMob init, but before requesting ads):

=== "Kotlin"
    ```kotlin
    OriviaSdk.getInstance(context).init(
        OriviaSdk.Config(
            mediation = Mediation.ADMOB,
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
            Mediation.ADMOB,
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

!!! info "Info"
    Unlike MAX, AdMob has no runtime SDK key — its App ID is configured via `AndroidManifest.xml` meta-data instead.

!!! warning "Important"
    `onInitCompleted()` is triggered after the SDK loads its first config from the server.  
    To ensure optimal ads performance, it's **highly recommended** to wait for this callback **before** starting to load any ads.

!!! warning "Important"
    Calling `init` multiple times **does not** reinitialize the SDK if it's already initialized or currently initializing.
    The method will only run again if the SDK has previously invoked the `onInitFailed` callback.
    If `init` is called after the SDK has been initialized, `onInitCompleted()` is invoked immediately for provided listener.
    If `init` is called while initialization is in progress, the newly provided listener will be notified as soon as initialization finishes, **together** with all previous listeners.

If a certain ad type isn't used, just pass the constant `OriviaSdk.AD_UNIT_ID_EMPTY` instead of ad unit id.

## Store AdMob Ad Results

Create a single `OriviaAdmobHelper` instance at class level:

=== "Kotlin"
    ```kotlin
    private val oriviaAdmobHelper: OriviaAdmobHelper by lazy {
        OriviaAdmobHelper(context)
    }
    ```

=== "Java"
    ```java
    private OriviaAdmobHelper oriviaAdmobHelper = new OriviaAdmobHelper(context);
    ```

This helper will remember the last success/failure per ad type and feed it to Orivia SDK.

## Interstitial

### Load & Show Interstitial

Use `OriviaAdmobHelper` instance to pick the ad unit id, load via AdMob, then forward the load/revenue events back to the helper:

=== "Kotlin"
    ```kotlin
    private var interstitialAd: InterstitialAd? = null

    private fun loadInterstitial() {
        val adUnitId = oriviaAdmobHelper.getAdUnitId(AdType.INTERSTITIAL)
        InterstitialAd.load(
            context,
            adUnitId,
            AdRequest.Builder().build(),
            object : InterstitialAdLoadCallback() {
                override fun onAdLoaded(ad: InterstitialAd) {
                    ad.onPaidEventListener = OnPaidEventListener { adValue ->
                        oriviaAdmobHelper.onPaidEvent(AdType.INTERSTITIAL, adUnitId, adValue, ad.responseInfo)
                    }
                    ad.fullScreenContentCallback = object : FullScreenContentCallback() {
                        override fun onAdDismissedFullScreenContent() {
                            // Handle interstitial hidden
                        }

                        override fun onAdFailedToShowFullScreenContent(error: AdError) {
                            // Handle interstitial display failed
                        }
                    }
                    oriviaAdmobHelper.onAdLoaded(AdType.INTERSTITIAL, adUnitId, ad.responseInfo)
                    interstitialAd = ad
                    // Handle your own interstitial loaded logic
                }

                override fun onAdFailedToLoad(error: LoadAdError) {
                    oriviaAdmobHelper.onAdLoadFailed(AdType.INTERSTITIAL, adUnitId, error)
                    // Handle your own interstitial failed logic
                }
            }
        )
    }

    private fun showInterstitial() {
        interstitialAd?.show(activity)
    }
    ```

=== "Java"
    ```java
    private InterstitialAd interstitialAd;

    private void loadInterstitial() {
        String adUnitId = oriviaAdmobHelper.getAdUnitId(AdType.INTERSTITIAL);
        InterstitialAd.load(
            context,
            adUnitId,
            new AdRequest.Builder().build(),
            new InterstitialAdLoadCallback() {
                @Override
                public void onAdLoaded(InterstitialAd ad) {
                    ad.setOnPaidEventListener(adValue ->
                        oriviaAdmobHelper.onPaidEvent(AdType.INTERSTITIAL, adUnitId, adValue, ad.getResponseInfo())
                    );
                    ad.setFullScreenContentCallback(new FullScreenContentCallback() {
                        @Override
                        public void onAdDismissedFullScreenContent() {
                            // Handle interstitial hidden
                        }

                        @Override
                        public void onAdFailedToShowFullScreenContent(AdError error) {
                            // Handle interstitial display failed
                        }
                    });
                    oriviaAdmobHelper.onAdLoaded(AdType.INTERSTITIAL, adUnitId, ad.getResponseInfo());
                    interstitialAd = ad;
                    // Handle your own interstitial loaded logic
                }

                @Override
                public void onAdFailedToLoad(LoadAdError error) {
                    oriviaAdmobHelper.onAdLoadFailed(AdType.INTERSTITIAL, adUnitId, error);
                    // Handle your own interstitial failed logic
                }
            }
        );
    }

    private void showInterstitial() {
        if (interstitialAd != null) {
            interstitialAd.show(activity);
        }
    }
    ```

!!! warning "Important"
    You should call these callbacks in the same place in your code where you handle your own callbacks or MMP callbacks.

    It's critical for the SDK to receive all events.

!!! warning "Important"
    `getAdUnitId` should **NOT** be used when `dataCollectionOnly` is `true`, as the SDK does not update ad units in this mode — it will only return the values returned from init configuration call.

## Rewarded

### Load & Show Rewarded

Use `OriviaAdmobHelper` instance to pick the ad unit id, load via AdMob, then forward the load/revenue events back to the helper:

=== "Kotlin"
    ```kotlin
    private var rewardedAd: RewardedAd? = null

    private fun loadRewarded() {
        val adUnitId = oriviaAdmobHelper.getAdUnitId(AdType.REWARDED)
        RewardedAd.load(
            context,
            adUnitId,
            AdRequest.Builder().build(),
            object : RewardedAdLoadCallback() {
                override fun onAdLoaded(ad: RewardedAd) {
                    ad.onPaidEventListener = OnPaidEventListener { adValue ->
                        oriviaAdmobHelper.onPaidEvent(AdType.REWARDED, adUnitId, adValue, ad.responseInfo)
                    }
                    ad.fullScreenContentCallback = object : FullScreenContentCallback() {
                        override fun onAdDismissedFullScreenContent() {
                            // Handle rewarded hidden
                        }

                        override fun onAdFailedToShowFullScreenContent(error: AdError) {
                            // Handle rewarded display failed
                        }
                    }
                    oriviaAdmobHelper.onAdLoaded(AdType.REWARDED, adUnitId, ad.responseInfo)
                    rewardedAd = ad
                    // Handle your own rewarded loaded logic
                }

                override fun onAdFailedToLoad(error: LoadAdError) {
                    oriviaAdmobHelper.onAdLoadFailed(AdType.REWARDED, adUnitId, error)
                    // Handle your own rewarded failed logic
                }
            }
        )
    }

    private fun showRewarded() {
        rewardedAd?.show(activity) { rewardItem ->
            // Handle user rewarded
        }
    }
    ```

=== "Java"
    ```java
    private RewardedAd rewardedAd;

    private void loadRewarded() {
        String adUnitId = oriviaAdmobHelper.getAdUnitId(AdType.REWARDED);
        RewardedAd.load(
            context,
            adUnitId,
            new AdRequest.Builder().build(),
            new RewardedAdLoadCallback() {
                @Override
                public void onAdLoaded(RewardedAd ad) {
                    ad.setOnPaidEventListener(adValue ->
                        oriviaAdmobHelper.onPaidEvent(AdType.REWARDED, adUnitId, adValue, ad.getResponseInfo())
                    );
                    ad.setFullScreenContentCallback(new FullScreenContentCallback() {
                        @Override
                        public void onAdDismissedFullScreenContent() {
                            // Handle rewarded hidden
                        }

                        @Override
                        public void onAdFailedToShowFullScreenContent(AdError error) {
                            // Handle rewarded display failed
                        }
                    });
                    oriviaAdmobHelper.onAdLoaded(AdType.REWARDED, adUnitId, ad.getResponseInfo());
                    rewardedAd = ad;
                    // Handle your own rewarded loaded logic
                }

                @Override
                public void onAdFailedToLoad(LoadAdError error) {
                    oriviaAdmobHelper.onAdLoadFailed(AdType.REWARDED, adUnitId, error);
                    // Handle your own rewarded failed logic
                }
            }
        );
    }

    private void showRewarded() {
        if (rewardedAd != null) {
            rewardedAd.show(activity, rewardItem -> {
                // Handle user rewarded
            });
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

=== "Kotlin"
    ```kotlin
    val value: String? = OriviaSdk.getInstance(context).getExtraValue(AdType.INTERSTITIAL, "my_key")
    ```

=== "Java"
    ```java
    String value = OriviaSdk.getInstance(context).getExtraValue(AdType.INTERSTITIAL, "my_key");
    ```

## Ad Config Update Listener

Get notified whenever the server-driven ad config for a given ad type changes:

=== "Kotlin"
    ```kotlin
    val listener = object : OriviaSdk.AdConfigUpdateListener {
        override fun onAdConfigUpdated(adType: AdType) {
            // ad config for adType was refreshed
        }
    }
    OriviaSdk.getInstance(context).addAdConfigUpdateListener(AdType.INTERSTITIAL, listener)

    // Later, when no longer needed:
    OriviaSdk.getInstance(context).removeAdConfigUpdateListener(AdType.INTERSTITIAL, listener)
    ```

=== "Java"
    ```java
    OriviaSdk.AdConfigUpdateListener listener = adType -> {
        // ad config for adType was refreshed
    };
    OriviaSdk.getInstance(context).addAdConfigUpdateListener(AdType.INTERSTITIAL, listener);

    // Later, when no longer needed:
    OriviaSdk.getInstance(context).removeAdConfigUpdateListener(AdType.INTERSTITIAL, listener);
    ```

!!! warning "Important"
    The listener fires on every config refresh, not only when a value actually changed, and is held with a strong reference until removed — always call `removeAdConfigUpdateListener` (e.g. in `onDestroy`) to avoid leaking.

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
