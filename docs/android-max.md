# Android Managed MAX

`OriviaMaxSdk` is a high-level wrapper over AppLovin MAX that manages the full ad lifecycle — loading, retry on failure — so you don't have to wire MAX callbacks yourself.

## Import Dependencies

Add the Orivia Maven repository, then the core SDK and MAX mediation module:

=== "Kotlin DSL"
    ```kotlin
    repositories {
        maven { url = uri("https://raw.githubusercontent.com/orivia-ai/orivia-maven/main") }
    }

    dependencies {
        implementation("ai.orivia:monetization:1.15.0")
        implementation("ai.orivia:mediation-max:1.15.0")
    }
    ```

=== "Groovy DSL"
    ```groovy
    repositories {
        maven { url "https://raw.githubusercontent.com/orivia-ai/orivia-maven/main" }
    }

    dependencies {
        implementation "ai.orivia:monetization:1.15.0"
        implementation "ai.orivia:mediation-max:1.15.0"
    }
    ```

## Setup

### 1. Define SDK Keys & Ad Unit IDs

Declare your MAX SDK key, Orivia publisher ID, and ad unit IDs:

=== "Kotlin"
    ```kotlin
    private const val MAX_SDK_KEY = "YOUR_MAX_SDK_KEY"
    private const val ORIVIA_PUBLISHER_ID = "YOUR_ORIVIA_PUBLISHER_ID"

    private const val INTERSTITIAL_AD_UNIT_ID = "YOUR_INTERSTITIAL_AD_UNIT_ID"
    private const val REWARDED_AD_UNIT_ID = "YOUR_REWARDED_AD_UNIT_ID"
    ```

=== "Java"
    ```java
    private static final String MAX_SDK_KEY = "YOUR_MAX_SDK_KEY";
    private static final String ORIVIA_PUBLISHER_ID = "YOUR_ORIVIA_PUBLISHER_ID";

    private static final String INTERSTITIAL_AD_UNIT_ID = "YOUR_INTERSTITIAL_AD_UNIT_ID";
    private static final String REWARDED_AD_UNIT_ID = "YOUR_REWARDED_AD_UNIT_ID";
    ```

### 2. Initialize

Get the `OriviaMaxSdk` singleton, subscribe to callbacks, then call `init`:

=== "Kotlin"
    ```kotlin
    private val oriviaMaxSdk: OriviaMaxSdk by lazy { OriviaMaxSdk.getInstance(context) }

    fun start() {
        // For debug purposes only.
        oriviaMaxSdk.setLoggingEnabled(true)

        oriviaMaxSdk.onInterstitialLoadedListener = OriviaMaxSdk.OnAdLoadedListener {
            // Ad is ready to show. isInterstitialReady() == true.
        }
        oriviaMaxSdk.onRewardedLoadedListener = OriviaMaxSdk.OnAdLoadedListener {
            // Ad is ready to show. isRewardedReady() == true.
        }

        oriviaMaxSdk.init(
            sdkKey = MAX_SDK_KEY,
            publisherId = ORIVIA_PUBLISHER_ID,
            defaultInterstitialAdUnitId = INTERSTITIAL_AD_UNIT_ID,
            defaultRewardedAdUnitId = REWARDED_AD_UNIT_ID,
            initListener = {
                // Both Orivia SDK and MAX are ready — safe to start loading
                oriviaMaxSdk.loadInterstitial()
                oriviaMaxSdk.loadRewarded()
            }
        )
    }
    ```

=== "Java"
    ```java
    private OriviaMaxSdk oriviaMaxSdk = OriviaMaxSdk.getInstance(context);

    void start() {
        // For debug purposes only.
        oriviaMaxSdk.setLoggingEnabled(true);

        oriviaMaxSdk.setOnInterstitialLoadedListener(() -> {
            // Ad is ready to show. isInterstitialReady() == true.
        });
        oriviaMaxSdk.setOnRewardedLoadedListener(() -> {
            // Ad is ready to show. isRewardedReady() == true.
        });

        oriviaMaxSdk.init(
            MAX_SDK_KEY,
            ORIVIA_PUBLISHER_ID,
            OriviaSdk.AD_UNIT_ID_EMPTY,
            INTERSTITIAL_AD_UNIT_ID,
            REWARDED_AD_UNIT_ID,
            false,
            () -> {
                // Both Orivia SDK and MAX are ready — safe to start loading
                oriviaMaxSdk.loadInterstitial();
                oriviaMaxSdk.loadRewarded();
            }
        );
    }
    ```

