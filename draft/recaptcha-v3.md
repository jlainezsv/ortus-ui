# reCAPTCHA v3

## Overview

This application uses Google reCAPTCHA v3 to protect the BoxLang contact form (`/bxcontactus`) from spam and automated abuse. reCAPTCHA v3 is an invisible, score-based system — it runs in the background without user interaction and returns a risk score (0.0 to 1.0) for each request.

The implementation consists of a custom `RecaptchaV3Service` and a frontend `RecaptchaV3Element` widget, both maintained within this repository. It is independent of the legacy `cbReCaptcha` module, which provides reCAPTCHA v2 (visible checkbox) and remains in use by other handlers (`doContact()`, `sendTips()`).

**Current scope:** BoxLang contact form only (`doContactbx` handler, `/bxcontactus` route).

---

## Architecture

```
Browser (BoxLang contact form)
  │
  ├─ RecaptchaV3Element.cfc widget (must be placed in page to activate)
  │    └─ grecaptcha.execute(siteKey, { action: "contact" })
  │       → hidden token field populated
  │
  ├─ POST /bxcontactus (FormData, X-Requested-With: fetch)
  │
  ▼
Router.cfc (line 26)
  post("/bxcontactus") → Site.doContactbx
  │
  ▼
Site.cfc :: doContactbx()
  1. Param form fields
  2. Blacklist check (contacts_blacklist setting)
  3. Resolve minScore (site → global → 0.5)
  4. recaptchaV3Service.verify(token, "contact", minScore)
  │
  ▼
RecaptchaV3Service.cfc :: verify()
  ├─ Read cb_recaptcha_secret_key from site settings
  ├─ POST https://www.google.com/recaptcha/api/siteverify
  ├─ Validate: HTTP 2xx, success=true, action match, score ≥ minScore
  └─ Resolve client IP (proxy headers → cgi.remote_addr)
  │
  ▼
  5. Failure → error response (JSON for AJAX, redirect for standard POST)
  6. Pass → compose and send email via mailServices
  7. Return success response
```

### Files involved

| File | Role |
|------|------|
| `handlers/Site.cfc` | Request handler — orchestrates validation and email send |
| `modules_app/contentbox-custom/models/recaptcha/RecaptchaV3Service.cfc` | Server-side token verification against Google API |
| `modules_app/contentbox-custom/_themes/boxlang/widgets/RecaptchaV3Element.cfc` | Frontend widget — renders Google JS, hidden field, AJAX submit |
| `modules_app/contentbox-custom/_themes/boxlang/views/_contactForm.cfm` | Contact form view template (currently renders legacy v2 reCAPTCHA) |
| `config/Router.cfc` | Route definition for `/bxcontactus` |
| `config/Coldbox.cfc` | Defines `contacts_blacklist` array setting |

---

## Google reCAPTCHA Configuration

