---
title: "Turnstile"
summary: Implementation of Cloudflare Turnstile and a Second Security Layer
---

## 1. Objective

This document describes the implementation applied to the BoxLang site's contact form and serves as a guide for replicating it on another ContentBox/ColdBox/BoxLang site within the company.

The solution has two main layers:

1. **Cloudflare Turnstile**: proves server-side that the request comes from an interaction accepted by Cloudflare.
2. **Server-side validation and business rules**: blocks unwanted requests even if a client attempts to bypass JavaScript or send a direct HTTP request.

Turnstile does not replace server-side validation. The browser token is always considered untrusted input until Cloudflare accepts it through **`siteverify`**.

## 2. Current Scope

The implementation described corresponds to the BoxLang flow:

- Form: BoxLang contact form.
- Endpoint: **`POST /bxcontactus`**.
- Handler: **`Site.doContactbx`**.
- Widget: **`TurnstileElement`**.
- Verification service: **`TurnstileService`**.
- Configuration: per-site ContentBox settings.

Legacy forms on the site, such as **`/contactus`**, **`/sendtips`**, and other existing forms, continue to use **`cbReCaptcha`**. Do not assume that all forms on the site use Turnstile simply because this module exists.

## 3. Implementation Files

| Responsibility | File |
| --- | --- |
| HTTP Route | **`config/Router.cfc`** |
| Handler and server-side rules | **`handlers/Site.cfc`** |
| Service that queries Cloudflare | **`modules_app/contentbox-custom/models/turnstile/TurnstileService.cfc`** |
| Form widget | **`modules_app/contentbox-custom/_themes/boxlang/widgets/TurnstileElement.cfc`** |
| Cloudflare script registration | **`modules_app/contentbox-custom/ModuleConfig.cfc`** |
| Additional blocking configuration | **`config/Coldbox.cfc`**, setting **`contacts_blacklist`** |

## 4. Complete Flow

```text
User

  |
  | 1. Opens the form
  v

ContentBox / BoxLang theme

  |
  | 2. Module adds api.js only if a site key exists
  | 3. Widget explicitly renders Turnstile
  v

Cloudflare Turnstile

  |
  | 4. Injects the hidden cf-turnstile-response field
  v

Browser

  |
  | 5. POST /bxcontactus with FormData + token
  v

Site.doContactbx

  |
  | 6. Checks blacklist and normalizes parameters
  | 7. Sends token to Cloudflare /siteverify
  | 8. Checks success, action, and hostname
  | 9. Checks message length and content
  v

MailService

  |
  | 10. Only after all validations is the email sent
  v

AJAX JSON response
```

The important condition is that the email is sent last. A client can remove the widget, modify the JavaScript, or call the endpoint directly, but it cannot trigger a valid send without passing the server-side checks.

## 5. Per-Site Configuration

Two ContentBox settings are used:

| Setting | Usage | Sensitivity |
| --- | --- | --- |
| **`cb_turnstile_site_key`** | Public key used by the browser to render Turnstile | Public |
| **`cb_turnstile_secret_key`** | Private key sent to Cloudflare in **`siteverify`** | Secret |

The names are defined in:

- **`TurnstileService.cfc`**: reads the site key and secret key.
- **`TurnstileElement.cfc`**: reads the site key for the HTML/JavaScript.
- **`ModuleConfig.cfc`**: detects the site key to load the script.

### Configuration Rules

- Create the keys in Cloudflare Turnstile for the site's actual hostname.
- Also register staging hostnames if they will be tested outside production.
- Store the secret key only as a protected setting or secure deployment variable.
- Do not include the secret key in JavaScript, HTML, the repository, screenshots, or logs.
- Configure both settings for the same ContentBox site. The service resolves the current site using **`siteService.discoverSite().getSlug()`**.
- If **`cb_turnstile_site_key`** does not exist, the script is not loaded and the widget returns empty HTML.
- If **`cb_turnstile_secret_key`** does not exist, verification fails safely and no email is sent.

Conceptual example of the values (use real values only in the administrator or secret store):

```text
Site: BoxLang

cb_turnstile_site_key: 0x4AAAA...

cb_turnstile_secret_key: 0x4AAAA...
```

## 6. Conditional Script Loading