`init` initializes Orivia SDK first, fetches remote config, then initializes AppLovin MAX. Both must complete before ads can be loaded.

`init` also accepts a `defaultBannerAdUnitId`. `OriviaMaxSdk` does not load or show banners itself — manage banners directly via the raw MAX API — but passing it in lets the ad unit rotation logic account for banner placements too.

!!! warning "Important"
    You may register a raw MAX listener via `addInterstitialAdListener` / `addRewardedAdListener` directly for informational purposes (e.g., analytics), but all ad display operations must go through `OriviaMaxSdk`.

### 3. Handle Initialization Completion

```kotlin
initListener = {
    // Both Orivia SDK and MAX are ready — safe to start loading
    oriviaMaxSdk.loadInterstitial()
    oriviaMaxSdk.loadRewarded()
}
```

!!! info "Info"
    `initListener` is called after both Orivia SDK config and AppLovin MAX have initialized successfully. It is the recommended place to trigger the first ad load. It fires on success only — a failed `OriviaSdk` initialization is only logged internally.

## Loading Ads

```kotlin
oriviaMaxSdk.loadInterstitial()
oriviaMaxSdk.loadRewarded()
```

**Behavior:**

- If an ad is **already ready** — fires `onInterstitialLoadedListener` / `onRewardedLoadedListener` immediately and returns.
- If a load is **already in progress** — skips silently.

After a failed load, `OriviaMaxSdk` retries automatically. You do not need to schedule retries yourself.

## Callbacks

Subscribe before calling `init` to avoid missing the first event:

```kotlin
oriviaMaxSdk.onInterstitialLoadedListener = OriviaMaxSdk.OnAdLoadedListener {
    // Ad is ready to show. isInterstitialReady() == true.
}

oriviaMaxSdk.onRewardedLoadedListener = OriviaMaxSdk.OnAdLoadedListener {
    // Ad is ready to show. isRewardedReady() == true.
}
```

The callback fires only when the full load cycle is complete. It is safe to call `showInterstitial` / `showRewarded` directly from the callback.

### Observing Raw MAX Events

Android's AppLovin MAX SDK has no global callback bus (unlike Unity's static `MaxSdkCallbacks.*`) — `OriviaMaxSdk` owns the ad instance and its listener internally, so this is the only way to see raw ad events (display, hide, click, load failure, display failure, revenue) for your own analytics:

=== "Kotlin"
    ```kotlin
    // OriviaMaxSdk holds these via WeakReference, so keep a strong reference to them yourself
    // for as long as you want to keep observing events.
    private val interstitialAdListener: MaxAdListener = object : MaxAdListener {
        override fun onAdLoaded(ad: MaxAd) { /* ... */ }
        override fun onAdDisplayed(ad: MaxAd) { /* ... */ }
        override fun onAdHidden(ad: MaxAd) { /* ... */ }
        override fun onAdClicked(ad: MaxAd) { /* ... */ }
        override fun onAdLoadFailed(adUnitId: String, error: MaxError) { /* ... */ }
        override fun onAdDisplayFailed(ad: MaxAd, error: MaxError) { /* ... */ }
    }

    oriviaMaxSdk.addInterstitialAdListener(interstitialAdListener)
    oriviaMaxSdk.addInterstitialAdRevenueListener { ad -> /* ... */ }
    oriviaMaxSdk.addRewardedAdListener(rewardedAdListener)
    oriviaMaxSdk.addRewardedAdRevenueListener { ad -> /* ... */ }
    ```

=== "Java"
    ```java
    private MaxAdListener interstitialAdListener = new MaxAdListener() {
        @Override public void onAdLoaded(MaxAd ad) { /* ... */ }
        @Override public void onAdDisplayed(MaxAd ad) { /* ... */ }
        @Override public void onAdHidden(MaxAd ad) { /* ... */ }
        @Override public void onAdClicked(MaxAd ad) { /* ... */ }
        @Override public void onAdLoadFailed(String adUnitId, MaxError error) { /* ... */ }
        @Override public void onAdDisplayFailed(MaxAd ad, MaxError error) { /* ... */ }
    };

    oriviaMaxSdk.addInterstitialAdListener(interstitialAdListener);
    oriviaMaxSdk.addInterstitialAdRevenueListener(ad -> { /* ... */ });
    oriviaMaxSdk.addRewardedAdListener(rewardedAdListener);
    oriviaMaxSdk.addRewardedAdRevenueListener(ad -> { /* ... */ });
    ```

