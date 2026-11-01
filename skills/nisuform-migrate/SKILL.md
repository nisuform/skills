---
name: nisuform-migrate
description: Move a website's existing forms to Nisuform from Formspree, Web3Forms, FormSubmit, Netlify Forms, Getform, Forminit, Formcarry, Basin, Formspark, Static Forms, EmailJS, or a custom email route (PHP mail, Nodemailer, Resend or SendGrid in an API route or server action). Use when the user wants to switch form backends, replace a form service, or remove server code that only sends form emails, and when the code contains formspree.io, api.web3forms.com, formsubmit.co, data-netlify, getform.io, formcarry.com, usebasin.com or similar.
license: MIT
---

# Migrate forms to Nisuform

The goal: every form on the site posts straight to its own Nisuform endpoint, `https://api.nisuform.com/s/{key}`, follows the submit contract in the `nisuform-forms` skill, and the old provider's code is gone. Field names stay the same, so the user's data and email templates keep making sense.

## 1. Find every form

Search the codebase for these patterns, skipping dependencies, build output and lockfiles:

```text
formspree\.io|@formspree|api\.web3forms\.com|@web3forms|formsubmit\.co|data-netlify|netlify-honeypot|name="form-name"|getform\.io|forminit\.com|formcarry\.com|usebasin\.com|submit-form\.com|staticforms\.xyz|formbackend\.com|emailjs
```

```sh
grep -rnE 'formspree\.io|@formspree|api\.web3forms\.com|@web3forms|formsubmit\.co|data-netlify|netlify-honeypot|name="form-name"|getform\.io|forminit\.com|formcarry\.com|usebasin\.com|submit-form\.com|staticforms\.xyz|formbackend\.com|emailjs' . \
  --exclude-dir=node_modules --exclude-dir=.git --exclude-dir=dist --exclude-dir=build --exclude-dir=.next --exclude-dir=.astro \
  --exclude=package-lock.json --exclude=pnpm-lock.yaml --exclude=yarn.lock --exclude=bun.lock
```

Also look for server code that exists only to email a form: API routes or server actions calling `nodemailer`, `resend`, `@sendgrid/mail` or `mail()`, and handlers posting to them.

List each form with its file, provider, fields and special behavior (redirect, captcha, file inputs, `fetch` handling) before changing anything.

## 2. Get one endpoint per form

One Nisuform form per form on the site, so each keeps its own inbox and notifications.

