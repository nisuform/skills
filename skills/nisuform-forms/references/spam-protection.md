# Spam protection

Every submission gets these checks with no setup: rate limits, the `_gotcha` honeypot, the `_elapsed` fill time, link spam detection and duplicate detection. Link spam and duplicates go to the form's spam folder, which never sends notifications.

The extra protections below are set per form in **Settings**, **Spam protection**, or with the MCP `update_form` tool. All work on every plan.

## Order of operations for a captcha

The form starts rejecting every submission without a valid token as soon as the captcha is turned on. So:

1. Add the widget to the site.
2. Then turn on the captcha with the site key and secret key.

The secret key is write only. Never put it in site code. Only the site key goes in the page.

## Cloudflare Turnstile

The user creates a widget in the Cloudflare dashboard under **Turnstile**, adds every hostname the form runs on, like `example.com` and `www.example.com`, and copies the site key and secret key.

With MCP: `update_form` with `captchaProvider: "turnstile"`, `captchaSiteKey`, `captchaSecretKey` and `captchaEnabled: true`, after the widget is live on the site. After that, `get_embed_code` includes the widget.

Plain HTML: load the script once and put the widget inside the `<form>`, so its hidden `cf-turnstile-response` field is sent.

```html
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>

<form action="https://api.nisuform.com/s/YOUR_FORM_KEY" method="POST">
  <input type="email" name="email" required>
  <textarea name="message" required></textarea>
  <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display:none">
  <div class="cf-turnstile" data-sitekey="YOUR_TURNSTILE_SITE_KEY"></div>
  <button type="submit">Send</button>
</form>
```

With `fetch`, post the whole `FormData` so the token goes along. A token works once, so reset the widget after a failed submit.

```js
const result = await response.json()
if (!result.ok && result.error === 'captcha_failed') {
  window.turnstile?.reset()
}
```

In React and other single-page apps, the script only renders `.cf-turnstile` elements that exist when it loads. Render the widget when the component mounts:

```jsx
import { useEffect, useRef } from 'react'

const TURNSTILE_SCRIPT = 'https://challenges.cloudflare.com/turnstile/v0/api.js'

export function useTurnstile() {
  const widget = useRef(null)

  useEffect(() => {
    if (window.turnstile) {
      if (!widget.current.childElementCount) window.turnstile.render(widget.current)
    } else if (!document.querySelector(`script[src="${TURNSTILE_SCRIPT}"]`)) {
      const script = document.createElement('script')
      script.src = TURNSTILE_SCRIPT
      script.async = true
      document.head.append(script)
    }
  }, [])

  return widget
}
```

Then render `<div ref={widget} className="cf-turnstile" data-sitekey="YOUR_TURNSTILE_SITE_KEY" />` inside the form, and call `window.turnstile?.reset(widget.current)` after a failed submit.

## Google reCAPTCHA

Turnstile is quicker to set up. Use reCAPTCHA when the user already has keys.

| Key type | Visitors see | Use with |
| --- | --- | --- |
| Checkbox (v2) | "I'm not a robot" | Plain HTML, the styled embed, generated code |
| Score (v3) | Nothing | Your own JavaScript only. Scores below 0.5 are rejected. |

With MCP: `update_form` with `captchaProvider: "recaptcha"`, `captchaSiteKey`, `captchaSecretKey` and `captchaEnabled: true`.

v2 checkbox, inside the form:

```html
<script src="https://www.google.com/recaptcha/api.js" async defer></script>
<div class="g-recaptcha" data-sitekey="YOUR_RECAPTCHA_SITE_KEY"></div>
```

v3, requesting the token at submit time because tokens expire after two minutes:

```html
<script src="https://www.google.com/recaptcha/api.js?render=YOUR_RECAPTCHA_SITE_KEY"></script>

<form id="contact" action="https://api.nisuform.com/s/YOUR_FORM_KEY" method="POST">
  <input type="email" name="email" required>
  <textarea name="message" required></textarea>
  <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display:none">
  <input type="hidden" name="g-recaptcha-response">
  <button type="submit">Send</button>
</form>

<script>
  const form = document.getElementById('contact')
  form.addEventListener('submit', (event) => {
    event.preventDefault()
    grecaptcha.ready(async () => {
      form.elements['g-recaptcha-response'].value = await grecaptcha.execute('YOUR_RECAPTCHA_SITE_KEY', { action: 'submit' })
      form.submit()
    })
  })
</script>
```

After a failed `fetch` with `captcha_failed`, call `grecaptcha.reset()` for v2.

hCaptcha is not verified by Nisuform. Replace it with Turnstile or reCAPTCHA.

## When every submission fails with `captcha_failed`

- The widget or the `g-recaptcha-response` input is outside the `<form>`.
- The site key in the page and the keys saved on the form belong to different widgets.
- The site's hostname isn't in the widget's hostname list (Cloudflare) or the key's domain list (Google).
- With `fetch`, the body isn't the form's `FormData`, so the token is missing.
- A checkbox key is used with `grecaptcha.execute`, or a score key with the checkbox.

## Allowed websites

Lists the origins allowed to submit, like `https://example.com` and `https://www.example.com`. Other sites get `403` with `origin_not_allowed`. Only scheme, host and port count, and `www` is a separate site. Add `http://localhost:3000` (or the dev server's port) while developing. With MCP: `update_form` with `allowedOrigins`.

This stops other websites from posting to the form, not scripts, which can omit the `Origin` header. Combine it with a captcha for that.

## Blocked IPs and countries

Exact IPv4 or IPv6 addresses (no ranges), and two-letter country codes. Blocked senders get a normal success response with no `submissionId`, and nothing is stored. With MCP: `update_form` with `blockedIps` or `blockedCountries`.

## Rate limits

| Limit | Free | Pro and Business |
| --- | --- | --- |
| One IP, all forms | 30 a minute | 300 a minute |
| One IP, one form | 5 a minute | 50 a minute |
| One form, everyone | 200 an hour | 2,000 an hour |

Over a limit, the response is `429` with `rate_limited`. Browser JavaScript can't read `Retry-After`, so tell visitors to wait a minute.

## Abused endpoint

If a form's key is being abused, rotate it in **Settings**, **Endpoint**, or with the REST API (`POST /v1/forms/{id}/rotate-key`). The old endpoint stops working at once, so update the site in the same change.
