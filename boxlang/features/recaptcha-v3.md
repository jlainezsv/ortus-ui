---
title: "reCAPTCHA v3"
summary: Invisible score-based spam protection for BoxLang contact forms.
---

## Overview

The BoxLang contact form uses Google reCAPTCHA v3 to protect against
spam and automated submissions without displaying a visible CAPTCHA
challenge.

reCAPTCHA v3 runs in the background and returns a risk score from `0.0`
to `1.0`. The application verifies the token, expected action, and score
on the server before processing the form submission.

**Current scope:** BoxLang contact form only (`/bxcontactus`).

The active form is stored as ContentBox content rather than a checked-in
`.cfm` template:

``` text
ContentStore("contact-form")
→ RecaptchaV3Element
→ POST /bxcontactus
→ Site.doContactbx()
→ RecaptchaV3Service.verify()
→ Google siteverify
→ mail delivery
```

------------------------------------------------------------------------

## How It Works

The integration has two main parts: frontend token generation and
server-side verification.

### 1. ContentBox renders the contact form

The active BoxLang page renders the ContentBox entry:

``` text
{{{ContentStore slug="contact-form" }}}
```

The `contact-form` Content Store entry contains the form markup and
invokes:

``` text
{{{RecaptchaV3Element}}}
```

ContentBox resolves the widget through the active BoxLang theme.

The active widget is:

``` text
modules_app/contentbox-custom/_themes/boxlang/widgets/RecaptchaV3Element.cfc
```

### 2. The widget prepares reCAPTCHA

`RecaptchaV3Element` reads the public site key:

``` text
cb_recaptcha_site_key
```

If the site key is empty, the widget renders nothing.

When configured, it renders Google's reCAPTCHA v3 API and a hidden token
field:

``` html
<input
    type="hidden"
    name="g-recaptcha-response"
    id="g-recaptcha-response"
    value=""
>
```

The secret key is never included in frontend output.

### 3. A token is generated when the user submits the form

The widget intercepts the form's submit event.

It:

1.  Prevents the default submission.
2.  Prevents duplicate submissions.
3.  Disables the submit button.
4.  Waits for `grecaptcha.ready()`.
5.  Calls:

``` javascript
window.grecaptcha.execute(siteKey, {
    action: "contact"
})
```

6.  Receives a reCAPTCHA token.
7.  Places the token in `g-recaptcha-response`.

The token is generated fresh for each submission.

### 4. The form sends the token to the backend

The widget serializes the form with `FormData` and sends:

``` http
POST /bxcontactus
X-Requested-With: fetch
```

The token is submitted as:

``` text
g-recaptcha-response
```

### 5. The backend verifies the token

The route:

``` text
POST /bxcontactus
```

is handled by:

``` text
Site.doContactbx()
```

The handler reads the submitted token and resolves the minimum accepted
score.

It then calls:

``` text
recaptchaV3Service.verify(
    token,
    "contact",
    minScore
)
```

### 6. `RecaptchaV3Service` contacts Google

The service sends the token and server-side secret to:

``` text
https://www.google.com/recaptcha/api/siteverify
```

The request uses:

``` text
application/x-www-form-urlencoded
```

and includes:

-   `secret`
-   `response`
-   `remoteip`, when available

### 7. The response is validated

The service validates:

-   Google verification succeeded.
-   The returned action matches `contact`.
-   The returned score is numeric.
-   The score is greater than or equal to `minScore`.

The service reads and returns additional Google response data such as:

-   `hostname`
-   `challenge_ts`
-   `error-codes`

The current implementation does **not** independently validate the
returned `hostname` or `challenge_ts`.

### 8. The form is processed only after verification

If verification succeeds, the contact handler continues with email
processing.

If verification fails:

-   The request is rejected.
-   No email is sent.
-   The user receives a generic error message.

For the frontend `fetch` flow, the user-facing failure response is:

``` text
We couldn't send your message. Please try again.
```

The detailed service error is not exposed directly to the user.

------------------------------------------------------------------------

## Configuration

Administrators configure reCAPTCHA through **ContentBox → Settings**.

  ---------------------------------------------------------------------------------
  Setting                     Purpose           Scope             Default /
                                                                  Fallback
  --------------------------- ----------------- ----------------- -----------------
  `cb_recaptcha_site_key`     Public Google     Site-specific     Empty
                              reCAPTCHA site                      
                              key                                 

  `cb_recaptcha_secret_key`   Server-side       Site-specific     Verification
                              Google reCAPTCHA                    fails if
                              secret key                          unavailable

  `cb_recaptcha_min_score`    Minimum accepted  Global with       `0.5`
                              risk score        site-specific     
                                                override          
  ---------------------------------------------------------------------------------

