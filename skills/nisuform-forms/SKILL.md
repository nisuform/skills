---
name: nisuform-forms
description: Add a working form (contact, waitlist, feedback, RSVP, signup, quote request or any custom form) to a website with Nisuform, a form backend that needs no server code. Use when the user wants a form that emails its submissions, a contact form for a static site, or a form in HTML, React, Next.js, Vue, Nuxt, Svelte, SvelteKit, Astro, Webflow, Framer or WordPress that posts to https://api.nisuform.com/s/{key}. Also use to fix a Nisuform form that isn't sending, and to add Cloudflare Turnstile, reCAPTCHA, file uploads, redirects or the _gotcha honeypot.
license: MIT
---

# Nisuform forms

Nisuform gives every form one public endpoint. The site's form posts straight to it, and Nisuform filters spam, stores the submission, emails the owner and forwards it to integrations.

```text
POST https://api.nisuform.com/s/{key}
```

The site needs no API route, server action, serverless function, environment variable or API key. The form key is public by design: it only lets people send submissions to that one form.

## 1. Get the endpoint

Never invent a form key. In this order:

1. **Nisuform MCP tools are available** (`list_forms`, `create_form`, `get_embed_code`): call `list_forms` and reuse a form that fits. Otherwise call `create_form` with a name and the closest template (`contact`, `waitlist`, `feedback`, `rsvp`, or `custom` for no fields). Use the `endpoint` it returns. Create one Nisuform form per form on the site.
2. **The user gave an endpoint or key**: use it. A key alone becomes `https://api.nisuform.com/s/{key}`.
3. **Neither**: write the code with `https://api.nisuform.com/s/YOUR_FORM_KEY` in exactly one place, and tell the user to create a free form at https://app.nisuform.com and paste the endpoint from the form's **Setup** tab. To let you create forms directly next time, they can connect the MCP server: `claude mcp add --transport http nisuform https://mcp.nisuform.com/mcp`.

## 2. Pick the approach

| Site | Approach |
| --- | --- |
| Plain HTML, Astro page, Hugo, Jekyll, Eleventy, server-rendered templates | HTML form with `action` set to the endpoint. Works with JavaScript off. |
| React, Next.js, Vue, Nuxt, Svelte, SvelteKit, or "no page reload" | A component that posts with `fetch` and reads the JSON response. |
| Webflow, Framer, WordPress, Carrd, Ghost, Shopify | The styled embed pasted into the builder's HTML or embed block. See [frameworks.md](references/frameworks.md). |

With MCP tools available, `get_embed_code` returns ready code for `html`, `react`, `next`, `vue`, `svelte` or `astro`. Use `style: "styled"` for the Nisuform-designed form, and add it exactly as returned. Use `style: "plain"`, or write the markup yourself, when the form should match the site's own design. When writing it yourself, reuse the site's existing input, button and label components and its styling approach. No form library or new dependency is needed.

## 3. Follow the submit contract

1. `method="POST"`. The endpoint only accepts `POST`. Opening it in a browser shows a `405` page, which is expected.
2. **Every input has a `name`.** The browser never sends inputs without one, or disabled ones (use `readonly`).
3. **Match the form's field names** when it has fields. Templates use: contact `name`, `email`, `message`. Waitlist `email`. Feedback `rating` (1 to 5), `comment`, `email`. RSVP `name`, `email`, `attending`, `guests` (0 to 10), `note`. With MCP, `get_form` lists them as `fields[].id`. Extra names are still stored, never rejected.
4. **Name the email input `email`** with `type="email"`. Nisuform sets the notification's Reply-To from it, and auto-replies go to it.
5. **Keep the honeypot**, hidden with inline CSS and never with `type="hidden"`:
   ```html
   <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display:none">
   ```
   In JSX: `<input type="text" name="_gotcha" tabIndex={-1} autoComplete="off" style={{ display: 'none' }} />`. A filled honeypot drops the submission silently.
6. **With JavaScript, send `_elapsed`**: milliseconds since the form was shown. Under 2000 is dropped as a bot. Leave it out of forms without JavaScript: a missing value skips the check.
7. **`fetch` sends `Accept: application/json`** and no other custom header. Add `Content-Type: application/json` only for a JSON body. With `FormData`, set no `Content-Type`. `Authorization` or `X-` headers break the CORS preflight. Don't send credentials.
8. **Read the JSON body**, not only the status:
   - `{ "ok": true, "submissionId": "..." }`: stored.
   - `{ "ok": true }` with no `submissionId`: dropped on purpose (honeypot, `_elapsed` under 2 seconds, or a blocked IP or country). Show success anyway.
   - `{ "ok": false, "error": "...", "message": "..." }`: rejected. Show `message` to the visitor as is, it is written for people. Branch on `error`, which never changes.
9. **Use browser validation** (`required`, `type="email"`, `min`, `max`). Nisuform never rejects a submission for failing field checks. It stores it and flags it in the inbox.
10. **Don't put real data in names that start with `_`.** Only `_gotcha`, `_elapsed` and `_redirect` mean something. Others, like `_subject`, `_next`, `_replyto` or `_cc` from other services, are ignored. Those settings live in the dashboard.
11. **Write the endpoint as a plain string**, in one constant. Don't move it into an environment variable or proxy it through a server.

Fields named `topics[]`, and checkboxes that share a name, are saved as one comma-separated answer. `FormData` keeps them all, while `Object.fromEntries(new FormData(form))` keeps only the last one, so post `FormData` when the form has checkbox groups, files or a captcha.

