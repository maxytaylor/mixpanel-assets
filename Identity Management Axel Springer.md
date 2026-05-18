# Identity Management — Plain English Guide

---

## The core concept in one sentence

Every user starts as an anonymous stranger. The moment they log in or register, Mixpanel needs to know who they are — and `identify()` is how you tell it.

---

## The three states a user can be in

**State 1 — Anonymous (never logged in)**
User opens the app, reads some articles, hits the paywall. Mixpanel gives them a random device ID automatically. You don't need to do anything. All their events are tracked against this anonymous ID.

**State 2 — Logged in**
User registers or logs in. You call `identify()` with their CSUID. Mixpanel links everything they did anonymously to their real profile. From this point on, all events are tracked against their CSUID.

**State 3 — Logged out**
User taps logout. You call `reset()`. Mixpanel clears the identity and starts fresh with a new anonymous ID. Ready for the next user.

---

## The golden rules

1. `identify()` fires **before** any `track()` call for logged-in users
2. `reset()` fires **after** any `track()` call on logout
3. `reset()` is called on **explicit logout only** — nothing else
4. `identify()` fires on **every app open** if the user is already logged in — not just at the moment of login

---

## Step by step — every scenario

---

### Scenario 1 — Brand new user, first time opening the app

Nothing to do for identity. Mixpanel assigns a device ID automatically. Just make sure super properties are registered before the first event fires.

```
App opens
    → registerSuperProperties({ product_type, subscription_status: "free" })
    → track mp_app_open
```

No `identify()` needed. User is anonymous.

---

### Scenario 2 — User registers for the first time

This is the most important moment. The order of operations is critical.

```
Registration form submitted
    → Registration API returns success
        → identify(CSUID)                                              ← step 1 — always first
        → registerSuperProperties({ subscription_status: "free" })    ← step 2
        → save CSUID to local storage                                  ← step 3
        → track mp_registration_completed                             ← step 4 — always last
```

**Why this order matters:** `identify()` tells Mixpanel to link everything this device did anonymously to the CSUID. If you track the event first, it fires on the anonymous profile and the link never happens.

**iOS**
```swift
func registrationDidSucceed(csuid: String) {
    Mixpanel.mainInstance().identify(distinctId: csuid)
    Mixpanel.mainInstance().registerSuperProperties(["subscription_status": "free"])
    UserDefaults.standard.set(csuid, forKey: "mixpanel_user_id")
    Mixpanel.mainInstance().track(event: "mp_registration_completed", properties: [...])
}
```

**Android**
```kotlin
fun onRegistrationSuccess(csuid: String) {
    mixpanel.identify(csuid)
    mixpanel.registerSuperProperties(JSONObject().put("subscription_status", "free"))
    prefs.edit().putString("mixpanel_user_id", csuid).apply()
    mixpanel.track("mp_registration_completed", JSONObject().apply { ... })
}
```

---

### Scenario 3 — Existing user logs in

Same order as registration.

```
Login form submitted
    → Login API returns success
        → identify(CSUID)                                                        ← first
        → registerSuperProperties({ subscription_status: currentStatus })
        → save CSUID to local storage
        → track mp_login_completed                                               ← last
```

**iOS**
```swift
func loginDidSucceed(csuid: String, subscriptionStatus: String) {
    Mixpanel.mainInstance().identify(distinctId: csuid)
    Mixpanel.mainInstance().registerSuperProperties(["subscription_status": subscriptionStatus])
    UserDefaults.standard.set(csuid, forKey: "mixpanel_user_id")
    Mixpanel.mainInstance().track(event: "mp_login_completed", properties: [...])
}
```

**Android**
```kotlin
fun onLoginSuccess(csuid: String, subscriptionStatus: String) {
    mixpanel.identify(csuid)
    mixpanel.registerSuperProperties(JSONObject().put("subscription_status", subscriptionStatus))
    prefs.edit().putString("mixpanel_user_id", csuid).apply()
    mixpanel.track("mp_login_completed", JSONObject().apply { ... })
}
```