- With the Nisuform MCP tools, `list_forms` first, then `create_form` with a fitting name and template (`contact`, `waitlist`, `feedback`, `rsvp`, or `custom` to keep the site's own field names without a schema). Use the returned `endpoint`.
- Otherwise ask the user for each endpoint, or use `https://api.nisuform.com/s/YOUR_FORM_KEY` in one constant per form and list the placeholders at the end.
- Free allows 3 forms per team. More than that needs Pro.

## 3. Rewrite each form

Change the URL, keep the field names, then clean up the provider's fields:

Provider fields without an underscore, like `subject` or `redirect`, would be stored as answers in Nisuform, so remove them rather than leaving them in place.

| Old field or attribute | In Nisuform |
| --- | --- |
| A honeypot such as `_gotcha`, `_honey`, `_honeypot`, `botcheck` or `netlify-honeypot="bot-field"` | One `<input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display:none">`. Remove the old honeypot. |
| An access key hidden input, such as `access_key` or `accessKey` | Remove. The key is in the endpoint URL. |
| `_replyto`, `replyto`, `replyTo` | Name the email input `email`. Reply-To is set from it automatically. |
| `_subject` or a `subject` hidden input | Remove. Set the subject in **Settings**, **Email**, with `{{form.name}}` and `{{field.<name>}}` placeholders (MCP: `update_form` with `notifySubject`). |
| `_next`, `redirect` or another redirect field | Pro: the form's redirect setting (`completionMode: "redirect"`, `redirectUrl`) or a `_redirect` hidden input on the same site. Free: the Nisuform thank-you page, or `fetch` and redirect in code. |
| Netlify's `action="/thanks"` | The action becomes the endpoint. Handle the thank-you page as in the row above. |
| `_cc`, `ccemail` or other copy fields | Remove. Pro sends notifications to any one address (a group address works), or connect Slack, Discord or Telegram. |
| Autoresponse fields such as `_autoresponse` | Remove. Pro has auto-replies in **Settings**, **Email**. |
| Template, captcha toggle and sender name fields, such as `_template`, `_captcha` or `from_name` | Remove. |
| Netlify's `form-name` hidden input, `data-netlify`, `netlify` attribute and `data-netlify-recaptcha` | Remove all of them. |
| hCaptcha widget and `h-captcha-response` | Nisuform doesn't verify hCaptcha. Remove it, and offer Turnstile or reCAPTCHA instead. |
| reCAPTCHA or Turnstile widget | Keep the widget, and save the same keys on the Nisuform form after deploying (see the `nisuform-forms` skill). |
| `action="mailto:..."` or an email address in the endpoint URL | The endpoint. The email address no longer appears in the page. |

Any other field name starting with `_` is ignored by Nisuform. Rename it without the underscore if its value matters.

### JavaScript submissions

Replace the provider's SDK or custom request with the `fetch` pattern from the `nisuform-forms` skill: `FormData` body, `_elapsed`, `Accept: application/json`, then read `ok` and show `message` on failure.

- `@formspree/react` (`useForm`, `ValidationError`): replace with a plain submit handler and the component's own state.
- `@web3forms/react`, `@emailjs/browser` (`emailjs.sendForm`): replace the call. Remove the public keys and service or template IDs they needed.
- Netlify's JS pattern that posts URL-encoded data to `/` with `form-name`: post to the Nisuform endpoint instead, and drop the hidden static form that only existed so Netlify could detect it.
- Old success checks like `response.ok` alone or `state.succeeded` become `result.ok`.

### Server email routes

When an API route, server action or PHP script exists only to email form contents, the form can post straight to Nisuform instead. Ask before deleting server code. After agreement, remove the route, its handler, and the packages and environment variables used only by it (like `RESEND_API_KEY` or `SMTP_*`), and tell the user which variables they can delete from their host. Keep the route when it does more than send email, such as writing to the site's database. In that case, consider a Nisuform webhook (the `nisuform-webhooks` skill) to feed it.

## 4. Remove what's left

- Uninstall provider packages that nothing imports anymore (`@formspree/react`, `@web3forms/react`, `@emailjs/browser`, `nodemailer`) with the project's package manager.
- Remove provider environment variables from `.env.example` and config, and provider scripts from HTML heads (for example the hCaptcha script).
- Leave unrelated Netlify configuration alone. Only the forms move.

## 5. Move the settings

Settings that lived in the markup or the old dashboard move to the Nisuform form, in **Settings** or with MCP `update_form`:

- Allowed websites (`allowedOrigins`) for the production domain, with and without `www`.
- Notification subject (`notifySubject`), and on Pro the recipient (`notifyEmail`), sender name, auto-reply and redirect.
- Captcha keys, once the widget is live.
- File uploads (`allowUploads`, Pro) for forms with file inputs, plus `enctype="multipart/form-data"`.

## 6. Verify and report

- Re-read each rewritten form against the submit contract: `POST`, every input named, `_gotcha`, `_elapsed` with JavaScript, `Accept: application/json`, `ok` and `message` handled.
- Search again with the step 1 command. Only intentional leftovers should remain.
- Run the project's build or type check.
- With MCP tools, `send_test_submission` on each form checks notifications.

Then tell the user:

- Which forms moved, and to which Nisuform form.
- Any placeholders that still need a real endpoint.
- Settings they must set in the dashboard, especially Pro-only ones.
- That past submissions stay with the old provider: export them there before closing that account.
- To keep the old account until the new forms have received real submissions in production.