Unregister with the matching `remove*` method (e.g. `removeInterstitialAdListener`) once you no longer need the events — for example in `onDestroy`. Has no effect when `dataCollectionOnly = true`, since you already own the raw ad instance and set your own listener on it directly.

## Showing Ads

`placement` and `customData` are passed directly to AppLovin MAX and behave identically to `MaxInterstitialAd.showAd` / `MaxRewardedAd.showAd`.

=== "Kotlin"
    ```kotlin
    oriviaMaxSdk.showInterstitial()                          // no placement
    oriviaMaxSdk.showInterstitial("game_over")                // with placement
    oriviaMaxSdk.showInterstitial("level_end", "my_data")     // with placement + custom data

    oriviaMaxSdk.showRewarded()
    oriviaMaxSdk.showRewarded("extra_life")
    oriviaMaxSdk.showRewarded("extra_life", "my_data")        // with placement + custom data
    ```

=== "Java"
    ```java
    oriviaMaxSdk.showInterstitial();                          // no placement
    oriviaMaxSdk.showInterstitial("game_over");                // with placement
    oriviaMaxSdk.showInterstitial("level_end", "my_data");     // with placement + custom data

    oriviaMaxSdk.showRewarded();
    oriviaMaxSdk.showRewarded("extra_life");
    oriviaMaxSdk.showRewarded("extra_life", "my_data");        // with placement + custom data
    ```

!!! warning "Important"
    `show` logs a warning and returns early if no ad is ready. Always check `isInterstitialReady()` / `isRewardedReady()` before showing, or load first and show from the loaded callback.

## Checking Readiness

=== "Kotlin"
    ```kotlin
    if (oriviaMaxSdk.isInterstitialReady()) {
        oriviaMaxSdk.showInterstitial("placement")
    }

    if (oriviaMaxSdk.isRewardedReady()) {
        oriviaMaxSdk.showRewarded("placement")
    }
    ```

=== "Java"
    ```java
    if (oriviaMaxSdk.isInterstitialReady()) {
        oriviaMaxSdk.showInterstitial("placement");
    }

    if (oriviaMaxSdk.isRewardedReady()) {
        oriviaMaxSdk.showRewarded("placement");
    }
    ```

## Data Collection Only Mode

In this mode `OriviaMaxSdk` only tracks events — it does not create, load, or show any MAX ad instance itself. The publisher initializes AppLovin MAX, creates and owns the raw `MaxInterstitialAd` / `MaxRewardedAd` instances, and drives loading and display directly.

Pass `dataCollectionOnly = true` to `init`, and report every load/failure/revenue event back to Orivia by calling `onAdLoaded`, `onAdLoadFailed`, and `onAdRevenuePaid` from your own listener — the same contract `OriviaMaxHelper` already exposes, mirrored here for a single entry point:

=== "Kotlin"
    ```kotlin
    oriviaMaxSdk.init(
        sdkKey = MAX_SDK_KEY, // unused in this mode, but still required
        publisherId = ORIVIA_PUBLISHER_ID,
        defaultInterstitialAdUnitId = INTERSTITIAL_AD_UNIT_ID,
        defaultRewardedAdUnitId = REWARDED_AD_UNIT_ID,
        dataCollectionOnly = true,
        initListener = { /* Orivia config is ready — MAX init is not awaited */ }
    )

    // The publisher initializes MAX and loads/shows ads as usual:
    val interstitialAd = MaxInterstitialAd(INTERSTITIAL_AD_UNIT_ID)
    interstitialAd.setListener(object : MaxAdListener {
        override fun onAdLoaded(ad: MaxAd) {
            oriviaMaxSdk.onAdLoaded(ad)
        }

        override fun onAdLoadFailed(adUnitId: String, error: MaxError) {
            oriviaMaxSdk.onAdLoadFailed(AdType.INTERSTITIAL, adUnitId)
        }
        // ... other MaxAdListener callbacks
    })
    interstitialAd.setRevenueListener { ad -> oriviaMaxSdk.onAdRevenuePaid(ad) }
    interstitialAd.loadAd()
    ```

