# Bild Web — Mixpanel via Tealium iQ Implementation Guide

This guide covers Mixpanel tracking for **bild.de** (web) via Tealium iQ, focused on affiliate marketing analytics. It maps Mixpanel's Tealium integration directly to Bild's existing tag hook setup — covering page views, affiliate link tracking, conversion paths, identity management, and UTM capture.

> **How this works:** The Mixpanel Tealium tag wraps the Mixpanel JS SDK. This means Tealium handles deployment and event triggering via your existing UDO data layer, while the full SDK runs underneath — giving you Mixpanel's default properties, identity management, and UTM capture automatically. You do not need to write `mixpanel.track()` calls manually; Tealium's data mappings handle this for you.

---

## Contents

- [Prerequisite — tag hooks](#prerequisite--tag-hooks)
- [Connecting Mixpanel to Tealium iQ](#connecting-mixpanel-to-tealium-iq)
- [SDK initialisation config](#sdk-initialisation-config)
- [Privacy and consent management](#privacy-and-consent-management)
- [Sending events via data mappings](#sending-events-via-data-mappings)
- [Adding event properties from the UDO](#adding-event-properties-from-the-udo)
- [Affiliate link tracking](#affiliate-link-tracking)
- [Path to success — conversion funnel](#path-to-success--conversion-funnel)
- [Identifying users](#identifying-users)
- [Setting user attributes](#setting-user-attributes)
- [UTM and campaign attribution](#utm-and-campaign-attribution)
- [Complete event property reference](#complete-event-property-reference)
- [Mixpanel documentation](#mixpanel-documentation)

---

## Prerequisite — tag hooks

Tealium iQ works through a Universal Data Object (UDO) — a JavaScript object on every page that holds structured data about the current page, user, and event. Events are triggered by setting a `tealium_event` value in the UDO, which Tealium picks up and routes to connected tags.

If you have been using Tealium for a while you almost certainly have `tealium_event` already mapped as a UDO variable. If not, here is how to add it:

1. Go to **Data Layer** in the Tealium iQ main menu
2. Click **Add Variable**
3. Set type to **UDO Variable** and source to `tealium_event`
4. Click **Apply**, then **Save and Publish**

Your UDO variables should then include `tealium_event` as a mapped variable. This is what Mixpanel's data mappings will use to trigger events.

Your existing UDO likely already contains events such as `page_view`, `product_view`, `button_click` etc. These are the hooks you will map to Mixpanel events in the steps below — no new instrumentation needed for events that already exist in the UDO.

**For new events not in your UDO** (e.g. `affiliate_link_click`), you will need to add them to the UDO data layer and ensure they fire at the right moment. See [Affiliate link tracking](#affiliate-link-tracking) below.

---

## Connecting Mixpanel to Tealium iQ

1. Go to **Tags** in Tealium iQ and click **Add Tag**
2. Search for **Mixpanel** and click **Add**
3. Enter your **Mixpanel project token** — this is found in your Mixpanel project settings
4. Set **Load Rules** — for testing, load on all pages. For production, add a consent condition (see [Privacy and consent management](#privacy-and-consent-management))
5. Proceed to the **Data Mappings** section — this is where events are configured
6. **Save and Publish**

> **EU data residency:** The Bild project uses EU data residency. After adding the tag, confirm the `api_host` is set to `https://api-eu.mixpanel.com` in the tag configuration. Events sent to the wrong region will not be ingested.

**Full Mixpanel docs for this step:** https://docs.mixpanel.com/docs/tracking-methods/integrations/tealium

---

## SDK initialisation config

Because Tealium wraps the Mixpanel JS SDK, the following configuration is applied inside the Tealium tag rather than in your site's source code. These settings control how the SDK behaves once Tealium deploys it.

```javascript
mixpanel.init('<your_project_token>', {
    debug: false,
    persistence: 'localStorage',            // avoids cookie consent complications
    track_pageview: false,                   // page views tracked via UDO mapping below
    opt_out_tracking_by_default: true,       // GDPR: all users start opted out
    api_host: 'https://api-eu.mixpanel.com', // EU data residency — required
    stop_utm_persistence: true               // UTMs read fresh on each page load
});
```

> **EU data residency:** Must always be set. Events sent without this will not be ingested into the Bild project.

> **Opt out by default:** No events fire until `mixpanel.opt_in_tracking()` is called. The consent flow below must be in place before any data is sent.

---

## Privacy and consent management

Because `opt_out_tracking_by_default: true` is set at init, no events will fire until the user has explicitly consented. This is the correct pattern for GDPR compliance on bild.de.

### Opt in when consent is given

Call this immediately when the user accepts analytics tracking in your CMP (OneTrust, Cookiebot, or equivalent):

```javascript
// Call when user accepts analytics in consent banner
function onConsentAccepted() {
    mixpanel.opt_in_tracking();
    // Now run the full initialisation sequence
    captureUTMs();
    mixpanel.register({
        'utm_source':   getUTMParam('utm_source'),
        'utm_medium':   getUTMParam('utm_medium'),
        'utm_campaign': getUTMParam('utm_campaign'),
    });
    initMixpanelIdentity();
    mixpanel.track('page_viewed', {
        'page_url':   window.location.pathname,
        'page_type':  getPageType()
    });
}
```

### Opt out when consent is withdrawn

Call this if the user withdraws consent via your CMP:

```javascript
// Call when user revokes analytics consent
function onConsentRevoked() {
    mixpanel.opt_out_tracking();
    // SDK immediately stops sending any data
    // Stored localStorage data is cleared
}
```

### Check consent status

Use this to check whether the user has already consented — useful for returning visitors:

```javascript
// Returns true if user has opted in
if (mixpanel.has_opted_in_tracking()) {
    // user has previously consented — proceed with tracking
    initMixpanelIdentity();
}

// Returns true if user has opted out
if (mixpanel.has_opted_out_tracking()) {
    // user has declined — do not track
}
```

### Returning visitor — consent already stored

For users who consented on a previous visit, their opt-in status is stored in localStorage. The SDK will read this automatically on init. You still need to call `opt_in_tracking()` via your CMP's callback if the stored preference isn't being read — or check `has_opted_in_tracking()` on page load and act accordingly.

```javascript
// On page load — check stored consent before running full init
if (mixpanel.has_opted_in_tracking()) {
    onConsentAccepted(); // run full init for returning consented user
}
```

> **Note on localStorage vs cookies:** With `persistence: 'localStorage'`, Mixpanel stores its device ID and opt-in/out status in localStorage rather than cookies. This means the consent state persists across sessions without requiring a cookie. Most CMP implementations target cookies specifically and will not block localStorage, but confirm with your legal team whether your CMP should also gate localStorage access under your specific privacy policy.

---

## Sending events via data mappings

Events are sent to Mixpanel by mapping `tealium_event` values to Mixpanel's `track` method in the Tealium data mappings UI. Each mapping says: "when this UDO event fires, call `mixpanel.track()` with this event name."

**In Tealium data mappings:**

1. Select variable: `tealium_event`
2. Set trigger value: the UDO event name (e.g. `page_view`)
3. Bind to: `track`
4. Add a custom value for the Mixpanel event name (e.g. `page_viewed`)
5. Map that custom value to `track → eventName`

Repeat this for each event you want to send to Mixpanel. Your existing UDO events map directly — if `page_view` already fires for Adobe, the same hook sends it to Mixpanel.

**Events to map from existing UDO hooks:**

| UDO `tealium_event` | Mixpanel event name | Notes |
|---|---|---|
| `page_view` | `page_viewed` | Already firing for Adobe |
| `product_view` | `product_viewed` | Already firing if in UDO |
| `button_click` | `button_clicked` | Already firing if in UDO |
| `affiliate_click` | `affiliate_link_clicked` | May need adding — see below |
| `login` | `login_completed` | Triggers `identify()` — see identity section |
| `registration` | `registration_completed` | Triggers `identify()` — see identity section |

---

## Adding event properties from the UDO

Custom event properties are mapped from UDO variables to Mixpanel using the pattern `track.properties.KEY_NAME` in the data mappings UI, where `KEY_NAME` is the property label that will appear in Mixpanel.

**In Tealium data mappings, for each property:**

1. Select the UDO variable (e.g. `page_name`, `article_id`, `affiliate_partner`)
2. Bind to: `track.properties.KEY_NAME` (e.g. `track.properties.article_id`)

The value will be read from the UDO at runtime and sent to Mixpanel as an event property.

**If these UDO variables already exist in your data layer for Adobe, no new instrumentation is needed — just map them to Mixpanel.**

Example mappings for the `page_viewed` event:

| UDO variable | Mixpanel property | Maps to |
|---|---|---|
| `page_name` | `page_type` | `track.properties.page_type` |
| `article_id` | `article_id` | `track.properties.article_id` |
| `article_name` | `article_title` | `track.properties.article_title` |
| `utm_source` | `utm_source` | `track.properties.utm_source` |

> **Default properties:** Because the Mixpanel tag wraps the JS SDK, Mixpanel's default properties (`$browser`, `$os`, `$current_url`, `$referrer` etc) are automatically included on every event. You do not need to map these manually.

---

## Affiliate link tracking

This is the most important event for the affiliate use case. If `affiliate_link_click` (or equivalent) already exists in your Tealium UDO, map it exactly as above. If it does not exist yet, it needs to be added to the data layer.

**Adding to the UDO if not already present:**

```javascript
// Fire when a user clicks an affiliate link
// Add this to the page JavaScript or via a Tealium extension
utag_data.tealium_event = 'affiliate_link_click';
utag_data.affiliate_partner = 'amazon';     // amazon, otto, zalando
utag_data.product_id = '12345';
utag_data.product_name = 'Sony WH-1000XM5';
utag_data.product_category = 'electronics';
utag_data.affiliate_url = 'https://amazon.de/...';
utag_data.article_id = getCurrentArticleId();
utag_data.position_in_page = getLinkPosition(element); // must be a number
utag.link(utag_data);  // fires the UDO event
```

**Then in Tealium data mappings, map these UDO variables:**

| UDO variable | Mixpanel property |
|---|---|
| `affiliate_partner` | `track.properties.affiliate_partner` |
| `product_id` | `track.properties.product_id` |
| `product_name` | `track.properties.product_name` |
| `product_category` | `track.properties.product_category` |
| `affiliate_url` | `track.properties.affiliate_url` |
| `article_id` | `track.properties.article_id` |
| `position_in_page` | `track.properties.position_in_page` |

> `position_in_page` must be a **number** in the UDO — not a string. This enables histogram analysis of which page positions drive the most affiliate clicks.

---

## Path to success — conversion funnel

Track the full journey from landing on a page through to an affiliate click or subscription. Build this as a Mixpanel Funnel using the events below.

### Step 1 — User lands on a page
Covered by `page_viewed` above.

### Step 2 — User views a product recommendation

Use an IntersectionObserver so the event only fires when the product card enters the visible viewport — not just when the page loads.

```javascript
const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
        if (entry.isIntersecting) {
            const card = entry.target;
            mixpanel.track('product_viewed', {
                'affiliate_partner': card.dataset.affiliatePartner,
                'product_id':        card.dataset.productId,
                'product_name':      card.dataset.productName,
                'product_category':  card.dataset.productCategory,
                'article_id':        getArticleId(),
                'position_in_page':  parseInt(card.dataset.position)
            });
            observer.unobserve(card); // fire once per product per page load
        }
    });
}, { threshold: 0.5 }); // fires when 50% of the card is visible

document.querySelectorAll('[data-affiliate]').forEach(card => {
    observer.observe(card);
});
```

### Step 3 — User clicks the affiliate link
Covered by `affiliate_link_clicked` above.

### Step 4 — User registers or subscribes (if applicable)
Covered in [Identity management](#identity-management) below.

**Funnel to build in Mixpanel:**
```
page_viewed → product_viewed → affiliate_link_clicked
```
Break down by `affiliate_partner` to compare Amazon vs Otto vs others, or by `utm_source` to see which campaigns drive the highest conversion through the funnel.

---

## Identifying users

Identity is handled through Tealium data mappings by binding a UDO variable containing the user ID to Mixpanel's `identify` method. This should only be called on authenticated (logged-in) pages or actions — never for anonymous users.

**In Tealium data mappings:**

1. Select the UDO variable that holds the user's unique ID (e.g. `user_id`, `csuid`, `id_of_user`)
2. Set trigger: the UDO event that fires at login (e.g. `login`, `sign_in`)
3. Bind to: `identify`
4. Map the user ID variable to `identify → uniqueID`

This tells Tealium: "when the `login` UDO event fires, call `mixpanel.identify()` with the value of `user_id`."

> **Important:** The value passed to `identify` must be the same user ID used across all devices and platforms. For Bild, this is the CSUID. Do not use email addresses — they can change and will break cross-device identity.

**For mid-session identity (returning logged-in users):**

If a user is already logged in when they arrive on the page, `identify()` must fire before any events. In Tealium, add an extension that checks for a stored user ID and calls `identify()` on page load:

```javascript
// Tealium extension — fires on page load
var storedUserId = localStorage.getItem('bild_user_id');
if (storedUserId) {
    mixpanel.identify(storedUserId);
}
```

**On logout:**

```javascript
// Fires when tealium_event = 'logout'
// Track the event first, then reset
mixpanel.track('logout_completed');
localStorage.removeItem('bild_user_id');
mixpanel.reset(); // clears identity and localStorage, generates new anonymous ID
```

> `reset()` must only be called on explicit user logout — never on page unload, session timeout, or app update.

---

## Setting user attributes

User attributes (called People properties in Mixpanel) are stored on the user's profile rather than on individual events. They allow you to filter and segment users by their current state — subscription status, acquisition source, first visit date, etc.

### Setting attributes at registration

```javascript
mixpanel.people.set({
    '$email':              email,
    '$name':               displayName,
    'subscription_status': 'free',
    '$created':            new Date().toISOString(),
    // Store acquisition UTMs on the profile for lifecycle analysis
    'utm_source':          getUTMParam('utm_source'),
    'utm_medium':          getUTMParam('utm_medium'),
    'utm_campaign':        getUTMParam('utm_campaign')
});
```

### Updating attributes when subscription status changes

```javascript
// When user subscribes
mixpanel.people.set({ 'subscription_status': 'bild_plus' });

// When subscription is cancelled
mixpanel.people.set({ 'subscription_status': 'free' });
```

### Incrementing a counter on the profile

```javascript
// Increment affiliate clicks counter on the user profile
mixpanel.people.increment('affiliate_clicks_total', 1);
```

**What user attributes unlock:**
- Cohort analysis — "show me the affiliate click rate for bild_plus subscribers vs free users"
- Targeting — export cohorts to Braze or other tools for personalised campaigns
- Attribution — see which acquisition campaign produced users who clicked affiliate links most

---

## UTM and campaign attribution

> **The SDK handles UTMs automatically.** The Mixpanel JS SDK reads UTM parameters from the page URL and attaches them to all events fired from that page load — including `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content` and advertising click IDs (`gclid`, `fbclid` etc). You do not need to read them manually from the URL.

The first time UTMs are seen for an identified user, the SDK also stores them on the user profile as `initial_utm_source`, `initial_utm_campaign` etc — permanent first-touch attribution, set once and never overwritten.

### Recommended init config for UTMs

```javascript
mixpanel.init('<project_token>', {
    stop_utm_persistence: true    // recommended — UTMs are read fresh on each page load
                                  // not carried forward as super properties
                                  // more accurate for last-touch attribution models
});
```

With `stop_utm_persistence: false` (default), UTMs from the landing page persist as super properties and attach to every subsequent event in the session, even on pages with no UTM parameters in the URL. Use `true` if you want each event to carry only the UTMs from that specific page load.

### Manual localStorage approach (optional)

Only needed if you want UTMs available in Tealium's data layer before the SDK fires, or want explicit first-touch vs last-touch logic:

```javascript
function captureUTMs() {
    const params = new URLSearchParams(window.location.search);
    const utmKeys = ['utm_source', 'utm_medium', 'utm_campaign', 'utm_content', 'utm_term'];

    let hasNewUTMs = false;
    utmKeys.forEach(key => {
        const value = params.get(key);
        if (value) {
            localStorage.setItem(key, value);
            hasNewUTMs = true;
        }
    });

    // Store first-touch UTMs separately — never overwrite
    if (hasNewUTMs && !localStorage.getItem('first_utm_source')) {
        utmKeys.forEach(key => {
            const value = params.get(key);
            if (value) localStorage.setItem('first_' + key, value);
        });
    }
}

function getUTMParam(key) {
    return localStorage.getItem(key) || '';
}
```

Always send empty strings rather than omitting UTM properties — missing properties are excluded from breakdowns entirely, whereas empty strings appear as `(none)` which is more useful for filtering.

### Full initialisation sequence (with consent gate)

This is the correct order of operations. Note that steps 2–6 only run after consent is granted — they live inside `onConsentAccepted()`:

```javascript
// Step 1 — Init SDK (runs on every page load, tracking disabled until consent)
mixpanel.init('<project_token>', {
    persistence: 'localStorage',
    track_pageview: false,
    opt_out_tracking_by_default: true,
    api_host: 'https://api-eu.mixpanel.com',
    stop_utm_persistence: true
});

// Step 2–6 — run only after user consents
function onConsentAccepted() {
    mixpanel.opt_in_tracking();      // 2. enable tracking
    captureUTMs();                    // 3. capture UTMs from URL (optional manual capture)
    initMixpanelIdentity();           // 4. identify logged-in user if applicable
    mixpanel.track('page_viewed', {   // 5. track page view
        'page_url':   window.location.pathname,
        'page_title': document.title,
        'page_type':  getPageType(),
        'referrer':   document.referrer
    });
    attachAffiliateListeners();       // 6. attach event listeners
    attachButtonListeners();
    attachProductObservers();
}

// For returning users who already consented
if (mixpanel.has_opted_in_tracking()) {
    onConsentAccepted();
}
```

---

## Complete event property reference

| Property | Type | Used on | Notes |
|---|---|---|---|
| `page_url` | String | `page_viewed`, `button_clicked`, `affiliate_link_clicked` | `window.location.pathname` |
| `page_title` | String | `page_viewed` | `document.title` |
| `page_type` | String | All | `article`, `affiliate_landing`, `comparison`, `home`, `search` |
| `referrer` | String | `page_viewed` | `document.referrer` |
| `button_name` | String | `button_clicked` | e.g. `newsletter_signup`, `app_download` |
| `button_location` | String | `button_clicked` | e.g. `article_footer`, `sidebar` |
| `affiliate_partner` | String | `affiliate_link_clicked`, `product_viewed` | `amazon`, `otto`, `zalando` |
| `product_id` | String | `affiliate_link_clicked`, `product_viewed` | Unique product identifier |
| `product_name` | String | `affiliate_link_clicked`, `product_viewed` | |
| `product_category` | String | `affiliate_link_clicked`, `product_viewed` | `electronics`, `fashion`, `books`, `home` |
| `affiliate_url` | String | `affiliate_link_clicked` | Full destination URL |
| `article_id` | String | `affiliate_link_clicked`, `product_viewed` | |
| `article_title` | String | `affiliate_link_clicked` | |
| `position_in_page` | **Int** | `affiliate_link_clicked`, `product_viewed` | 1-indexed — must be Int not String |
| `utm_source` | String | All key events | |
| `utm_medium` | String | All key events | |
| `utm_campaign` | String | All key events | |
| `utm_content` | String | All key events | |
| `utm_term` | String | All key events | |

> `position_in_page` must always be an integer — sending as a string prevents histogram analysis in Mixpanel.

> Always send empty strings `""` rather than `null` for missing values — missing properties are excluded from breakdowns entirely.

---

## Mixpanel documentation

| Topic | Link |
|---|---|
| **Tealium integration (primary reference)** | https://docs.mixpanel.com/docs/tracking-methods/integrations/tealium |
| JavaScript SDK | https://docs.mixpanel.com/docs/tracking-methods/sdks/javascript |
| JavaScript quickstart | https://docs.mixpanel.com/docs/tracking/javascript-quickstart |
| Privacy-friendly tracking & opt-in/out | https://docs.mixpanel.com/docs/tracking-methods/sdks/javascript#privacy-friendly-tracking |
| GDPR compliance | https://docs.mixpanel.com/docs/privacy/gdpr-compliance |
| EU data residency | https://docs.mixpanel.com/docs/privacy/eu-residency |
| Identifying users (Simplified ID Merge) | https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified |
| User profiles | https://docs.mixpanel.com/docs/data-structure/user-profiles |
| Super properties | https://docs.mixpanel.com/docs/tracking-methods/sdks/javascript#super-properties |
| Traffic attribution and UTMs | https://docs.mixpanel.com/docs/tracking-best-practices/traffic-attribution |
| Funnels | https://docs.mixpanel.com/docs/reports/funnels |
| Flows | https://docs.mixpanel.com/docs/reports/flows |

---

*Maintained by Max Taylor, Solutions Engineering — Mixpanel*
*Last updated: June 2026*

