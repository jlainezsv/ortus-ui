---
title: "reCAPTCHA v3"
summary: Invisible score-based spam protection for BoxLang contact forms.
---

## Overview

The BoxLang contact form uses Google reCAPTCHA v3 to protect against spam and automated submissions.

reCAPTCHA v3 runs in the background and returns a risk score from `0.0` to `1.0`. The server validates this score before processing the form submission.

**Current scope:** BoxLang contact form only (`/bxcontactus`).

---

## How It Works

1. The form requests a reCAPTCHA token from Google.
2. The token is submitted with the form.
3. The server sends the token to Google's verification API.
4. The response is validated for:
   - Successful verification
   - Expected action (`contact`)
   - Minimum score
5. If validation passes, the form is processed.
6. If validation fails, the submission is rejected.

---

## Configuration

Administrators configure reCAPTCHA through **ContentBox → Settings**.

| Setting | Purpose | Default |
|---------|---------|---------|
| `cb_recaptcha_site_key` | Public Google reCAPTCHA site key | Empty |
| `cb_recaptcha_secret_key` | Server-side Google reCAPTCHA secret key | Empty |
| `cb_recaptcha_min_score` | Sets how strict reCAPTCHA is. Configurable in ContentBox settings; defaults to `0.5` | `0.5` |

### Site Key

Used by the frontend to initialize reCAPTCHA.

If it is empty, reCAPTCHA is not rendered.

### Secret Key

Used only on the server to verify tokens with Google.

**Never expose the secret key in client-side code, HTML, or logs.**

### Minimum Score

`cb_recaptcha_min_score` controls how strict the reCAPTCHA validation is.

The value represents the minimum risk score a submission must reach to pass verification:

- A **lower value** is less strict and allows more submissions to pass.
- A **higher value** is more strict and requires a higher score.
- The value must be between `0.0` and `1.0`.

```text
Score >= minScore → Pass
Score < minScore  → Fail
```

The default is `0.5`.

If `cb_recaptcha_min_score` is not configured, invalid, or not greater than `0`, the system falls back to `0.5`.

The setting can be configured per site, with a global setting used as a fallback.

Adjust the value based on the site's reCAPTCHA score distribution and observed false positives or spam.

---

## Google Configuration

Create a **reCAPTCHA v3** key pair in the Google reCAPTCHA Admin Console.

Configure all domains that will use the contact form, including staging domains when applicable.

The site key is public. The secret key must remain server-side.

---

## Action Validation

The contact form uses the action:

```text
contact
```

The action must match between the frontend and server.

A mismatch causes verification to fail.

---

## Server-Side Verification

The `RecaptchaV3Service` handles communication with Google's verification API.

**File:**

```text
modules_app/contentbox-custom/models/recaptcha/RecaptchaV3Service.cfc
```

The main method is:

```cfc
verify(
    token,
    action,
    minScore
)
```

It validates:

- Google verification status
- reCAPTCHA action
- Score against `minScore`

The service returns a result containing the verification status, score, action, hostname, and error information.

---

## Frontend Integration

The `RecaptchaV3Element` widget handles the client-side integration.

**File:**

```text
modules_app/contentbox-custom/_themes/boxlang/widgets/RecaptchaV3Element.cfc
```

It:

1. Loads the Google reCAPTCHA API.
2. Generates a token when the form is submitted.
3. Stores the token in `g-recaptcha-response`.
4. Submits the form to `/bxcontactus`.
5. Displays the resulting success or error message.

reCAPTCHA v3 does not display a visible CAPTCHA challenge.

---

## Error Handling

Users receive a generic message when reCAPTCHA validation fails:

```text
Invalid security code. Please try again.
```

Detailed failure information is logged server-side.

Common causes include:

| Problem | Check |
|---------|-------|
| Missing site key | `cb_recaptcha_site_key` |
| Missing secret key | `cb_recaptcha_secret_key` |
| Low score | `cb_recaptcha_min_score` |
| Action mismatch | Frontend and server both use `contact` |
| Invalid domain | Domain configured in Google reCAPTCHA |
| Invalid token | Token may be expired or already used |
| Google unavailable | Check network connectivity |

---

## Testing

Verify the following:

- The form loads without a visible CAPTCHA challenge.
- Submitting the form generates a reCAPTCHA token.
- Valid submissions are processed successfully.
- Low-score submissions are rejected.
- Invalid or missing tokens are rejected.
- Errors are recorded in the server logs.

---

## Security

- Keep `cb_recaptcha_secret_key` server-side.
- Validate the score on the server.
- Validate the reCAPTCHA action.
- Do not log tokens or secret keys.
- Use a fresh token for each submission.