=== "Java"
    ```java
    oriviaMaxSdk.init(
        MAX_SDK_KEY, // unused in this mode, but still required
        ORIVIA_PUBLISHER_ID,
        OriviaSdk.AD_UNIT_ID_EMPTY,
        INTERSTITIAL_AD_UNIT_ID,
        REWARDED_AD_UNIT_ID,
        true,
        () -> { /* Orivia config is ready — MAX init is not awaited */ }
    );

    // The publisher initializes MAX and loads/shows ads as usual:
    MaxInterstitialAd interstitialAd = new MaxInterstitialAd(INTERSTITIAL_AD_UNIT_ID);
    interstitialAd.setListener(new MaxAdListener() {
        @Override
        public void onAdLoaded(MaxAd ad) {
            oriviaMaxSdk.onAdLoaded(ad, null);
        }

        @Override
        public void onAdLoadFailed(String adUnitId, MaxError error) {
            oriviaMaxSdk.onAdLoadFailed(AdType.INTERSTITIAL, adUnitId, null);
        }
        // ... other MaxAdListener callbacks
    });
    interstitialAd.setRevenueListener(ad -> oriviaMaxSdk.onAdRevenuePaid(ad, null));
    interstitialAd.loadAd();
    ```

`loadInterstitial`, `showInterstitial`, `loadRewarded`, and `showRewarded` are disabled in this mode and log a warning if called. `isInterstitialReady` / `isRewardedReady` always return `false` — track readiness yourself. `addInterstitialAdListener` and related registration methods have no effect in this mode, since you already own the raw ad instance and its listener.

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

If the parameters should be passed in the init request, call `setClientParams` before `OriviaMaxSdk.init`.

!!! warning "Important"
    Passing `null` to this method has no effect. To clear the parameters, pass an empty `ValueMap`.

## Logging

Turn on internal logs for debugging:

=== "Kotlin"
    ```kotlin
    oriviaMaxSdk.setLoggingEnabled(true)
    ```

=== "Java"
    ```java
    oriviaMaxSdk.setLoggingEnabled(true);
    ```

Logs appear in **Logcat**.

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

## Full Example

=== "Kotlin"
    ```kotlin
    class AdsActivity : AppCompatActivity() {

        private val oriviaMaxSdk: OriviaMaxSdk by lazy { OriviaMaxSdk.getInstance(this) }

        override fun onCreate(savedInstanceState: Bundle?) {
            super.onCreate(savedInstanceState)

            // For debug purposes only.
            oriviaMaxSdk.setLoggingEnabled(true)

            oriviaMaxSdk.onInterstitialLoadedListener = OriviaMaxSdk.OnAdLoadedListener {
                // Ad is ready — show it when appropriate
            }
            oriviaMaxSdk.onRewardedLoadedListener = OriviaMaxSdk.OnAdLoadedListener {
                // Ad is ready — show it when appropriate
            }

            oriviaMaxSdk.init(
                sdkKey = "YOUR_MAX_SDK_KEY",
                publisherId = "YOUR_PUBLISHER_ID",
                defaultInterstitialAdUnitId = "YOUR_INTERSTITIAL_AD_UNIT_ID",
                defaultRewardedAdUnitId = "YOUR_REWARDED_AD_UNIT_ID",
                initListener = {
                    oriviaMaxSdk.loadInterstitial()
                    oriviaMaxSdk.loadRewarded()
                }
            )
        }

        fun showInterstitial() {
            if (oriviaMaxSdk.isInterstitialReady()) {
                oriviaMaxSdk.showInterstitial("my_placement")
            }
        }

        fun showRewarded() {
            if (oriviaMaxSdk.isRewardedReady()) {
                oriviaMaxSdk.showRewarded("my_placement")
            }
        }
    }
    ```

=== "Java"
    ```java
    public class AdsActivity extends AppCompatActivity {

        private final OriviaMaxSdk oriviaMaxSdk = OriviaMaxSdk.getInstance(this);

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            // For debug purposes only.
            oriviaMaxSdk.setLoggingEnabled(true);

            oriviaMaxSdk.setOnInterstitialLoadedListener(() -> {
                // Ad is ready — show it when appropriate
            });
            oriviaMaxSdk.setOnRewardedLoadedListener(() -> {
                // Ad is ready — show it when appropriate
            });

            oriviaMaxSdk.init(
                "YOUR_MAX_SDK_KEY",
                "YOUR_PUBLISHER_ID",
                OriviaSdk.AD_UNIT_ID_EMPTY,
                "YOUR_INTERSTITIAL_AD_UNIT_ID",
                "YOUR_REWARDED_AD_UNIT_ID",
                false,
                () -> {
                    oriviaMaxSdk.loadInterstitial();
                    oriviaMaxSdk.loadRewarded();
                }
            );
        }

        void showInterstitial() {
            if (oriviaMaxSdk.isInterstitialReady()) {
                oriviaMaxSdk.showInterstitial("my_placement");
            }
        }

        void showRewarded() {
            if (oriviaMaxSdk.isRewardedReady()) {
                oriviaMaxSdk.showRewarded("my_placement");
            }
        }
    }
    ```