---

### Scenario 4 — Already logged-in user opens the app

This is the one most commonly missed. `identify()` must fire on every app open for a logged-in user — not just at the moment they originally logged in. If it only fires at login, returning users who open the app in a new session may appear as new anonymous users.

```
App opens
    → Check local storage for saved CSUID
        → If CSUID exists (user is logged in):
            → identify(CSUID)                                          ← fire immediately
            → registerSuperProperties({ subscription_status: currentStatus })
            → track mp_app_open
        → If no CSUID (user is not logged in):
            → registerSuperProperties({ subscription_status: "free" })
            → track mp_app_open
```

**iOS**
```swift
func applicationDidBecomeActive(_ application: UIApplication) {
    if let csuid = UserDefaults.standard.string(forKey: "mixpanel_user_id") {
        Mixpanel.mainInstance().identify(distinctId: csuid)
        Mixpanel.mainInstance().registerSuperProperties([
            "subscription_status": getCurrentSubscriptionStatus()
        ])
    }
    Mixpanel.mainInstance().track(event: "mp_app_open", properties: ["start_type": "cold_start"])
}
```

**Android**
```kotlin
override fun onResume() {
    super.onResume()
    val csuid = prefs.getString("mixpanel_user_id", null)
    if (csuid != null) {
        mixpanel.identify(csuid)
        mixpanel.registerSuperProperties(JSONObject().put("subscription_status", getCurrentSubscriptionStatus()))
    }
    mixpanel.track("mp_app_open", JSONObject().put("start_type", "cold_start"))
}
```

---

### Scenario 5 — User goes to background and comes back

Do nothing with identity. The user is still the same person in the same session. Just track the app open if applicable.

```
App comes to foreground (warm start)
    → track mp_app_open with start_type: "warm_start"
    → nothing else needed
```

Do **not** call `identify()` again. Do **not** call `reset()`.

---

### Scenario 6 — User logs out

Track the event first, then reset. Never the other way around.

```
User taps logout button
    → track mp_logout_completed                    ← first
    → clear CSUID from local storage
    → reset()                                      ← last
```

**iOS**
```swift
func logoutDidSucceed() {
    Mixpanel.mainInstance().track(event: "mp_logout_completed", properties: [...])
    UserDefaults.standard.removeObject(forKey: "mixpanel_user_id")
    Mixpanel.mainInstance().reset()
}
```

**Android**
```kotlin
fun onLogoutSuccess() {
    mixpanel.track("mp_logout_completed", JSONObject().apply { ... })
    prefs.edit().remove("mixpanel_user_id").apply()
    mixpanel.reset()
}
```

After `reset()`, Mixpanel generates a brand new anonymous device ID. Super properties are wiped. The next user to open the app on this device starts fresh.

---

### Scenario 7 — User subscribes

No identity change needed — they are already identified. Just update super properties to reflect the new subscription status.

```
Purchase confirmed
    → track mp_subscription_started (with $amount, currency, product_type)
    → registerSuperProperties({ subscription_status: "bild_plus" })
```

---

### Scenario 8 — User cancels their subscription

No identity change. Update super properties and track the event — ideally server-side via webhook.

```
Cancellation confirmed
    → track mp_subscription_cancelled
    → registerSuperProperties({ subscription_status: "free" })
```

---

## What should never trigger reset()

`reset()` should **only** fire when a user explicitly taps a logout button. Calling it anywhere else generates a new anonymous device ID and makes the same real user appear as a brand new person in Mixpanel.

| Scenario | What to do instead |
|---|---|
| App goes to background | Nothing |
| App crashes and restarts | Call `identify()` on next open if CSUID in storage |
| Session timeout | Nothing — session continues |
| App update | Nothing — CSUID persists in local storage |
| Device restart | Call `identify()` on next open if CSUID in storage |