**`ModuleConfig.cfc`** registers an interceptor for **`cbui_beforeHeadEnd`**. The interceptor checks the current site key and adds the following once:

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js?render=explicit"></script>
```

Loading is conditional for two reasons:

1. Do not load Cloudflare JavaScript on sites that do not have Turnstile configured.
2. Avoid duplicates if another component has already added the same URL.

Pattern to replicate:

```cfml
function cbui_beforeHeadEnd(event, interceptData, buffer) {

    if (!len(getCurrentTurnstileSiteKey(arguments.event))) {
        return;
    }

    if (toString(arguments.buffer).findNoCase(variables.turnstileScriptUrl) > 0) {
        return;
    }

    arguments.buffer.append(
        '<script src="' & variables.turnstileScriptUrl & '"></script>'
    );
}
```

## 7. Widget Rendering

The widget uses the explicit rendering API. This allows control over the **`sitekey`**, **`action`**, appearance, size, and response field name.

Current configuration:

```text
action: contact
size: flexible
appearance: execute
theme: auto
response-field: true
response-field-name: cf-turnstile-response
```

Client-side rendering example:

```html
<div id="turnstileWidget"></div>

<script>
    window.turnstileContactWidget = function () {
        var container = document.getElementById("turnstileWidget");

        if (!container || container.dataset.turnstileRendered === "1") {
            return;
        }

        if (!window.turnstile || typeof window.turnstile.render !== "function") {
            return;
        }

        container.dataset.turnstileRendered = "1";

        window.turnstile.render(container, {
            sitekey: "PUBLIC_SITE_KEY",
            action: "contact",
            theme: "auto",
            size: "flexible",
            appearance: "execute",
            "response-field": true,
            "response-field-name": "cf-turnstile-response"
        });
    };

    if (window.turnstile) {
        window.turnstile.ready(window.turnstileContactWidget);
    }
</script>
```

In the actual implementation, the site key is inserted from the site setting. When producing HTML with configuration values, appropriate attribute/JavaScript encoding must be applied. The secret key must never be inserted into this block.

The hidden field generated by Turnstile is automatically submitted with **`FormData`**:

```text
cf-turnstile-response=<single-use-token>
```

Tokens are single-use and expire. Therefore, they must not be stored for later retries or reused after a rejected response.

## 8. AJAX Form Submission

The widget connects the form to the endpoint and submits all its fields, including the token generated by Turnstile:

```javascript
form.addEventListener("submit", function (event) {
    event.preventDefault();

    fetch("/bxcontactus", {
        method: "POST",
        body: new FormData(form),
        headers: {
            "X-Requested-With": "fetch"
        }
    })
    .then(function (response) {
        return response.json();
    })
    .then(function (data) {
        if (data.success) {
            form.reset();
        }
    });
});
```

The **`X-Requested-With: fetch`** header allows the handler to return JSON for the AJAX experience. It is not a security measure: an attacker can forge it.

The client also:

- Prevents double submission while the request is pending.
- Disables the button if the message has fewer than 20 characters.
- Displays error messages without reloading the page.

These validations improve the user experience, but the server repeats them because the browser is not a trust boundary.

## 9. Server-Side Verification Against Cloudflare

The **`TurnstileService`** is a singleton and centralizes all verification. It receives:

```cfml
turnstileService.verify(
    token  = rc["cf-turnstile-response"],
    action = "contact"
)
```

It then makes a form-encoded **`POST`** to:

```text
https://challenges.cloudflare.com/turnstile/v0/siteverify
```

Parameters sent:

```text
secret   = site's secret key
response = token received from the browser
remoteip = client IP, when available
```

Reduced and portable example:

```cfml
struct function verify(required string token, string action = "") {

    var result = {
        success: false,
        action: "",
        hostname: "",
        errorCodes: [],
        errorMessage: ""
    };

    if (!len(trim(arguments.token))) {
        result.errorCodes = ["missing-input-response"];
        result.errorMessage = "Turnstile token is required.";
        return result;
    }

    var httpService = new http(
        method = "post",
        url = "https://challenges.cloudflare.com/turnstile/v0/siteverify",
        timeout = 10
    );

    httpService.addParam(
        type = "formfield",
        name = "secret",
        value = getSecretKey()
    );

    httpService.addParam(
        type = "formfield",
        name = "response",
        value = arguments.token
    );

    var response = httpService.send().getPrefix();
    var payload = deserializeJSON(response.fileContent ?: "{}");

    if (!isStruct(payload) || !payload.success) {
        result.errorMessage = "Cloudflare rejected the Turnstile token.";
        return result;
    }

    if (len(arguments.action) && payload.action != arguments.action) {
        result.errorCodes = ["action-mismatch"];
        result.errorMessage = "The Turnstile action was invalid.";
        return result;
    }