Administrators must create and configure a reCAPTCHA v3 key pair in the [Google reCAPTCHA Admin Console](https://www.google.com/recaptcha/admin).

### Key setup

| Item | Details |
|------|---------|
| **Type** | reCAPTCHA v3 |
| **Site Key** | Public key — rendered in browser JavaScript |
| **Secret Key** | Private key — used server-side only, never exposed to the client |
| **Domains** | Must match the production and staging hostnames exactly |

### Domain considerations

- Add every domain that will serve the contact form (e.g., `boxlang.io`, `www.boxlang.io`, any staging domains).
- Localhost is not automatically included — add it explicitly for local development if needed.
- Google validates the hostname on each verification request; mismatched domains cause silent failure.

### Staging vs production

- Use separate reCAPTCHA v3 key pairs for staging and production if you want isolated score analytics.
- The same keys can be shared across environments if score tuning is not environment-specific.
- The `minScore` setting can be adjusted per environment via ContentBox settings.

---

## ContentBox Settings

All settings are managed through the ContentBox admin under **Settings**. Three settings control the reCAPTCHA v3 integration:

| Setting | Purpose | Type | Sensitivity | Scope | Default/Fallback |
|---------|---------|------|-------------|-------|------------------|
| `cb_recaptcha_site_key` | Google reCAPTCHA v3 site key for the JS API | String | **Public** | Site-specific | Empty — reCAPTCHA not rendered if missing |
| `cb_recaptcha_secret_key` | Google reCAPTCHA v3 secret key for server verification | String | **Secret** — never expose client-side | Site-specific | Empty — verification fails if missing |
| `cb_recaptcha_min_score` | Minimum acceptable score threshold | Numeric (0.0–1.0) | Internal | Site-specific, with global fallback | `0.5` (code default) |

### Setting details

**`cb_recaptcha_site_key`**
- Read by `RecaptchaV3Element.cfc:21` via `cb.siteSetting()`.
- Rendered into the Google JS API URL: `https://www.google.com/recaptcha/api.js?render=SITE_KEY`.
- If empty, the widget returns an empty string — no CAPTCHA script or hidden field is rendered.

**`cb_recaptcha_secret_key`**
- Read by `RecaptchaV3Service.cfc:34` via `settingService.getAllSiteSettings()`.
- Sent as the `secret` POST parameter to Google's `siteverify` endpoint.
- If empty, verification returns `"reCAPTCHA configuration is unavailable."` without calling Google.

**`cb_recaptcha_min_score`**
- Read by `Site.cfc:165` with a three-tier fallback:
  1. Site-specific setting via `getAllSiteSettings(slug)`
  2. Global setting via `getSetting("cb_recaptcha_min_score", "")`
  3. Code default of `0.5`
- If the resolved value is non-numeric or ≤ 0, the code default of `0.5` applies.

### Important: no `cb_recaptcha_enabled` flag

There is no feature toggle. The system is implicitly enabled when both `cb_recaptcha_site_key` and `cb_recaptcha_secret_key` contain valid values. To disable reCAPTCHA, clear the site key or secret key in ContentBox settings.

---

## Score / Risk Evaluation

reCAPTCHA v3 returns a score between `0.0` (almost certainly a bot) and `1.0` (almost certainly a human). The score is based on the user's interaction history with Google services and other risk signals.

### Threshold behavior

| Score | Meaning | Result |
|-------|---------|--------|
| ≥ `minScore` | Acceptable risk | Verification passes |
| < `minScore` | Elevated risk | Verification fails — `"Invalid security code. Please try again."` |
| Non-numeric | Indeterminate | Verification fails |

### Current default

The configured default threshold is **0.5**. This is the standard recommendation from Google for most sites.

### How the score is evaluated

1. The client-side widget obtains a token via `grecaptcha.execute()`.
2. The server sends the token to Google's `siteverify` API.
3. Google returns a JSON response containing `score`, `action`, `hostname`, and `challenge_ts`.
4. The service compares the returned score against `minScore`.
5. If the score is below the threshold, the request is rejected with error code `"score-too-low"`.

### Tuning guidance

- Start with the default `0.5`. Adjust only if you observe legitimate users being blocked or spam passing through.
- Scores are relative — a score of `0.3` on a high-traffic site may indicate risk, while the same score on a low-traffic site may be normal.
- Review score distributions in the Google reCAPTCHA Admin Console before adjusting thresholds.

---

## Action Validation

Each reCAPTCHA v3 token includes an **action** string that identifies the user's intent. This prevents tokens generated for one form from being replayed against another.

| Item | Value |
|------|-------|
| **Action name** | `"contact"` |
| **Generated in** | `RecaptchaV3Element.cfc:68` — `grecaptcha.execute(siteKey, { action: "contact" })` |
| **Validated in** | `RecaptchaV3Service.cfc:110-114` — compares against the `action` parameter passed to `verify()` |
| **Called from** | `Site.cfc:179` — `recaptchaV3Service.verify(token, "contact", minScore)` |

The action string must match exactly between the client-side `execute()` call and the server-side `verify()` call. A mismatch results in an `"action-mismatch"` error code and verification failure.

---

## Developer Implementation

### RecaptchaV3Service

**File:** `modules_app/contentbox-custom/models/recaptcha/RecaptchaV3Service.cfc`

A singleton service injected via WireBox. Handles all server-side interaction with Google's reCAPTCHA v3 API.

#### `verify()` method

```cfc
struct function verify(
    required string token,
    string action       = "",
    numeric minScore    = 0.5,
    string remoteIP     = ""
)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `token` | string | Yes | The `g-recaptcha-response` value from the client |
| `action` | string | No | Expected action to validate against Google's response |
| `minScore` | numeric | No | Minimum acceptable score (default: `0.5`) |
| `remoteIP` | string | No | Client IP override; if empty, resolved automatically |

**Return structure:**

```cfc
{
    success     : false,       // boolean — whether all checks passed
    score       : "",          // string  — Google's score (0.0–1.0)
    action      : "",          // string  — action echoed from Google
    hostname    : "",          // string  — hostname that submitted the token
    challenge_ts: "",          // string  — challenge timestamp
    errorCodes  : [],          // array   — error codes (Google or custom)
    errorMessage: ""           // string  — human-readable error description
}
```

**Error handling behavior:**

The service uses two mechanisms to report failures:

1. **`errorCodes` array** — populated only for action mismatch and score threshold failures:

| Code | Meaning |
|------|---------|
| `"action-mismatch"` | Token action does not match expected action |
| `"score-too-low"` | Score is below `minScore` or non-numeric |

2. **`errorMessage` string** — set for all other failure modes (the `errorCodes` array remains empty):

| Error message | Meaning |
|---------------|---------|
| `"reCAPTCHA configuration is unavailable."` | `cb_recaptcha_secret_key` is empty or site settings could not be loaded |
| `"reCAPTCHA verification failed."` | Non-2xx HTTP response from Google |
| `"Google rejected the reCAPTCHA token."` | Google returned `success: false` (invalid, expired, or tampered token) |

All failure modes set `success: false`. The caller (`Site.cfc`) uses a single user-facing message for all captcha failures: `"Invalid security code. Please try again."`. Service-level error details are logged but not exposed to the user.

#### IP resolution

`resolveRemoteIP()` checks headers in this order:
1. `X-Cluster-Client-IP` (load balancer/proxy)
2. `X-Forwarded-For` (proxy)
3. `cgi.remote_addr` (direct connection)
4. `"127.0.0.1"` (fallback)

### RecaptchaV3Element widget

**File:** `modules_app/contentbox-custom/_themes/boxlang/widgets/RecaptchaV3Element.cfc`

A ContentBox theme widget that renders the client-side reCAPTCHA v3 integration.

> **Current integration state:** The `RecaptchaV3Element` widget exists as a standalone widget but is not currently rendered by any view template. The BoxLang contact form view (`_contactForm.cfm`) still references the legacy v2 `recaptchaService@cbRecaptcha` for its reCAPTCHA rendering. The widget must be explicitly placed into a page or template (e.g., via `cb.widget("RecaptchaV3Element")` or `getInstance("RecaptchaV3Element").renderIt()`) to activate the v3 frontend flow.

**What it renders:**

1. `<script>` tag loading `https://www.google.com/recaptcha/api.js?render=SITE_KEY`
2. Hidden `<input>` field (`g-recaptcha-response`)
3. Feedback `<div>` for inline success/error messages
4. JavaScript that intercepts form submit via `fetch()` API

**Frontend token flow:**

1. On form submit, `event.preventDefault()` stops native submission.
2. `grecaptcha.ready()` waits for the API to load.
3. `grecaptcha.execute(siteKey, { action: "contact" })` generates a token.
4. Token is set on the hidden field and a `FormData` object is built from the form.
5. An AJAX `POST` is sent to `/bxcontactus` with header `X-Requested-With: fetch`.
6. The JSON response is parsed — success shows a confirmation message, failure shows an error.
7. The submit button shows a loading state ("Sending...") during the request.

### AJAX vs standard POST behavior

The handler (`Site.cfc :: doContactbx()`) supports both response modes:

| Mode | Detection | Success response | Error response |
|------|-----------|-----------------|----------------|
| AJAX | `X-Requested-With: fetch` header present | JSON: `{ success: true, message: "..." }` | JSON: `{ success: false, message: "..." }` |
| Standard | No fetch header | Redirect to `/` with flash info message | Redirect to `/` with flash error and persisted form fields |

The BoxLang contact form uses AJAX. The standard POST path exists as a fallback if the JavaScript widget fails to load.

### Error handling

- All Google API errors are caught and logged via `log.error()`.
- The service never throws — it always returns a result struct with `success: false` on failure.
- Network timeouts to Google are set to 10 seconds.
- Email send failures are caught separately with a user-facing "Failed to send message" response.

---

## Administrator Setup

### Checklist

1. **Create reCAPTCHA v3 keys** in the [Google reCAPTCHA Admin Console](https://www.google.com/recaptcha/admin).
   - Select "reCAPTCHA v3" as the type.
   - Add all domains that will serve the contact form.

2. **Configure ContentBox settings:**
   - Set `cb_recaptcha_site_key` to the site key (public, safe for client-side).
   - Set `cb_recaptcha_secret_key` to the secret key (server-side only).
   - Optionally set `cb_recaptcha_min_score` (defaults to `0.5` if not set).

3. **Verify domain configuration:**
   - Ensure all production and staging domains are listed in the Google admin console.
   - Mismatched domains cause silent verification failure.

4. **Verify secret key is server-side only:**
   - The secret key must never appear in templates, JavaScript, or HTML source.

5. **Test the contact form:**
   - Submit the form and verify a success message appears.
   - Check server logs for reCAPTCHA-related errors.
   - Verify the email is received.

6. **Verify rejected submissions:**
   - Test with an empty or invalid token to confirm error handling works.
   - Review the Google reCAPTCHA admin console for score analytics.

---

## Testing / Verification

### Expected browser behavior

- On page load, the Google reCAPTCHA v3 script loads asynchronously.
- No visible CAPTCHA widget appears (reCAPTCHA v3 is invisible).
- On form submit, the button shows "Sending..." briefly while the token is generated and the request is sent.
- A success or error message appears inline below the form.

### Expected POST payload

```
POST /bxcontactus
Content-Type: multipart/form-data
X-Requested-With: fetch

name=...
email=...
phone=...
company=...
message=...
g-recaptcha-response=03AGdBq2...
```

### Verification scenarios

| Scenario | User-facing result | Service-level detail (logged) |
|----------|-------------------|-------------------------------|
| Valid token, score ≥ minScore | Email sent, success message | `"reCAPTCHA verification passed"` |
| Valid token, score < minScore | `"Invalid security code. Please try again."` | `"score-too-low"` error code |
| Empty or missing token | `"Invalid security code. Please try again."` | Google rejection or missing response |
| Invalid/tampered token | `"Invalid security code. Please try again."` | `"Google rejected the reCAPTCHA token."` |
| Action mismatch | `"Invalid security code. Please try again."` | `"action-mismatch"` error code |
| Blacklisted email | AJAX: `{ success: false, message: "Thank you, we will respond shortly!" }` shown as error | No email sent, no log entry |
| Missing site key | Widget not rendered, no reCAPTCHA on the page | N/A — client-side only |
| Missing secret key | `"Invalid security code. Please try again."` | `"reCAPTCHA configuration is unavailable."` |
| Google API unreachable | `"Invalid security code. Please try again."` | `"reCAPTCHA verification failed."` |

### Log entries

Successful verification logs:
```
reCAPTCHA verification passed (score: 0.9, action: contact)
```

Failed verification logs the specific reason (missing key, HTTP error, rejection, action mismatch, low score).

---

## Troubleshooting

### Missing or empty reCAPTCHA widget

- **Cause:** `cb_recaptcha_site_key` is empty or not set in ContentBox site settings.
- **User sees:** No CAPTCHA script loaded, form submits without reCAPTCHA token.
- **Fix:** Set the site key in ContentBox admin → Settings → Site Settings for the BoxLang site.

### "Invalid security code. Please try again." (all captcha failures)

This single message covers all reCAPTCHA-related failures. Check server logs for the specific cause:

- **Missing secret key:** `cb_recaptcha_secret_key` is empty. Set it in ContentBox site settings.
- **Invalid token:** Token is expired (tokens last 2 minutes), already used, or tampered. Ensure the site key and secret key are a matching pair.
- **Action mismatch:** The action string in `grecaptcha.execute()` does not match `verify()`. Both must use `"contact"`.
- **Score too low:** Google scored the request below `minScore`. Review score distribution in Google admin. Adjust `cb_recaptcha_min_score` if needed.
- **Google API unreachable:** Network issue or Google downtime. Check connectivity and Google status.

### "Incorrect domain or site key"

- **Cause:** The domain serving the form is not listed in the Google reCAPTCHA admin console for this key.
- **Fix:** Add the domain in the Google admin console. Include both `www` and non-`www` variants.

### Configuration not being picked up

- **Cause:** The setting is stored as a global setting (All Sites) rather than site-specific, or the site slug does not match.
- **Fix:** Verify the setting is assigned to the correct site in ContentBox. The minScore fallback reads: site-specific → global → code default (`0.5`).

---

## Security Considerations

- **Secret key:** Never expose `cb_recaptcha_secret_key` in client-side code, templates, HTML source, or logs. It is read from ContentBox site settings and used only server-side in `RecaptchaV3Service.cfc`.
- **Token validation:** Every form submission must include a fresh `g-recaptcha-response` token. Tokens are single-use and expire after 2 minutes.
- **Score validation:** The score is evaluated server-side. A client-side score check would be trivially bypassed.
- **Action validation:** The action string prevents token replay across different forms. It must match between client and server.
- **No secrets in logs:** The implementation never logs the token value or secret key. Only the score, action, hostname, and error codes are logged.
- **IP resolution:** Client IP is resolved from proxy headers (`X-Cluster-Client-IP`, `X-Forwarded-For`) with a `127.0.0.1` fallback. This is used by Google for analytics only and does not affect verification.
- **Blacklist bypass:** Blacklisted emails receive a deceptively friendly response (`"Thank you, we will respond shortly!"`) to prevent email enumeration. On the AJAX path, this is returned as `{ success: false, ... }` which displays as an error to the user — the email is silently discarded without sending.

---

## Maintenance / Future Migration

### Current scope

The reCAPTCHA v3 implementation is used exclusively by the BoxLang contact form (`doContactbx` handler, `/bxcontactus` route). It is not integrated with any other forms in the application.

### Relationship to legacy `cbReCaptcha`

| Aspect | Legacy `cbReCaptcha` (v2) | New implementation (v3) |
|--------|--------------------------|------------------------|
| reCAPTCHA type | v2 — visible checkbox/challenge | v3 — invisible, score-based |
| Widget | `reCaptchaService.renderForm()` | Custom `RecaptchaV3Element.cfc` |
| Verification | `recaptchaService.isValid(response)` | `recaptchaV3Service.verify(token, action, minScore)` |
| Score threshold | Not applicable | Configurable (default 0.5) |
| Action validation | Not applicable | Validates action string match |
| Error detail | Boolean only | Full error codes, messages, Google response fields |
| Used by | `doContact()`, `sendTips()`, store handlers | `doContactbx()` only |

The legacy `cbReCaptcha` module is an external ForgeBox dependency. It is not modified by this implementation and remains in use by other handlers.

### Extending to additional forms

To use the v3 service with another form:

1. Include the `RecaptchaV3Element` widget (or replicate its JS logic) in the form view.
2. Ensure the form includes a `g-recaptcha-response` hidden field.
3. In the handler, call `recaptchaV3Service.verify()` with the token and an appropriate action string.
4. Use a distinct action name per form (e.g., `"newsletter"`, `"purchase"`) for accurate analytics.

### Extracting into a ContentBox module

The implementation currently lives in `contentbox-custom` under `models/recaptcha/` and `_themes/boxlang/widgets/`. If additional forms adopt reCAPTCHA v3, consider extracting the service and widget into a standalone ContentBox module (similar to `cbReCaptcha`) for reuse across themes and modules.
