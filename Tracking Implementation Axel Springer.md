# Bild App — Mixpanel Tracking Implementation Guide

This document covers the full implementation spec for all new and extended Mixpanel events across the Bild iOS and Android apps. It includes SDK code, server-side patterns, AppsFlyer integration, UTM capture, and links to relevant Mixpanel documentation.

---

## Contents

- [Core principles](#core-principles)
- [Identity management](#identity-management)
- [Super properties](#super-properties)
- [UTM capture — how it works with AppsFlyer](#utm-capture)
- [New events](#new-events)
  - [mp_stage_viewed](#mp_stage_viewed)
  - [mp_push_notification_sent](#mp_push_notification_sent)
  - [mp_push_notification_opened](#mp_push_notification_opened)
  - [mp_registration_completed](#mp_registration_completed)
  - [mp_login_completed](#mp_login_completed)
  - [mp_logout_completed](#mp_logout_completed)
  - [mp_subscription_cancelled](#mp_subscription_cancelled)
  - [mp_home_item_tapped](#mp_home_item_tapped)
  - [mp_heywebview_viewed](#mp_heywebview_viewed)
- [Extended existing events](#extended-existing-events)
  - [mp_app_open](#mp_app_open)
  - [mp_article_view](#mp_article_view)
  - [mp_article_closed](#mp_article_closed)
  - [mp_subscription_started](#mp_subscription_started)
  - [mp_paywall_viewed](#mp_paywall_viewed)
- [Property types reference](#property-types-reference)
- [Mixpanel documentation](#mixpanel-documentation)

---

## Core principles

### Property types
Mixpanel supports the following types. Always use the correct type — using a String where a Number is expected will break aggregations and Revenue Analytics.

| Type | Use case | Example |
|---|---|---|
| `String` | IDs, names, categories, status values | `"bild_news"`, `"free"` |
| `Int` | Whole numbers — positions, durations, depths | `3`, `120`, `75` |
| `Double` | Decimal numbers — revenue amounts | `7.99`, `79.98` |
| `Boolean` | Binary states | `true`, `false` |

### Event naming
All Bild events use the `mp_` prefix. Use consistent snake_case property names across iOS and Android. Property names are case-sensitive in Mixpanel.

### Empty strings vs null
Always send `""` rather than `null` or omitting a property for optional fields. Missing properties are excluded from breakdowns entirely; empty strings appear as `(none)` which is more useful for filtering and analysis.

---

## Identity management

Mixpanel uses Simplified ID Merge to link anonymous device sessions to authenticated user profiles.

```swift
// iOS — call identify() on every app open for logged-in users
// NOT just at the moment of login
func applicationDidBecomeActive(_ application: UIApplication) {
    if let storedUserId = UserDefaults.standard.string(forKey: "mixpanel_user_id") {
        Mixpanel.mainInstance().identify(distinctId: storedUserId)
    }
}
```

```kotlin
// Android — same pattern in Application.onCreate or MainActivity.onResume
val storedUserId = prefs.getString("mixpanel_user_id", null)
if (storedUserId != null) {
    mixpanel.identify(storedUserId)
}
```

**Rules:**
- `identify(userId)` → call on login, registration, and every subsequent app open for logged-in users
- `reset()` → call on explicit user-initiated logout only. Never on session timeout, app background, or app update
- `identify()` must always be called **before** `track()` for authentication events

**Relevant docs:**
- [Identifying users](https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified)
- [Simplified ID merge](https://docs.mixpanel.com/docs/tracking-methods/id-management)

---

## Super properties

Super properties are set once and automatically attached to every subsequent event. Do not re-send them as explicit event properties unless overriding.

| Property | Type | Values | When to update |
|---|---|---|---|
| `product_type` | String | `bild_news`, `bild_sport` | On app init — use `registerSuperPropertiesOnce()` |
| `subscription_status` | String | `free`, `bild_plus`, `premium` | On login, registration, subscription change, logout |
| `platform` | String | `ios`, `android` | On app init |
| `app_version` | String | e.g. `"4.2.1"` | On app init |

```swift
// iOS — on app init, before any track() calls
Mixpanel.mainInstance().registerSuperPropertiesOnce(["product_type": "bild_news"])
Mixpanel.mainInstance().registerSuperProperties([
    "subscription_status": currentUser?.subscriptionStatus ?? "free",
    "platform": "ios",
    "app_version": Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? ""
])
```

```kotlin
// Android — on app init
mixpanel.registerSuperPropertiesOnce(JSONObject().put("product_type", "bild_news"))
mixpanel.registerSuperProperties(JSONObject().apply {
    put("subscription_status", currentUser?.subscriptionStatus ?: "free")
    put("platform", "android")
    put("app_version", BuildConfig.VERSION_NAME)
})
```

**Relevant docs:**
- [Super properties](https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#super-properties)

---

## UTM capture

### How AppsFlyer integrates with Mixpanel

AppsFlyer is the recommended method for UTM capture on mobile. It sits between the campaign link and the app install, capturing attribution data at install time and making it available via the AppsFlyer SDK. Mixpanel and AppsFlyer run in parallel — attribution data from AppsFlyer is stored locally on first open and read back into Mixpanel events throughout the user journey.

**Flow:**
```
Campaign link (with UTMs)
    → App Store / Play Store install
        → AppsFlyer SDK captures attribution
            → Store UTMs in local storage
                → Read UTMs into Mixpanel events throughout lifecycle
```

There are two integration patterns:

**Pattern A — Client-side SDK (for install attribution)**

```swift
// iOS — in AppDelegate, after both SDKs are initialised
AppsFlyerLib.shared().onConversionDataSuccess = { [weak self] data in
    let utmMapping: [(String, String)] = [
        ("utm_source",   (data["media_source"] as? String) ?? ""),
        ("utm_medium",   (data["af_channel"] as? String) ?? ""),
        ("utm_campaign", (data["campaign"] as? String) ?? ""),
        ("utm_content",  (data["af_ad"] as? String) ?? ""),
        ("utm_term",     (data["af_keywords"] as? String) ?? "")
    ]
    utmMapping.forEach { key, value in
        // Store for use in all subsequent Mixpanel events
        UserDefaults.standard.set(value, forKey: key)
        // Also set on Mixpanel user profile
        Mixpanel.mainInstance().people.set(key, to: value)
    }
}
```

```kotlin
// Android
AppsFlyerLib.getInstance().registerConversionListener(context, object : AppsFlyerConversionListener {
    override fun onConversionDataSuccess(data: Map<String, Any>) {
        val mappings = mapOf(
            "utm_source"   to (data["media_source"] as? String ?: ""),
            "utm_medium"   to (data["af_channel"] as? String ?: ""),
            "utm_campaign" to (data["campaign"] as? String ?: ""),
            "utm_content"  to (data["af_ad"] as? String ?: ""),
            "utm_term"     to (data["af_keywords"] as? String ?: "")
        )
        mappings.forEach { (key, value) ->
            prefs.edit().putString(key, value).apply()
            mixpanel.people.set(key, value)
        }
    }
})
```

**Pattern B — Server-side postback (for conversion attribution)**

AppsFlyer can send server-to-server postbacks to Mixpanel when a conversion event fires (registration, subscription). This is the most reliable method for revenue attribution as it doesn't depend on the app being open.

Configure in AppsFlyer dashboard:
1. Go to **Configuration → Partner Marketplace → Mixpanel**
2. Enable **In-app event postbacks**
3. Map AppsFlyer events to Mixpanel event names
4. Enable **Send revenue** for subscription events

This automatically sends UTM data from AppsFlyer to Mixpanel as user properties, available for breakdown on any event.

**Relevant docs:**
- [AppsFlyer integration with Mixpanel](https://docs.mixpanel.com/docs/tracking-methods/integrations/mobile-attribution-tracking)
- [AppsFlyer Mixpanel partner docs](https://support.appsflyer.com/hc/en-us/articles/211200306-Mixpanel-integration)

---

### Deep link / universal link UTM parsing

For users arriving via a clicked link (social, email, web referral) rather than a fresh install:

```swift
// iOS — in SceneDelegate or AppDelegate
func scene(_ scene: UIScene, openURLContexts URLContexts: Set<UIOpenURLContext>) {
    guard let url = URLContexts.first?.url,
          let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
          let queryItems = components.queryItems else { return }
    queryItems
        .filter { $0.name.hasPrefix("utm_") }
        .compactMap { item -> (String, String)? in
            guard let value = item.value, !value.isEmpty else { return nil }
            return (item.name, value)
        }
        .forEach { key, value in
            UserDefaults.standard.set(value, forKey: key)
        }
}
```

```kotlin
// Android — in MainActivity.onCreate
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    intent?.data?.let { uri ->
        listOf("utm_source","utm_medium","utm_campaign","utm_content","utm_term")
            .forEach { key ->
                uri.getQueryParameter(key)?.takeIf { it.isNotEmpty() }?.let { value ->
                    prefs.edit().putString(key, value).apply()
                }
            }
    }
}
```

---

### UTM helper function

Use a single helper to retrieve UTMs consistently across all events:

```swift
// iOS
func storedUTMs() -> [String: String] {
    return [
        "utm_source":   UserDefaults.standard.string(forKey: "utm_source") ?? "",
        "utm_medium":   UserDefaults.standard.string(forKey: "utm_medium") ?? "",
        "utm_campaign": UserDefaults.standard.string(forKey: "utm_campaign") ?? "",
        "utm_content":  UserDefaults.standard.string(forKey: "utm_content") ?? "",
        "utm_term":     UserDefaults.standard.string(forKey: "utm_term") ?? ""
    ]
}
```

```kotlin
// Android
fun storedUtms(): Map<String, String> = mapOf(
    "utm_source"   to (prefs.getString("utm_source", "") ?: ""),
    "utm_medium"   to (prefs.getString("utm_medium", "") ?: ""),
    "utm_campaign" to (prefs.getString("utm_campaign", "") ?: ""),
    "utm_content"  to (prefs.getString("utm_content", "") ?: ""),
    "utm_term"     to (prefs.getString("utm_term", "") ?: "")
)
```

---

## New events

---

### `mp_stage_viewed`

**Trigger:** Stage screen becomes visible  
**Purpose:** Analyse progression and drop-offs between onboarding or subscription stages

```swift
// iOS — fire on viewDidAppear, not viewDidLoad
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
    Mixpanel.mainInstance().track(event: "mp_stage_viewed", properties: [
        "stage_id": "subscription_select" // agree shared taxonomy with Android
    ])
}
```

```kotlin
// Android — fire in onResume
override fun onResume() {
    super.onResume()
    mixpanel.track("mp_stage_viewed", JSONObject().apply {
        put("stage_id", "subscription_select")
    })
}
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `stage_id` | String | Agree shared taxonomy across iOS and Android before implementation. e.g. `onboarding_welcome`, `paywall_intro`, `subscription_select`, `payment_confirm` |
| `product_type` | String | Super property — automatic |
| `subscription_status` | String | Super property — automatic |

**Analysis enabled:** Funnels report across `stage_id` values — break down by `stage_id` to see drop-off at each step  
**Relevant docs:** [Funnels](https://docs.mixpanel.com/docs/reports/funnels)

---

### `mp_push_notification_sent`

**Trigger:** Push successfully sent to device  
**Purpose:** Forms step 1 of the push performance funnel  
**Implementation:** Server-side only — the app is not open when a push is sent

```bash
# Server-side via HTTP Ingestion API
# Fire from your push dispatch layer (FCM / APNs integration)

POST https://api-eu.mixpanel.com/track
Content-Type: application/json

{
  "event": "mp_push_notification_sent",
  "properties": {
    "distinct_id":       "CSUID-mixpanel-xxxxx",
    "token":             "<project_token>",
    "article_id":        "12345",
    "article_title":     "Bundesliga preview",
    "article_category":  "sport",
    "content_type":      "article",
    "product_type":      "bild_sport",
    "subscription_status": "free",
    "time":              1714297200
  }
}
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `article_id` | String | Must match `article_id` on `mp_push_notification_opened` and `mp_article_view` for funnel stitching |
| `article_title` | String | |
| `article_category` | String | |
| `content_type` | String | `article`, `video`, `breaking_news` |
| `product_type` | String | Set explicitly server-side — not available from super properties |
| `subscription_status` | String | Read from your backend user record |

> `distinct_id` must match the `$user_id` used in `identify()` on the device. Use the same CSUID format throughout.

**AppsFlyer note:** AppsFlyer can track push sends as custom events via its S2S API. If you use AppsFlyer for push campaign attribution, map `mp_push_notification_sent` in the AppsFlyer event postback configuration so push campaign UTMs flow into Mixpanel automatically.

**Relevant docs:** [HTTP Ingestion API](https://docs.mixpanel.com/docs/tracking/http-api) · [Server-side best practices](https://docs.mixpanel.com/docs/tracking-best-practices/server-side-best-practices)

---

### `mp_push_notification_opened`

**Trigger:** App opened via push notification link  
**Purpose:** Measure open rates and downstream engagement  
**Implementation:** Client-side SDK — fire in the notification handler before `mp_app_open`

```swift
// iOS — in UNUserNotificationCenterDelegate
func userNotificationCenter(_ center: UNUserNotificationCenter,
    didReceive response: UNNotificationResponse,
    withCompletionHandler completionHandler: @escaping () -> Void) {

    let payload = response.notification.request.content.userInfo
    Mixpanel.mainInstance().track(event: "mp_push_notification_opened", properties: [
        "article_id":       payload["article_id"] as? String ?? "",
        "article_title":    payload["article_title"] as? String ?? "",
        "article_category": payload["article_category"] as? String ?? "",
        "content_type":     payload["content_type"] as? String ?? "",
        // UTMs from push payload — override stored UTMs for this session
        "utm_source":       payload["utm_source"] as? String ?? "push",
        "utm_medium":       payload["utm_medium"] as? String ?? "push",
        "utm_campaign":     payload["utm_campaign"] as? String ?? "",
        "utm_content":      payload["utm_content"] as? String ?? "",
        "utm_term":         payload["utm_term"] as? String ?? ""
    ])
    completionHandler()
}
```

```kotlin
// Android — in FirebaseMessagingService or notification click handler
override fun onMessageReceived(message: RemoteMessage) {
    val data = message.data
    mixpanel.track("mp_push_notification_opened", JSONObject().apply {
        put("article_id",       data["article_id"] ?: "")
        put("article_title",    data["article_title"] ?: "")
        put("article_category", data["article_category"] ?: "")
        put("content_type",     data["content_type"] ?: "")
        put("utm_source",       data["utm_source"] ?: "push")
        put("utm_medium",       data["utm_medium"] ?: "push")
        put("utm_campaign",     data["utm_campaign"] ?: "")
        put("utm_content",      data["utm_content"] ?: "")
        put("utm_term",         data["utm_term"] ?: "")
    })
}
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `article_id` | String | Must match `mp_push_notification_sent` and `mp_article_view` |
| `article_title` | String | From notification payload |
| `article_category` | String | From notification payload |
| `content_type` | String | From notification payload |
| `utm_source` | String | Use push campaign UTMs from payload, not stored install UTMs |
| `utm_medium` | String | Default to `"push"` if not in payload |
| `utm_campaign` | String | Push campaign name |
| `product_type` | String | Super property — automatic |
| `subscription_status` | String | Super property — automatic |

**Push performance funnel:** `mp_push_notification_sent → mp_push_notification_opened → mp_article_view → mp_paywall_viewed → mp_subscription_started`

**AppsFlyer note:** If using AppsFlyer for push attribution, AppsFlyer will capture push open data independently. Compare AppsFlyer's push open count against Mixpanel's `mp_push_notification_opened` count as a validation step.

**Relevant docs:** [Funnels](https://docs.mixpanel.com/docs/reports/funnels)

---

### `mp_registration_completed`

**Trigger:** Registration API returns success  
**Purpose:** Marks conversion from anonymous to known user — most important identity event  
**Implementation:** Client-side SDK — `identify()` must be called before `track()`

```swift
// iOS — in registration success callback
func registrationDidSucceed(userId: String, subscriptionStatus: String) {

    // Step 1 — identify BEFORE tracking
    Mixpanel.mainInstance().identify(distinctId: userId)

    // Step 2 — update super properties
    Mixpanel.mainInstance().registerSuperProperties([
        "subscription_status": "free"
    ])

    // Step 3 — set user profile properties including UTMs
    Mixpanel.mainInstance().people.set(properties: [
        "subscription_status": "free",
        "$created": ISO8601DateFormatter().string(from: Date())
    ].merging(storedUTMs()) { _, new in new })

    // Step 4 — store userId for future app opens
    UserDefaults.standard.set(userId, forKey: "mixpanel_user_id")

    // Step 5 — track the event
    var properties: Properties = [
        "product_id":   "bild_news_free",
        "product_name": "BILD News",
        "product_type": "bild_news",
    ]
    properties.merge(storedUTMs()) { _, new in new }
    Mixpanel.mainInstance().track(event: "mp_registration_completed", properties: properties)
}
```

```kotlin
// Android
fun onRegistrationSuccess(userId: String) {

    // Step 1 — identify BEFORE tracking
    mixpanel.identify(userId)

    // Step 2 — update super properties
    mixpanel.registerSuperProperties(JSONObject().apply {
        put("subscription_status", "free")
    })

    // Step 3 — set user profile properties including UTMs
    mixpanel.people.set(JSONObject().apply {
        put("subscription_status", "free")
        put("\$created", System.currentTimeMillis())
        storedUtms().forEach { (k, v) -> put(k, v) }
    })

    // Step 4 — store userId
    prefs.edit().putString("mixpanel_user_id", userId).apply()

    // Step 5 — track the event
    mixpanel.track("mp_registration_completed", JSONObject().apply {
        put("product_id",   "bild_news_free")
        put("product_name", "BILD News")
        put("product_type", "bild_news")
        storedUtms().forEach { (k, v) -> put(k, v) }
    })
}
```

**Properties:**

| Property | Type | Source | Notes |
|---|---|---|---|
| `product_id` | String | App | e.g. `bild_news_free` |
| `product_name` | String | App | e.g. `BILD News` |
| `product_type` | String | Super property | `bild_news` or `bild_sport` |
| `subscription_status` | String | Super property | `free` at registration |
| `utm_source` | String | AppsFlyer / stored | Acquisition source |
| `utm_medium` | String | AppsFlyer / stored | |
| `utm_campaign` | String | AppsFlyer / stored | |
| `utm_content` | String | AppsFlyer / stored | |
| `utm_term` | String | AppsFlyer / stored | |

**AppsFlyer note:** If AppsFlyer postbacks are configured to Mixpanel, AppsFlyer will also send its own conversion event with attribution data at this point. The client-side UTMs and the AppsFlyer postback UTMs should match — if they don't, the AppsFlyer postback is the more reliable source.

**Relevant docs:** [Identifying users](https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified) · [User profiles](https://docs.mixpanel.com/docs/data-structure/user-profiles) · [AppsFlyer integration](https://docs.mixpanel.com/docs/tracking-methods/integrations/mobile-attribution-tracking)

---

### `mp_login_completed`

**Trigger:** Login API returns success  
**Purpose:** Track returning user behaviour and authentication flows

```swift
// iOS
func loginDidSucceed(userId: String, subscriptionStatus: String) {

    // Step 1 — identify BEFORE tracking
    Mixpanel.mainInstance().identify(distinctId: userId)

    // Step 2 — update super properties with current subscription state
    Mixpanel.mainInstance().registerSuperProperties([
        "subscription_status": subscriptionStatus
    ])

    // Step 3 — store userId for future app opens
    UserDefaults.standard.set(userId, forKey: "mixpanel_user_id")

    // Step 4 — track the event
    Mixpanel.mainInstance().track(event: "mp_login_completed", properties: [
        "product_id":   "bild_news_free",
        "product_name": "BILD News",
        "product_type": "bild_news"
    ])
}
```

```kotlin
// Android
fun onLoginSuccess(userId: String, subscriptionStatus: String) {
    mixpanel.identify(userId)
    mixpanel.registerSuperProperties(JSONObject().apply {
        put("subscription_status", subscriptionStatus)
    })
    prefs.edit().putString("mixpanel_user_id", userId).apply()
    mixpanel.track("mp_login_completed", JSONObject().apply {
        put("product_id",   "bild_news_free")
        put("product_name", "BILD News")
        put("product_type", "bild_news")
    })
}
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `product_id` | String | |
| `product_name` | String | |
| `product_type` | String | Super property — automatic |
| `subscription_status` | String | Super property — read from user session |

> No UTMs on this event — the user already exists, acquisition attribution was captured at registration.

**Relevant docs:** [Identifying users](https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified)

---

### `mp_logout_completed`

**Trigger:** User logs out successfully  
**Purpose:** Understand session endings and account switching

```swift
// iOS — track BEFORE reset()
func logoutDidSucceed() {

    // Step 1 — track BEFORE reset
    Mixpanel.mainInstance().track(event: "mp_logout_completed", properties: [
        "product_id":   "bild_news_free",
        "product_name": "BILD News",
        "product_type": "bild_news"
    ])

    // Step 2 — clear stored userId
    UserDefaults.standard.removeObject(forKey: "mixpanel_user_id")

    // Step 3 — reset() LAST — clears device_id, user_id and super properties
    Mixpanel.mainInstance().reset()
}
```

```kotlin
// Android
fun onLogoutSuccess() {
    mixpanel.track("mp_logout_completed", JSONObject().apply {
        put("product_id",   "bild_news_free")
        put("product_name", "BILD News")
        put("product_type", "bild_news")
    })
    prefs.edit().remove("mixpanel_user_id").apply()
    mixpanel.reset()
}
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `product_id` | String | |
| `product_name` | String | |
| `product_type` | String | Super property — automatic |
| `subscription_status` | String | Super property — automatic |

> `reset()` must be called **after** `track()` and only on explicit user-initiated logout. Calling it anywhere else generates orphaned anonymous profiles and inflates DAU.

**Relevant docs:** [Reset](https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#reset-at-logout)

---

### `mp_subscription_cancelled`

**Trigger:** Cancellation confirmed by backend or store  
**Purpose:** Churn analysis and revenue impact tracking  
**Implementation:** Server-side preferred — most cancellations happen outside the app via Apple/Google subscription management

```bash
# Server-side — fire from App Store Server Notifications (iOS)
# or Google Play Real-time Developer Notifications (Android) webhook handler

POST https://api-eu.mixpanel.com/track
Content-Type: application/json

{
  "event": "mp_subscription_cancelled",
  "properties": {
    "distinct_id":         "CSUID-mixpanel-xxxxx",
    "token":               "<project_token>",
    "$amount":             7.99,
    "currency":            "EUR",
    "event_action":        "cancel",
    "event_type":          "subscription",
    "product_id":          "bild_plus_monthly",
    "product_name":        "BILDplus Monthly",
    "product_type":        "bild_news",
    "subscription_status": "free",
    "time":                1714297200
  }
}
```

```swift
// iOS — client-side fallback if in-app cancellation flow exists
// Also update subscription_status super property
Mixpanel.mainInstance().registerSuperProperties(["subscription_status": "free"])
Mixpanel.mainInstance().track(event: "mp_subscription_cancelled", properties: [
    "$amount":             7.99,    // Double — last billed amount
    "currency":            "EUR",
    "event_action":        "cancel",
    "event_type":          "subscription",
    "product_id":          product.productIdentifier,
    "product_name":        product.localizedTitle,
])
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `$amount` | Double | Last billed amount — must be Double not String |
| `currency` | String | `EUR`, `CHF`, `USD` |
| `event_action` | String | `cancel` |
| `event_type` | String | `subscription` |
| `product_id` | String | |
| `product_name` | String | |
| `product_type` | String | `bild_news` or `bild_sport` |
| `subscription_status` | String | `free` after cancellation |

**AppsFlyer note:** Configure cancellation events as in-app event postbacks in AppsFlyer to ensure churn is reflected in campaign-level LTV reporting.

**Relevant docs:** [HTTP Ingestion API](https://docs.mixpanel.com/docs/tracking/http-api)

---

### `mp_home_item_tapped`

**Trigger:** User taps a feed item  
**Purpose:** Identify which placements and content types drive engagement

```swift
// iOS
Mixpanel.mainInstance().track(event: "mp_home_item_tapped", properties: [
    "content_id":       article.id,
    "content_type":     article.format,     // article, video, breaking_news, sponsored
    "position_in_feed": indexPath.row + 1,  // Int — 1-indexed
    "article_category": article.category,
])
```

```kotlin
// Android
mixpanel.track("mp_home_item_tapped", JSONObject().apply {
    put("content_id",       article.id)
    put("content_type",     article.format)
    put("position_in_feed", adapterPosition + 1)  // Int — 1-indexed
    put("article_category", article.category)
})
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `content_id` | String | Must match `article_id` on `mp_article_view` — enables Flows analysis |
| `content_type` | String | Agree fixed taxonomy: `article`, `video`, `breaking_news`, `sponsored` |
| `position_in_feed` | Int | 1-indexed. Enables histogram of which positions drive most taps |
| `article_category` | String | |
| `product_type` | String | Super property — automatic |
| `subscription_status` | String | Super property — automatic |

> `position_in_feed` **must be Int** — sending as String prevents histogram analysis in Mixpanel.

**Analysis enabled:** Funnel `mp_home_item_tapped → mp_article_view` broken down by `position_in_feed` — shows which feed positions convert to article reads  
**Relevant docs:** [Insights](https://docs.mixpanel.com/docs/reports/insights) · [Flows](https://docs.mixpanel.com/docs/reports/flows)

---

### `mp_heywebview_viewed`

**Trigger:** User views hey webview / embed  
**Purpose:** Track engagement with embedded webview placements

```swift
// iOS
Mixpanel.mainInstance().track(event: "mp_heywebview_viewed", properties: [
    "placement": "article_bottom"  // agree fixed taxonomy
])
```

```kotlin
// Android
mixpanel.track("mp_heywebview_viewed", JSONObject().apply {
    put("placement", "article_bottom")
})
```

**Properties:**

| Property | Type | Notes |
|---|---|---|
| `placement` | String | Fixed taxonomy: `article_bottom`, `home_feed`, `paywall_pre`, `navigation_tab` |
| `product_type` | String | Super property — automatic |
| `subscription_status` | String | Super property — automatic |

---

## Extended existing events

---

### `mp_app_open`

**Extension:** Add `start_type` and UTM parameters on cold start only

```swift
// iOS
func applicationDidFinishLaunching(_ application: UIApplication) {
    var properties: Properties = ["start_type": "cold_start"]
    properties.merge(storedUTMs()) { _, new in new }
    Mixpanel.mainInstance().track(event: "mp_app_open", properties: properties)
}
```

```kotlin
// Android — cold start only
if (isTaskRoot) {
    mixpanel.track("mp_app_open", JSONObject().apply {
        put("start_type", "cold_start")
        storedUtms().forEach { (k, v) -> put(k, v) }
    })
}
```

**New properties:**

| Property | Type | Notes |
|---|---|---|
| `start_type` | String | `cold_start` or `warm_start` |
| `utm_source` | String | From AppsFlyer / deep link — read from local storage |
| `utm_medium` | String | |
| `utm_campaign` | String | |
| `utm_content` | String | |
| `utm_term` | String | |

> Attach UTMs on `cold_start` only. On `warm_start` the user is returning to an existing session, not arriving from a campaign.

**Relevant docs:** [Swift SDK](https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#sending-events) · [AppsFlyer integration](https://docs.mixpanel.com/docs/tracking-methods/integrations/mobile-attribution-tracking)

---

### `mp_article_view`

**Extension:** Add content context and UTM parameters

```swift
// iOS
func articleDidAppear(article: Article, entrySource: String) {
    // Resolve UTMs based on entry source
    let utms = entrySource == "push_notification"
        ? pushNotificationUTMs()   // from notification payload
        : storedUTMs()             // from AppsFlyer / deep link

    var properties: Properties = [
        "article_id":       article.id,
        "article_title":    article.title,
        "article_category": article.category,   // NEW — critical for content analysis
        "content_type":     article.format,     // NEW — article, video, gallery
        "entry_source":     entrySource,        // NEW — home_feed, push_notification, search
    ]
    properties.merge(utms) { _, new in new }
    Mixpanel.mainInstance().track(event: "mp_article_view", properties: properties)
}
```

```kotlin
// Android
fun onArticleViewed(article: Article, entrySource: String) {
    val utms = if (entrySource == "push_notification")
        pushNotificationUtms() else storedUtms()

    mixpanel.track("mp_article_view", JSONObject().apply {
        put("article_id",       article.id)
        put("article_title",    article.title)
        put("article_category", article.category)
        put("content_type",     article.format)
        put("entry_source",     entrySource)
        utms.forEach { (k, v) -> put(k, v) }
    })
}
```

**New properties:**

| Property | Type | Notes |
|---|---|---|
| `article_id` | String | Must match `content_id` on `mp_home_item_tapped` and `mp_push_notification_sent` |
| `article_title` | String | |
| `article_category` | String | **Highest priority new property** — currently undefined on all events, blocks Content Topic dashboard |
| `content_type` | String | `article`, `video`, `gallery` |
| `entry_source` | String | `home_feed`, `push_notification`, `search`, `recommendation`, `direct`, `share_link` |
| `utm_*` | String | Use push payload UTMs when `entry_source = push_notification`, stored UTMs otherwise |

**Relevant docs:** [Flows](https://docs.mixpanel.com/docs/reports/flows) · [Attribution](https://docs.mixpanel.com/docs/tracking-best-practices/traffic-attribution)

---

### `mp_article_closed`

**Extension:** Add `exit_type`

```swift
// iOS
func articleWillDisappear(readDuration: Int, scrollDepth: Int, exitType: String) {
    Mixpanel.mainInstance().track(event: "mp_article_closed", properties: [
        "read_duration": readDuration,  // Int — seconds. Timer must pause on applicationWillResignActive
        "scroll_depth":  scrollDepth,  // Int — percentage 0-100
        "exit_type":     exitType      // back_button, next_article, share, app_background
    ])
}
```

```kotlin
// Android
fun onArticleClosed(readDuration: Int, scrollDepth: Int, exitType: String) {
    mixpanel.track("mp_article_closed", JSONObject().apply {
        put("read_duration", readDuration)
        put("scroll_depth",  scrollDepth)
        put("exit_type",     exitType)
    })
}
```

**New properties:**

| Property | Type | Notes |
|---|---|---|
| `exit_type` | String | `back_button`, `next_article`, `share`, `app_background` |
| `read_duration` | Int | Timer must pause on `applicationWillResignActive` (iOS) and `onPause` (Android) |
| `scroll_depth` | Int | Percentage 0–100. Currently null on all Android events — fix required |

> iOS `read_duration` is currently inflated vs Adobe because the timer doesn't pause on app background. Fix: call `timer.pause()` on `applicationWillResignActive` and `timer.resume()` on `applicationDidBecomeActive`.

**Relevant docs:** [Insights](https://docs.mixpanel.com/docs/reports/insights)

---

### `mp_subscription_started`

**Extension:** Add UTM parameters. Fix `product_type` values.

```swift
// iOS — in StoreKit purchase completion handler
func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
    for transaction in transactions where transaction.transactionState == .purchased {

        var properties: Properties = [
            "product_type":        "bild_news",    // MUST be bild_news or bild_sport — NOT regular or pass
            "product_id":          transaction.payment.productIdentifier,
            "product_name":        product.localizedTitle,
            "$amount":             price,           // Double — NOT String
            "currency":            "EUR",
            "subscription_status": "bild_plus",
        ]
        properties.merge(storedUTMs()) { _, new in new }
        Mixpanel.mainInstance().track(event: "mp_subscription_started", properties: properties)

        // Update subscription_status super property
        Mixpanel.mainInstance().registerSuperProperties(["subscription_status": "bild_plus"])
    }
}
```

```kotlin
// Android — in BillingClient.PurchasesUpdatedListener
override fun onPurchasesUpdated(result: BillingResult, purchases: List<Purchase>?) {
    if (result.responseCode == BillingClient.BillingResponseCode.OK) {
        purchases?.forEach { purchase ->
            mixpanel.track("mp_subscription_started", JSONObject().apply {
                put("product_type",        "bild_news")  // NOT regular or pass
                put("product_id",          purchase.products.first())
                put("product_name",        productName)
                put("\$amount",            price)         // Double
                put("currency",            "EUR")
                put("subscription_status", "bild_plus")
                storedUtms().forEach { (k, v) -> put(k, v) }
            })
            mixpanel.registerSuperProperties(JSONObject().apply {
                put("subscription_status", "bild_plus")
            })
        }
    }
}
```

**Critical fixes:**

| Issue | Fix |
|---|---|
| `product_type` sending `regular` or `pass` on iOS | Change to `bild_news` or `bild_sport` — this is causing iOS subscriptions to disappear from filtered funnels |
| `currency` missing on ~34% of Android events | Ensure `currency` is explicitly set on all purchase code paths |
| `$amount` sent as String on some events | Cast to Double before passing to `track()` |

**New properties:**

| Property | Type | Notes |
|---|---|---|
| `utm_source` | String | From AppsFlyer or stored — closes revenue attribution loop |
| `utm_medium` | String | |
| `utm_campaign` | String | |
| `utm_content` | String | |
| `utm_term` | String | |

**AppsFlyer note:** Configure `mp_subscription_started` as a revenue event in AppsFlyer dashboard (**Configuration → In-App Events**) with `$amount` mapped as the revenue field. This enables LTV reporting per campaign and channel in AppsFlyer, with the event also flowing to Mixpanel via postback.

**Server-side alternative for reliability:**
```bash
POST https://api-eu.mixpanel.com/track
{
  "event": "mp_subscription_started",
  "properties": {
    "distinct_id":         "CSUID-mixpanel-xxxxx",
    "token":               "<project_token>",
    "product_type":        "bild_news",
    "$amount":             7.99,
    "currency":            "EUR",
    "subscription_status": "bild_plus",
    "utm_source":          "facebook",
    "utm_medium":          "paid_social",
    "utm_campaign":        "bild_plus_spring_2026",
    "time":                1714297200
  }
}
```

**Relevant docs:** [HTTP API](https://docs.mixpanel.com/docs/tracking/http-api) · [AppsFlyer integration](https://docs.mixpanel.com/docs/tracking-methods/integrations/mobile-attribution-tracking)

---

### `mp_paywall_viewed`

**Extension:** Add `source` property

```swift
// iOS
Mixpanel.mainInstance().track(event: "mp_paywall_viewed", properties: [
    "source": paywallSource  // article_limit, premium_content, direct_navigation, push_notification
])
```

```kotlin
// Android
mixpanel.track("mp_paywall_viewed", JSONObject().apply {
    put("source", paywallSource)
})
```

**New properties:**

| Property | Type | Notes |
|---|---|---|
| `source` | String | Fixed taxonomy: `article_limit`, `premium_content`, `direct_navigation`, `push_notification` |

---

## Property types reference

| Property | Type | Used on |
|---|---|---|
| `article_id` | String | `mp_article_view`, `mp_push_notification_sent`, `mp_push_notification_opened`, `mp_home_item_tapped` |
| `article_title` | String | `mp_article_view`, `mp_push_notification_sent`, `mp_push_notification_opened` |
| `article_category` | String | `mp_article_view`, `mp_push_notification_sent`, `mp_push_notification_opened`, `mp_home_item_tapped` |
| `content_type` | String | `mp_article_view`, `mp_push_notification_sent`, `mp_push_notification_opened`, `mp_home_item_tapped` |
| `content_id` | String | `mp_home_item_tapped` |
| `entry_source` | String | `mp_article_view` |
| `exit_type` | String | `mp_article_closed` |
| `position_in_feed` | **Int** | `mp_home_item_tapped` |
| `read_duration` | **Int** | `mp_article_closed` |
| `scroll_depth` | **Int** | `mp_article_closed` |
| `$amount` | **Double** | `mp_subscription_started`, `mp_subscription_cancelled` |
| `currency` | String | `mp_subscription_started`, `mp_subscription_cancelled` |
| `stage_id` | String | `mp_stage_viewed` |
| `source` | String | `mp_paywall_viewed` |
| `placement` | String | `mp_heywebview_viewed` |
| `start_type` | String | `mp_app_open` |
| `utm_source` | String | `mp_app_open`, `mp_article_view`, `mp_registration_completed`, `mp_subscription_started`, `mp_push_notification_opened` |
| `utm_medium` | String | Same as above |
| `utm_campaign` | String | Same as above |
| `utm_content` | String | Same as above |
| `utm_term` | String | Same as above |
| `product_type` | String | Super property — all events |
| `subscription_status` | String | Super property — all events |

---

## Mixpanel documentation

| Topic | Link |
|---|---|
| Swift SDK | https://docs.mixpanel.com/docs/tracking-methods/sdks/swift |
| Android SDK | https://docs.mixpanel.com/docs/tracking-methods/sdks/android |
| HTTP Ingestion API | https://docs.mixpanel.com/docs/tracking/http-api |
| Identifying users | https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified |
| Identity management | https://docs.mixpanel.com/docs/tracking-methods/id-management |
| Super properties | https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#super-properties |
| User profiles | https://docs.mixpanel.com/docs/data-structure/user-profiles |
| Reset at logout | https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#reset-at-logout |
| Attribution | https://docs.mixpanel.com/docs/tracking-best-practices/traffic-attribution |
| AppsFlyer integration | https://docs.mixpanel.com/docs/tracking-methods/integrations/mobile-attribution-tracking |
| Server-side best practices | https://docs.mixpanel.com/docs/tracking-best-practices/server-side-best-practices |
| Insights | https://docs.mixpanel.com/docs/reports/insights |
| Funnels | https://docs.mixpanel.com/docs/reports/funnels |
| Flows | https://docs.mixpanel.com/docs/reports/flows |

---

*Maintained by Max Taylor, Solutions Engineering — Mixpanel*  
*Last updated: May 2026*