    result.success = true;
    result.action = payload.action ?: "";
    result.hostname = payload.hostname ?: "";
    return result;
}
```

The actual implementation is stricter than this example and also:

- Rejects tokens longer than 2048 characters.
- Uses a 10-second HTTP timeout.
- Handles unsuccessful HTTP status codes.
- Handles empty responses or unexpected JSON.
- Preserves **`error-codes`**, **`action`**, **`hostname`**, **`challenge_ts`**, and **`cdata`**.
- Logs failures in LogBox without exposing the token or secret.
- Rejects the response if **`success`** is missing or false.
- Rejects **`action`** values other than **`contact`**.
- Compares the hostname returned by Cloudflare with the public hostname of the request.

## 10. Hostname and IP Resolution

To validate the hostname, the service uses this priority:

1. **`X-Forwarded-Host`**
2. **`Host`**
3. **`cgi.http_host`**

The port is removed and the value is compared in lowercase with the hostname returned by Cloudflare.

To send **`remoteip`**, it uses this priority:

1. **`cf-connecting-ip`**
2. **`x-cluster-client-ip`**
3. **`X-Forwarded-For`**
4. **`cgi.remote_addr`**
5. **`127.0.0.1`** as the final fallback.

When replicating this behind a proxy, load balancer, or CDN, only accept IP headers that the trusted proxy overwrites and controls. Do not blindly trust headers sent by the client from the Internet.

## 11. Second Layer: Server-Side Form Rules

### 11.1 Email Blacklist

Before processing the submission, **`doContactbx`** compares the email against the **`contacts_blacklist`** setting. If it finds a match, it responds as if the request had been accepted, but does not send email.

This behavior is deliberately silent so that it does not confirm to the abuser that their address is blocked.

The list is currently configured in **`config/Coldbox.cfc`**:

```cfml
settings = {
    contacts_blacklist: [
        "blocked@example.com"
    ]
};
```

For another site, it is recommended to keep the list outside the code when possible, normalize the email before comparison, and record internal metrics without revealing the list to the user.

### 11.2 Required Token and Action/Hostname Verification

The endpoint normalizes the parameter:

```cfml
event.paramValue("cf-turnstile-response", "");
```

An empty token fails. A token valid for another action or hostname also fails. Therefore, simply copying a token generated for another form or domain is not sufficient.

### 11.3 Server-Side Message Validation

Content validation is performed only if Turnstile is valid:

```cfml
var trimmedMessage = trim(rc.message);