### Site Key

`cb_recaptcha_site_key` is the public key used by the browser.

It is:

-   Retrieved from the current ContentBox site.
-   Used in the Google script URL.
-   Used by `grecaptcha.execute()`.

The site key is public by design.

If it is empty or contains only whitespace, `RecaptchaV3Element` returns
an empty string and does not render the reCAPTCHA integration.

### Secret Key

`cb_recaptcha_secret_key` is used only by the server.

It is retrieved from the current ContentBox site and sent to Google's
verification endpoint as the `secret` parameter.

**Never expose the secret key in client-side code, HTML, browser output,
or logs.**

If the secret cannot be read or is empty, verification fails with:

``` text
reCAPTCHA configuration is unavailable.
```

### Minimum Score

`cb_recaptcha_min_score` controls how strict score validation is.

``` text
Score >= minScore → Pass
Score < minScore  → Fail
```

The accepted range is `0.0` through `1.0`.

The current resolution order is:

``` text
1. Site-specific cb_recaptcha_min_score
2. Global cb_recaptcha_min_score
3. Code fallback: 0.5
```

A configured value is used only when it is numeric and between `0` and
`1`, inclusive.

If the value is missing, blank, invalid, out of range, or settings
resolution fails, the application uses `0.5`.

The current ContentBox data stores `cb_recaptcha_min_score` globally
with a value of `0.5`.

A higher threshold requires a higher reCAPTCHA score. A lower threshold
is more permissive.

Adjust the threshold based on the site's observed score distribution,
spam, and false-positive behavior.

------------------------------------------------------------------------

## Google Configuration

Create a **reCAPTCHA v3** key pair in the Google reCAPTCHA Admin
Console.

You need:

-   Site key
-   Secret key

Configure all domains that will use the integration, including
applicable staging domains.

The site key is public. The secret key must remain server-side.

For materially different sites or environments, use credentials that are
valid for the target hostname rather than assuming that credentials from
another site can be reused.

------------------------------------------------------------------------

## Action Validation

The current contact form uses:

``` text
contact
```

The frontend generates the token with:

``` javascript
grecaptcha.execute(siteKey, {
    action: "contact"
})
```

The backend passes the same action to the verification service:

``` text
action = "contact"
```

The service compares the returned Google action using exact equality.

The action must therefore match between the frontend and backend.

An action mismatch causes verification to fail.

This is important when reusing the integration for other forms: the
frontend and backend must agree on the action being verified.

------------------------------------------------------------------------

## Server-Side Verification

The verification service is:

``` text
modules_app/contentbox-custom/models/recaptcha/RecaptchaV3Service.cfc
```

Its main method is:

``` text
verify(
    token,
    action,
    minScore
)
```

The service:

1.  Resolves the current site's secret key.
2.  Builds a request to Google's `siteverify` endpoint.
3.  Sends the secret and token.
4.  Optionally sends the client IP.
5.  Parses Google's response.
6.  Validates the response.
7.  Returns a structured result.

### Google Verification Request

The endpoint is:

``` text
https://www.google.com/recaptcha/api/siteverify
```

The request uses:

``` http
Content-Type: application/x-www-form-urlencoded
```

with:

``` text
secret
response
remoteip
```

The request timeout is `10` seconds.

When a client IP is available, the service resolves it in this order:

``` text
x-cluster-client-ip
→ X-Forwarded-For
→ cgi.remote_addr
→ 127.0.0.1
```

Only use forwarded client-IP headers when they are controlled by a
trusted proxy boundary.

### Validation Rules

The service requires:

``` text
Google success === true
AND
returned action === expected action
AND
score is numeric
AND
score >= minScore
```

Common service-level failures include:

``` text
The reCAPTCHA action was invalid.
The reCAPTCHA score was too low.
reCAPTCHA verification failed.
reCAPTCHA configuration is unavailable.
```

The handler converts these detailed errors into a generic user-facing
failure response.

------------------------------------------------------------------------

## Frontend Integration

The client-side integration is implemented by:

``` text
modules_app/contentbox-custom/_themes/boxlang/widgets/RecaptchaV3Element.cfc
```

The widget:

1.  Reads the current site's public site key.
2.  Loads Google's reCAPTCHA API.
3.  Creates the hidden `g-recaptcha-response` field.
4.  Finds the active contact form.
5.  Intercepts submission.
6.  Generates a fresh token.
7.  Adds the token to the form.
8.  Sends the form using `fetch`.
9.  Displays the server response.
10. Restores the submit button after completion.

The current widget expects:

``` text
form id="contactForm"
token field id="g-recaptcha-response"
feedback element id="contactFormFeedback"
```

It also assumes a submit button whose text can be changed while the
request is running.

### Frontend Failure Handling

The widget handles:

-   Google API unavailable.
-   `grecaptcha.ready()` unavailable.
-   `grecaptcha.execute()` failure.
-   Token promise rejection.
-   Network failure.
-   Invalid JSON response.
-   Server response with `success: false`.

In these cases:

-   The form is not falsely reported as successful.
-   The feedback element shows an error.
-   The submit button is restored.
-   The form is not reset.

The widget does not generate a token during initial page rendering.
Token generation happens when the user submits the form.

### Environment Behavior

The reCAPTCHA widget does not have a production-only guard.

It runs wherever the Content Store content is rendered and the widget
can be resolved.

The application has a separate environment check that changes the
contact email subject outside production, but that check does not
disable reCAPTCHA.

------------------------------------------------------------------------

## Implementing on Another Site

The current implementation combines generic Google reCAPTCHA v3 concepts
with BoxLang and ContentBox-specific integration.

The following describes what must be recreated versus what must be
adapted.

### 1. Google Setup

Create a reCAPTCHA v3 configuration for the target site.

You need:

-   Public site key.
-   Server-side secret key.
-   Valid hostname/domain configuration.

Keep the site key available to the frontend and keep the secret key
exclusively on the server.

### 2. Frontend

The frontend must:

1.  Load Google's reCAPTCHA JavaScript API.
2.  Obtain the public site key from configuration.
3.  Generate a token when the protected form/action occurs.
4.  Use a named action.
5.  Place the token in the submitted form data.
6.  Send the token to the backend.
7.  Handle asynchronous success and failure states.

The generic pattern is:

``` javascript
grecaptcha.ready(function () {
    grecaptcha.execute(siteKey, {
        action: "..."
    }).then(function (token) {
        // Submit the token with the protected request.
    });
});
```

The current implementation uses:

``` text
action = "contact"
```

and:

``` text
g-recaptcha-response
```

as the token field.

### 3. Backend

The backend must:

1.  Register a route for the protected request.
2.  Read the submitted token.
3.  Resolve the minimum accepted score.
4.  Send `secret` and `response` to Google's `siteverify` endpoint.
5.  Validate Google's response.
6.  Validate the expected action.
7.  Validate the score.
8.  Reject the request before business processing if verification fails.
9.  Continue with the protected operation only after successful
    verification.
10. Avoid logging the secret or raw token.

The current implementation uses:

``` text
POST /bxcontactus
→ Site.doContactbx()
→ RecaptchaV3Service.verify()
```

A different site would replace the route and handler with its own
application architecture.

### 4. Configuration

A reusable implementation should keep these concerns separate:

``` text
Public site key
Server-side secret key
Minimum score
```

The current ContentBox implementation uses:

``` text
cb_recaptcha_site_key
cb_recaptcha_secret_key
cb_recaptcha_min_score
```

These names are implementation-specific. Another site does not need to
use the same setting names.

The important behavior is:

``` text
site key → frontend
secret key → server only
minimum score → server validation
```

### 5. Form Integration

The protected form must:

-   Render the reCAPTCHA integration.
-   Include the token in the submitted request.
-   Submit to a backend endpoint that performs verification.
-   Use the same action name on frontend and backend.
-   Process the protected operation only after successful verification.

For this repository, the ContentBox-specific implementation requires:

``` text
{{{RecaptchaV3Element}}}
```

inside the `contact-form` Content Store entry.

The form currently uses:

``` html
<form
    method="post"
    id="contactForm"
    action="/bxcontactus">
```

and the page renders the Content Store entry with:

``` text
{{{ContentStore slug="contact-form" }}}
```

These details are specific to this implementation and should not be
treated as requirements of Google reCAPTCHA itself.

------------------------------------------------------------------------

## Generic vs. BoxLang / ContentBox-Specific

### Generic reCAPTCHA v3 concepts

The reusable security flow consists of:

-   Google-issued public site key.
-   Server-only secret key.
-   Loading Google's JavaScript API.
-   Calling `grecaptcha.execute()` on the protected action.
-   Using a named action.
-   Sending the generated token to the server.
-   Server-side verification through `siteverify`.
-   Checking Google's `success` result.
-   Checking the returned action.
-   Applying a score threshold.
-   Generating a fresh token for each submission.
-   Failing closed when verification fails.
-   Keeping the secret key out of frontend code and logs.

### BoxLang / ContentBox-specific pieces

The current implementation additionally depends on:

-   ContentBox `ContentStore`.
-   Content Store slug `contact-form`.
-   `RecaptchaV3Element`.
-   ContentBox widget discovery.
-   BoxLang theme widget resolution.
-   `Site.doContactbx()`.
-   `/bxcontactus`.
-   ContentBox site discovery.
-   ContentBox settings and setting services.
-   ContentBox dependency injection.
-   BoxLang HTTP and JSON APIs.
-   ContentBox mail services.
-   The current contact form's DOM structure.
-   The current JSON/fetch response contract.

These pieces would need to be replaced or adapted when implementing the
same reCAPTCHA flow elsewhere.

------------------------------------------------------------------------

## Current Coupling

The current implementation is intentionally scoped to the BoxLang
contact form. The following dependencies are important when considering
future reuse.

### Contact form

The widget currently assumes:

``` text
contactForm
g-recaptcha-response
contactFormFeedback
```

It also assumes a single form instance and a submit button whose state
can be changed.

### Endpoint

The frontend hard-codes:

``` text
/bxcontactus
```

The router maps it to:

``` text
Site.doContactbx()
```

A different endpoint would require changes to the integration.

### Action

Both frontend and backend currently use:

``` text
contact
```

The verification service itself already accepts an action argument, but
the widget and handler currently hard-code the contact action.

### ContentBox

The current implementation relies on:

-   ContentBox widget discovery.
-   ContentBox site discovery.
-   ContentBox settings.
-   Content Store rendering.
-   ContentBox dependency injection.

### Configuration

The implementation expects:

``` text
cb_recaptcha_site_key
cb_recaptcha_secret_key
cb_recaptcha_min_score
```

The site key and secret are site-specific. The minimum score currently
supports a site-specific override with a global fallback.

### Mail processing

The contact handler currently combines reCAPTCHA verification with:

-   Email construction.
-   Mail delivery.
-   Success/failure response handling.
-   Contact-specific recipient and subject configuration.

This mail behavior is not inherently part of reCAPTCHA and would not
belong in a generic verification widget.

------------------------------------------------------------------------

## Next Steps

The current implementation works for the BoxLang contact form, but the intended direction is to evolve the reCAPTCHA integration into a reusable global widget that can be used across sites and forms.

| Area | Status | Notes |
| --- | --- | --- |
| BoxLang Integration | ✅ Completed | reCAPTCHA v3 implemented for the BoxLang contact form. |
| Reusable Widget | 🔜 Planned | Extract the current implementation into a reusable reCAPTCHA v3 widget. |
| Global Widget | 🔜 Planned | Make the widget available globally so it can be used by multiple sites and forms. |
| Configurable Actions | 🔜 Planned | Allow the widget to receive the reCAPTCHA action instead of hard-coding `contact`. |
| Form Integration | 🔜 Planned | Allow different forms to use the widget without duplicating the implementation. |
| Documentation | 🔄 In Progress | Document the current implementation, replication process, and reusable architecture. |

### Future Extraction Opportunities

The current implementation identifies several areas that should be considered when evolving the widget:

1. **Configurable action**  
   The verification service already accepts an action, but the current widget is coupled to `contact`.

2. **Configurable endpoint**  
   The current widget posts directly to `/bxcontactus`.

3. **Form and element targeting**  
   The current widget relies on fixed DOM IDs and a single-form model.

4. **Reusable verification service**  
   `RecaptchaV3Service.verify()` is already separated from the contact handler, making the verification logic a natural reusable boundary.

5. **Configuration abstraction**  
   The verification logic itself does not inherently require ContentBox. The current dependency is primarily in how configuration and the current site are resolved.

6. **Multiple widget instances**  
   The current global submission state and fixed selectors would need to be generalized before supporting multiple forms on the same page.

7. **Reusable response handling**  
   The current frontend expects a contact-form-specific JSON response containing `success` and `message`. A global widget should not depend on that contract.

8. **Score policy**  
   The current site-specific → global → `0.5` resolution is ContentBox-specific and should remain configurable when the integration becomes reusable.

The goal is to preserve the secure server-side verification model while removing the assumptions that currently tie the implementation to a single BoxLang contact form.