## 4. Reference code

### HTML

```html
<form id="contact-form" action="https://api.nisuform.com/s/YOUR_FORM_KEY" method="POST">
  <label for="name">Name</label>
  <input id="name" type="text" name="name" autocomplete="name" required>

  <label for="email">Email</label>
  <input id="email" type="email" name="email" autocomplete="email" required>

  <label for="message">Message</label>
  <textarea id="message" name="message" rows="5" required></textarea>

  <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display:none">
  <button type="submit">Send</button>
  <p id="form-status" role="status"></p>
</form>
```

After submitting, the browser shows Nisuform's thank-you page with a link back to the site. On Pro, the form's settings can show a custom message or `303` redirect to the site's own page.

To keep visitors on the page while staying functional without JavaScript, add this script after the same form:

```html
<script>
  const form = document.getElementById('contact-form')
  const status = document.getElementById('form-status')
  const shownAt = Date.now()

  form.addEventListener('submit', async (event) => {
    event.preventDefault()
    const button = form.querySelector('[type="submit"]')
    button.disabled = true
    const data = new FormData(form)
    data.set('_elapsed', String(Date.now() - shownAt))
    try {
      const response = await fetch(form.action, { method: 'POST', headers: { Accept: 'application/json' }, body: data })
      const result = await response.json()
      if (result.ok) {
        form.reset()
        status.textContent = 'Thanks! We got your message.'
      } else {
        status.textContent = result.message
      }
    } catch {
      status.textContent = 'Something went wrong. Please try again.'
    }
    button.disabled = false
  })
</script>
```

### React

Add `'use client'` as the first line for the Next.js App Router. Nothing else changes, and no API route or server action is needed.

```jsx
import { useRef, useState } from 'react'

const ENDPOINT = 'https://api.nisuform.com/s/YOUR_FORM_KEY'

export function ContactForm() {
  const shownAt = useRef(Date.now())
  const [status, setStatus] = useState('idle')
  const [error, setError] = useState('')

  async function onSubmit(event) {
    event.preventDefault()
    setStatus('sending')
    const data = new FormData(event.currentTarget)
    data.set('_elapsed', String(Date.now() - shownAt.current))
    try {
      const response = await fetch(ENDPOINT, {
        method: 'POST',
        headers: { Accept: 'application/json' },
        body: data,
      })
      const result = await response.json()
      if (result.ok) {
        setStatus('sent')
        return
      }
      setError(result.message)
    } catch {
      setError('Something went wrong. Please try again.')
    }
    setStatus('error')
  }

  if (status === 'sent') {
    return <p role="status">Thanks! We got your message.</p>
  }

  return (
    <form onSubmit={onSubmit}>
      <label htmlFor="name">Name</label>
      <input id="name" type="text" name="name" autoComplete="name" required />

      <label htmlFor="email">Email</label>
      <input id="email" type="email" name="email" autoComplete="email" required />

      <label htmlFor="message">Message</label>
      <textarea id="message" name="message" rows={5} required />

      <input type="text" name="_gotcha" tabIndex={-1} autoComplete="off" style={{ display: 'none' }} />
      <button type="submit" disabled={status === 'sending'}>
        {status === 'sending' ? 'Sending...' : 'Send'}
      </button>
      {status === 'error' && <p role="alert">{error}</p>}
    </form>
  )
}
```

Vue, Nuxt, Svelte, SvelteKit, Astro, TypeScript and site builders: [references/frameworks.md](references/frameworks.md).

## 5. Verify

- Re-read the code against the contract in step 3.
- With MCP tools, `send_test_submission` stores a sample marked as a test and fires the email and integrations. It doesn't count toward the monthly limit.
- A live test exercises the real endpoint, but it is a real submission: it emails the owner and counts toward the monthly limit (100 on Free). Ask before sending one:
  ```sh
  curl https://api.nisuform.com/s/YOUR_FORM_KEY \
    -H 'Content-Type: application/json' \
    -H 'Accept: application/json' \
    -d '{"name":"Test","email":"test@example.com","message":"Hello from a test"}'
  ```
  `{"ok":true,"submissionId":"..."}` means it works.

When you finish, tell the user where the endpoint is used, and point them to the form's **Settings** in https://app.nisuform.com for notifications, allowed websites and spam protection.

## Plans

Don't build on a Pro feature unless the user is on Pro or agrees to upgrade.

| Free | Pro and Business only |
| --- | --- |
| 100 submissions a month, 3 forms, email notifications to the owner with a custom subject, spam filtering, Turnstile and reCAPTCHA, allowed websites, inbox and CSV export, styled embed | 5,000 (Pro) or 25,000 (Business) submissions a month, unlimited forms, file uploads, redirects and `_redirect`, custom thank-you and closed messages, auto-replies, notifications to any address, webhooks, Slack, Discord, Telegram and Google Sheets |

## More

- [references/frameworks.md](references/frameworks.md): Vue, Nuxt, Svelte, SvelteKit, Astro, TypeScript, site builders, hidden fields, redirects.
- [references/spam-protection.md](references/spam-protection.md): Cloudflare Turnstile, Google reCAPTCHA v2 and v3, allowed websites, rate limits.
- [references/file-uploads.md](references/file-uploads.md): accepting images and PDFs (Pro).
- [references/troubleshooting.md](references/troubleshooting.md): every error code, missing submissions, CORS errors, missing emails.
- Full docs for agents: https://nisuform.com/llms-full.txt. Any docs page as Markdown: `https://nisuform.com/docs/<page>.md`.