if (len(trimmedMessage) < 20) {
    arrayAppend(errors, "Please provide at least 20 characters in your message.");
} else if (
    len(trimmedMessage) >= 15
    && !reFind("\s", trimmedMessage)
    && reFind("^[[:alnum:]]+$", trimmedMessage)
) {
    arrayAppend(errors, "Please provide a more detailed message.");
}
```

This blocks empty messages, messages that are too short, and alphanumeric strings without spaces that often indicate automated or low-value input. This rule is not intended to classify all spam; it is a second quality and abuse barrier.

### 11.4 Order of Checks

The current order is important:

1. Normalize parameters.
2. Apply blacklist.
3. Verify Turnstile against Cloudflare.
4. Validate the message.
5. Build and send the email.
6. Return JSON or redirect depending on the request type.

The email send must never be moved before Turnstile verification and validation.

## 12. Reference Handler

This is a summarized example of the server-side endpoint:

```cfml
function doContactbx(event, rc, prc) {
    var errors = [];
    var isAjax = event.getHTTPHeader("X-Requested-With", "");

    event
        .paramValue("email", "")
        .paramValue("message", "")
        .paramValue("cf-turnstile-response", "");

    if (arrayFind(blacklist, rc.email)) {
        if (isAjax == "fetch") {
            return {
                success: false,
                message: "Thank you, we will respond shortly!"
            };
        }

        relocate("/");
    }

    var turnstileResult = turnstileService.verify(
        token = rc["cf-turnstile-response"],
        action = "contact"
    );

    if (!turnstileResult.success) {
        arrayAppend(errors, "We couldn't send your message. Please try again.");
    }

    if (turnstileResult.success && len(trim(rc.message)) < 20) {
        arrayAppend(errors, "Please provide at least 20 characters in your message.");
    }

    if (arrayLen(errors)) {
        if (isAjax == "fetch") {
            return {
                success: false,
                message: errors.toList(" ")
            };
        }

        return relocate("/?cbcache=true");
    }

    // Build and send the email here, after all validations.
    return {
        success: true,
        message: "Thank you, we will respond shortly!"
    };
}
```

The actual code retains the complete form fields, mail error handling, and responses for both AJAX and traditional requests.

## 13. Replication Checklist

### Cloudflare

- [ ] Create a Turnstile widget for each required hostname.
- [ ] Register production and staging separately when appropriate.
- [ ] Store the site key and secret key.
- [ ] Confirm that the hostname returned by Cloudflare matches the public hostname.

### ContentBox

- [ ] Create **`cb_turnstile_site_key`** in the site settings.
- [ ] Create **`cb_turnstile_secret_key`** in the site settings.
- [ ] Verify that both settings belong to the same site slug.
- [ ] Do not version-control the secret key.

### ColdBox / Module

- [ ] Register **`TurnstileService`** as a singleton model or use auto-mapping.
- [ ] Register the **`api.js`** loading interceptor.
- [ ] Include the widget in the correct form.
- [ ] Add a dedicated POST route to the handler.
- [ ] Inject the service into the handler.
- [ ] Maintain the timeout, token limit, and error handling.

### Form

- [ ] Render Turnstile with **`response-field`** enabled.
- [ ] Use a form-specific **`action`**.
- [ ] Submit **`FormData`** to include the hidden token.
- [ ] Prevent double submission while a request is active.
- [ ] Do not trust JavaScript validation.

### Server

- [ ] Reject missing, oversized, expired, or reused tokens.
- [ ] Check **`success`**.
- [ ] Check **`action`**.
- [ ] Check **`hostname`**.
- [ ] Validate email, message, and input limits.
- [ ] Apply blacklist or anti-abuse rules before sending email.
- [ ] Log failures without logging secrets or complete tokens.
- [ ] Review trust in **`X-Forwarded-*`** according to the actual proxy.

## 14. Test Plan

### Functional Tests

1. Open the form with both settings configured.
2. Confirm that **`api.js`** appears only once.
3. Confirm that the widget renders and creates **`cf-turnstile-response`**.
4. Submit a valid message and confirm a successful JSON response.
5. Confirm that only one email is received.

### Negative Tests

1. Submit without **`cf-turnstile-response`**: it must fail and no email must be sent.
2. Submit an invalid token: it must fail.
3. Reuse the same token: it must fail because tokens are single-use.
4. Submit a token with an action other than **`contact`**: it must fail.
5. Submit a token issued for another hostname: it must fail.
6. Submit a message with fewer than 20 characters: it must fail.
7. Submit an alphanumeric string without spaces: it must fail when the rule applies.
8. Submit an email from the blacklist: it must not generate an email and must not reveal the block.
9. Remove JavaScript and make a direct POST: the server must continue rejecting the request without a valid token.
10. Simulate a Cloudflare error or timeout: no email must be sent.

### Operational Tests

- Review LogBox for configuration errors, unsuccessful HTTP responses, and Cloudflare rejections.
- Verify that tokens or secret keys do not appear in logs.
- Test behind the actual proxy and confirm hostname/IP.
- Test staging with its hostname registered in Cloudflare.
- Confirm that legacy forms continue using their intended mechanism and do not accidentally depend on Turnstile.

## 15. Limitations and Recommended Improvements

The current implementation protects the flow against bots and automated input, but it is not a complete solution against volumetric abuse. For a higher-risk site, it is recommended to add:

- Rate limiting by IP, email, and time window at the edge or reverse proxy.
- CSRF token for authenticated endpoints or flows where the browser maintains a sensitive session.
- Length limits for all fields before building the email.
- Email normalization before checking the blacklist.
- A clear policy for trusted headers at the proxy.
- Rejection-rate metrics and alerts without storing sensitive data.
- Secret-key rotation and a revocation procedure.
- A CSP policy compatible with **`https://challenges.cloudflare.com`**.

These improvements are complementary. They do not replace server-side Turnstile verification, **`action`** and hostname checks, or message validation.

## 16. Summary for a New Implementation

The minimum reproducible recipe is:

1. Create the Turnstile widget and register the hostname.
2. Store the site key and secret key as per-site settings.
3. Load **`api.js`** only when the site key exists.
4. Explicitly render Turnstile with a dedicated action.
5. Send the token in a known field, for example **`cf-turnstile-response`**.
6. Verify the token from the server using **`/siteverify`**.
7. Require correct **`success`**, **`action`**, and **`hostname`** values.
8. Perform email, blacklist, and content validation on the server.
9. Send email only after all validations.
10. Test positive cases, invalid tokens, replay, incorrect hostname/action, and direct POST requests.
