# Bild Web — Mixpanel JavaScript SDK Implementation Guide

This guide covers Mixpanel tracking for **bild.de** (web), focused on affiliate marketing analytics. It maps core Mixpanel JS SDK concepts directly to Bild's affiliate use cases — page views, button clicks, affiliate link tracking, conversion paths, identity management, user attributes, and UTM capture.

---

## Contents

- [How to load the SDK via Tealium](#how-to-load-the-sdk-via-tealium)
- [SDK initialisation](#sdk-initialisation)
- [Tracking page views](#tracking-page-views)
- [Tracking button clicks](#tracking-button-clicks)
- [Tracking affiliate link clicks](#tracking-affiliate-link-clicks)
- [Path to success — conversion funnel](#path-to-success--conversion-funnel)
- [Identity management](#identity-management)
- [Setting user attributes](#setting-user-attributes)
- [UTM and campaign attribution](#utm-and-campaign-attribution)
- [Complete event property reference](#complete-event-property-reference)
- [Mixpanel documentation](#mixpanel-documentation)

---

## How to load the SDK via Tealium

Load the Mixpanel JS SDK as a JavaScript tag through Tealium rather than using Tealium's native Mixpanel connector. This gives you full access to the SDK — custom event properties, identity management, UTM capture — exactly as documented below.

**In Tealium iQ:**
1. Go to **Tags → Add Tag → Custom JavaScript**
2. Paste the Mixpanel SDK snippet (see [SDK initialisation](#sdk-initialisation) below) as the tag code
3. Set the load rule to fire on **All Pages**
4. Publish the tag

Once the tag is live, all tracking code in this guide runs directly from the SDK — Tealium is simply the delivery mechanism.

---

## SDK initialisation

Run this once when the SDK loads. It sets up the Mixpanel instance before any tracking calls.

```javascript
mixpanel.init('<your_project_token>', {
    debug: false,
    persistence: 'localStorage',          // avoids cookie consent complications
    track_pageview: false,                 // we track page views manually for full control
    opt_out_tracking_by_default: true,     // GDPR: all users start opted out
    api_host: 'https://api-eu.mixpanel.com', // EU data residency — required for this project
    stop_utm_persistence: false            // keep UTMs as super properties (see UTM section)
});
```

> **Important — EU data residency:** The Bild project uses EU data residency. The `api_host` must always be set to `https://api-eu.mixpanel.com` or events will be routed to the US servers and will not be ingested into this project.

> **Important — opt out by default:** With `opt_out_tracking_by_default: true`, the SDK initialises with tracking disabled for all users. No events will be sent until `mixpanel.opt_in_tracking()` is called. This means the consent flow below must be in place before any tracking fires.

Immediately after init, capture and register UTMs (see [UTM section](#utm-and-campaign-attribution)) and check for a logged-in user (see [Identity management](#identity-management)) before any `track()` calls.

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

## Tracking page views

Track on every page load. Attach the page type and any UTMs so every page view is attributable to a campaign.

```javascript
// Call on every page load
mixpanel.track('page_viewed', {
    'page_url':      window.location.pathname,
    'page_title':    document.title,
    'page_type':     getPageType(),   // 'article', 'affiliate_landing', 'home', 'search', 'comparison'
    'referrer':      document.referrer,
    'utm_source':    getUTMParam('utm_source'),
    'utm_medium':    getUTMParam('utm_medium'),
    'utm_campaign':  getUTMParam('utm_campaign'),
    'utm_content':   getUTMParam('utm_content'),
    'utm_term':      getUTMParam('utm_term')
});
```

**What this unlocks in Mixpanel:**
- Which pages drive the most traffic and from which campaigns
- Entry point analysis — where do users land before clicking an affiliate link
- Referrer breakdown to understand organic vs paid vs direct traffic

---

## Tracking button clicks

Track any significant button interaction — newsletter signups, login prompts, app download banners, paywall CTAs.

```javascript
document.querySelectorAll('[data-track-click]').forEach(button => {
    button.addEventListener('click', function() {
        mixpanel.track('button_clicked', {
            'button_name':    this.dataset.trackClick,   // e.g. 'newsletter_signup', 'app_download'
            'button_location': this.dataset.location,    // e.g. 'article_footer', 'sidebar', 'hero'
            'page_url':       window.location.pathname,
            'page_type':      getPageType()
        });
    });
});
```

Add `data-track-click="button_name"` and `data-location="location"` attributes to any button in the HTML you want to track — no additional JavaScript needed per button.

---

## Tracking affiliate link clicks

The most important event for affiliate analytics. Fires when a user clicks any outbound affiliate link to Amazon, Otto, Zalando, or other partners.

```javascript
// Attach to all affiliate links
document.querySelectorAll('[data-affiliate]').forEach(link => {
    link.addEventListener('click', function() {
        mixpanel.track('affiliate_link_clicked', {
            'affiliate_partner':  this.dataset.affiliatePartner,  // 'amazon', 'otto', 'zalando'
            'product_id':         this.dataset.productId,
            'product_name':       this.dataset.productName,
            'product_category':   this.dataset.productCategory,   // 'electronics', 'fashion', 'books'
            'affiliate_url':      this.href,
            'article_id':         getArticleId(),
            'article_title':      getArticleTitle(),
            'position_in_page':   getLinkPosition(this),          // Int — 1-indexed position on page
            'page_type':          getPageType(),
            'utm_source':         getUTMParam('utm_source'),
            'utm_medium':         getUTMParam('utm_medium'),
            'utm_campaign':       getUTMParam('utm_campaign')
        });
    });
});
```

**Add these data attributes to affiliate links in HTML:**
```html
<a href="https://amazon.de/product/..."
   data-affiliate
   data-affiliate-partner="amazon"
   data-product-id="B08N5WRWNW"
   data-product-name="Sony WH-1000XM5"
   data-product-category="electronics">
   Buy on Amazon
</a>
```

**Important — dynamically injected affiliate links:**
If affiliate links are injected by a third-party network script after page load, use event delegation instead:

```javascript
// Attach to document body — catches dynamically injected links
document.body.addEventListener('click', function(e) {
    const link = e.target.closest('[data-affiliate]');
    if (link) {
        mixpanel.track('affiliate_link_clicked', {
            'affiliate_partner': link.dataset.affiliatePartner,
            'product_id':        link.dataset.productId,
            'product_name':      link.dataset.productName,
            'product_category':  link.dataset.productCategory,
            'affiliate_url':     link.href,
            'article_id':        getArticleId(),
            'position_in_page':  getLinkPosition(link)
        });
    }
});
```

**What this unlocks in Mixpanel:**
- Which affiliate partners drive the most clicks
- Which articles and content types generate the most affiliate engagement
- Which page positions convert best (`position_in_page` histogram)
- Which campaigns drive users who click affiliate links

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

## Identity management

Web users are anonymous by default. Mixpanel assigns each new visitor a random device ID stored in localStorage. You only need to call `identify()` when a user logs in or registers.

### On page load — check for existing logged-in user

```javascript
// Run on every page load before any track() calls
function initMixpanelIdentity() {
    const storedUserId = localStorage.getItem('bild_user_id');
    if (storedUserId) {
        mixpanel.identify(storedUserId);   // link this session to the known user
    }
    // If no stored ID, user is anonymous — do nothing, Mixpanel handles it
}

initMixpanelIdentity(); // call before any track() calls
```

### On registration

```javascript
function onRegistrationSuccess(userId, email) {
    mixpanel.identify(userId);           // link anonymous history to this user — always first
    localStorage.setItem('bild_user_id', userId);
    mixpanel.people.set({                // set user profile attributes
        '$email':            email,
        'subscription_status': 'free',
        '$created':          new Date().toISOString()
    });
    mixpanel.track('registration_completed', {
        'utm_source':   getUTMParam('utm_source'),
        'utm_medium':   getUTMParam('utm_medium'),
        'utm_campaign': getUTMParam('utm_campaign')
    });
}
```

### On login

```javascript
function onLoginSuccess(userId, subscriptionStatus) {
    mixpanel.identify(userId);
    localStorage.setItem('bild_user_id', userId);
    mixpanel.people.set({ 'subscription_status': subscriptionStatus });
    mixpanel.track('login_completed');
}
```

### On logout

```javascript
function onLogoutSuccess() {
    mixpanel.track('logout_completed');            // track first
    localStorage.removeItem('bild_user_id');
    mixpanel.reset();                              // reset last — generates new anonymous ID
}
```

**The rules:**
- `identify()` always fires **before** `track()` for authentication events
- `reset()` always fires **after** `track()` on logout
- `reset()` is only ever called on explicit user-initiated logout — never on page unload or session timeout

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
| JavaScript SDK | https://docs.mixpanel.com/docs/tracking-methods/sdks/javascript |
| JavaScript quickstart | https://docs.mixpanel.com/docs/tracking/javascript-quickstart |
| Privacy-friendly tracking & opt-in/out | https://docs.mixpanel.com/docs/tracking-methods/sdks/javascript#privacy-friendly-tracking |
| GDPR compliance | https://docs.mixpanel.com/docs/privacy/gdpr-compliance |
| EU data residency | https://docs.mixpanel.com/docs/privacy/eu-residency |
| Identifying users | https://docs.mixpanel.com/docs/tracking-methods/id-management/identifying-users-simplified |
| User profiles | https://docs.mixpanel.com/docs/data-structure/user-profiles |
| Super properties | https://docs.mixpanel.com/docs/tracking-methods/sdks/javascript#super-properties |
| Traffic attribution and UTMs | https://docs.mixpanel.com/docs/tracking-best-practices/traffic-attribution |
| Funnels | https://docs.mixpanel.com/docs/reports/funnels |
| Flows | https://docs.mixpanel.com/docs/reports/flows |
| Tealium integration | https://docs.mixpanel.com/docs/tracking-methods/integrations/tealium |

---

*Maintained by Max Taylor, Solutions Engineering — Mixpanel*
*Last updated: June 2026*