---

## The local storage rule

The CSUID should be saved to local storage at login and registration, and removed only at logout. This is the source of truth for whether a user is logged in. On every app open, check local storage first.

```
On every app open:
    storedCSUID = read from local storage

    if storedCSUID exists:
        identify(storedCSUID)    ← always, before any track() call
    else:
        continue as anonymous
```

---

## Super properties — when to update them

Super properties attach automatically to every event. They need to reflect the current user state accurately at all times.

| When | What to update |
|---|---|
| App first opens | Register `product_type` once using `registerSuperPropertiesOnce()` |
| App opens (every time) | Register `subscription_status` with current value |
| Registration | Set `subscription_status` to `free` |
| Login | Set `subscription_status` to current value from your backend |
| Subscription purchased | Set `subscription_status` to `bild_plus` |
| Subscription cancelled | Set `subscription_status` to `free` |
| Logout | `reset()` wipes everything automatically |

---

## The complete flow at a glance

```
FIRST OPEN (anonymous)
    registerSuperProperties
    track mp_app_open

REGISTRATION
    identify(CSUID)                    ← first
    registerSuperProperties
    save CSUID locally
    track mp_registration_completed    ← last

EVERY SUBSEQUENT APP OPEN (logged in)
    read CSUID from storage
    identify(CSUID)                    ← first
    registerSuperProperties
    track mp_app_open

LOGIN
    identify(CSUID)                    ← first
    registerSuperProperties
    save CSUID locally
    track mp_login_completed           ← last

LOGOUT
    track mp_logout_completed          ← first
    clear CSUID from storage
    reset()                            ← last

SUBSCRIPTION
    track mp_subscription_started
    registerSuperProperties({ subscription_status: "bild_plus" })

CANCELLATION
    track mp_subscription_cancelled
    registerSuperProperties({ subscription_status: "free" })

BACKGROUND / FOREGROUND
    track mp_app_open (warm_start)
    nothing else
```

---

## The two most important things to verify in the codebase

1. **Is `identify()` called on every app open for logged-in users, or only at the moment of login?**
If the answer is only at login, this is the primary fix needed — add the local storage check to `applicationDidBecomeActive` (iOS) and `onResume` (Android).

2. **Is the SDK being initialised with the CSUID passed as `distinctId`, or is `identify()` called separately?**
It must always be called separately. Initialising the SDK with a `distinctId` parameter bypasses Simplified ID Merge and means `$user_id` is never formally recorded on the profile.

```swift
// iOS — incorrect
Mixpanel.initialize(token: "TOKEN", distinctId: "CSUID-mixpanel-xxxxx")

// iOS — correct
Mixpanel.initialize(token: "TOKEN")
Mixpanel.mainInstance().identify(distinctId: "CSUID-mixpanel-xxxxx")
```

```kotlin
// Android — incorrect
MixpanelAPI.getInstance(context, "TOKEN", true, "CSUID-mixpanel-xxxxx")

// Android — correct
val mixpanel = MixpanelAPI.getInstance(context, "TOKEN", true)
mixpanel.identify("CSUID-mixpanel-xxxxx")
```

---

## Relevant Mixpanel documentation

| Topic | Link |
|---|---|
| Identifying users (Simplified ID Merge) | https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified |
| Identity management overview | https://docs.mixpanel.com/docs/tracking-methods/id-management |
| iOS Swift SDK | https://docs.mixpanel.com/docs/tracking-methods/sdks/swift |
| Android SDK | https://docs.mixpanel.com/docs/tracking-methods/sdks/android |
| Reset at logout | https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#reset-at-logout |
| Super properties | https://docs.mixpanel.com/docs/tracking-methods/sdks/swift#super-properties |

---

*Maintained by Max Taylor, Solutions Engineering — Mixpanel*
*Last updated: May 2026*
