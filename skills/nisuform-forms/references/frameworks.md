# Framework recipes

Every recipe follows the submit contract in [SKILL.md](../SKILL.md): `POST` to the endpoint, a `name` on every input, the hidden `_gotcha` honeypot, `_elapsed` when JavaScript submits, `Accept: application/json`, and reading `ok` and `message` from the JSON body. Replace `YOUR_FORM_KEY` with the real key, and the fields with the form's own.

## Contents

- [TypeScript (React and Next.js)](#typescript-react-and-nextjs)
- [Next.js](#nextjs)
- [Vue](#vue)
- [Nuxt](#nuxt)
- [Svelte and SvelteKit](#svelte-and-sveltekit)
- [Astro](#astro)
- [Site builders](#site-builders)
- [Hidden fields](#hidden-fields)
- [Checkbox groups](#checkbox-groups)
- [Redirects after submit](#redirects-after-submit)
- [Submitting from a server](#submitting-from-a-server)

## TypeScript (React and Next.js)

The React component in SKILL.md only needs types on the handler and the response:

```tsx
type SubmitResult = { ok: true; submissionId?: string; warning?: 'near_limit' } | { ok: false; error: string; message: string }

async function onSubmit(event: React.FormEvent<HTMLFormElement>) {
  event.preventDefault()
  const data = new FormData(event.currentTarget)
  data.set('_elapsed', String(Date.now() - shownAt.current))
  const response = await fetch(ENDPOINT, { method: 'POST', headers: { Accept: 'application/json' }, body: data })
  const result: SubmitResult = await response.json()
}
```

## Next.js

- App Router: put `'use client'` on the first line of the form component, and render it from any page. The page can stay a server component.
- Pages Router: the same component works without `'use client'`.
- Works with `output: 'export'`, because the browser posts straight to Nisuform.
- Don't add `app/api/contact/route.ts`, a server action, Resend or Nodemailer, or a `NEXT_PUBLIC_` variable for the key.

```jsx
import { ContactForm } from './ContactForm'

export default function ContactPage() {
  return <ContactForm />
}
```

## Vue

```vue
<script setup>
import { ref } from 'vue'

const endpoint = 'https://api.nisuform.com/s/YOUR_FORM_KEY'
const shownAt = Date.now()
const status = ref('idle')
const error = ref('')

async function onSubmit(event) {
  status.value = 'sending'
  const data = new FormData(event.target)
  data.set('_elapsed', String(Date.now() - shownAt))
  try {
    const response = await fetch(endpoint, { method: 'POST', headers: { Accept: 'application/json' }, body: data })
    const result = await response.json()
    if (result.ok) {
      status.value = 'sent'
      return
    }
    error.value = result.message
  } catch {
    error.value = 'Something went wrong. Please try again.'
  }
  status.value = 'error'
}
</script>

<template>
  <p v-if="status === 'sent'" role="status">Thanks! We got your message.</p>
  <form v-else @submit.prevent="onSubmit">
    <label for="email">Email</label>
    <input id="email" type="email" name="email" autocomplete="email" required />
    <label for="message">Message</label>
    <textarea id="message" name="message" rows="5" required></textarea>
    <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display: none" />
    <button type="submit" :disabled="status === 'sending'">Send</button>
    <p v-if="status === 'error'" role="alert">{{ error }}</p>
  </form>
</template>
```

## Nuxt

Use the Vue component as `components/ContactForm.vue`. Nuxt imports `ref` automatically, so the `import { ref } from 'vue'` line is optional. Render `<ContactForm />` from a page. The submit runs in the browser, so no `server/api` route or runtime config is needed.

## Svelte and SvelteKit

Svelte 5 runes:

```svelte
<script>
  const endpoint = 'https://api.nisuform.com/s/YOUR_FORM_KEY'
  const shownAt = Date.now()
  let status = $state('idle')
  let error = $state('')

  async function onsubmit(event) {
    event.preventDefault()
    status = 'sending'
    const data = new FormData(event.currentTarget)
    data.set('_elapsed', String(Date.now() - shownAt))
    try {
      const response = await fetch(endpoint, { method: 'POST', headers: { Accept: 'application/json' }, body: data })
      const result = await response.json()
      if (result.ok) {
        status = 'sent'
        return
      }
      error = result.message
    } catch {
      error = 'Something went wrong. Please try again.'
    }
    status = 'error'
  }
</script>

{#if status === 'sent'}
  <p role="status">Thanks! We got your message.</p>
{:else}
  <form {onsubmit}>
    <label for="email">Email</label>
    <input id="email" type="email" name="email" autocomplete="email" required />
    <label for="message">Message</label>
    <textarea id="message" name="message" rows="5" required></textarea>
    <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display: none" />
    <button type="submit" disabled={status === 'sending'}>Send</button>
    {#if status === 'error'}
      <p role="alert">{error}</p>
    {/if}
  </form>
{/if}
```

For Svelte 4, use plain `let` variables instead of `$state`, and `on:submit={onsubmit}` instead of `{onsubmit}`. In SvelteKit, put the component in `src/lib/` and render it from `+page.svelte`. Don't add a form action or `+page.server.js`.

## Astro

A plain form needs zero client JavaScript:

```astro
---
const endpoint = 'https://api.nisuform.com/s/YOUR_FORM_KEY'
---

<form action={endpoint} method="POST">
  <label>
    Email
    <input type="email" name="email" required />
  </label>
  <label>
    Message
    <textarea name="message" rows="5" required></textarea>
  </label>
  <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display: none" />
  <button type="submit">Send</button>
</form>
```

To submit without a page reload, either add the inline script from SKILL.md, or render the React, Vue or Svelte component as an island with the matching integration (`@astrojs/react`, `@astrojs/vue`, `@astrojs/svelte`) and `client:load`:

```astro
---
import { ContactForm } from '../components/ContactForm.jsx'
---

<ContactForm client:load />
```

## Site builders

Webflow, Framer, WordPress, Carrd, Ghost, Shopify and Squarespace take the styled embed: with MCP tools, `get_embed_code` with `framework: "html"` and `style: "styled"`. Otherwise the user copies it from the form's **Setup** tab, **Site builder**. Then:

| Builder | Where to paste |
| --- | --- |
| Webflow | A **Code Embed** element. Or keep Webflow's form block: set **Action** to the endpoint, **Method** to `POST`, and name every input. |
| Framer | An **Embed** component set to **HTML**. |
| WordPress | A **Custom HTML** block. No plugin needed. If WordPress strips the `<script>`, the form still works and shows the Nisuform thank-you page. |
| Others | Any HTML or embed block. If scripts are blocked, use the plain HTML form, which needs no JavaScript. |

The embed is a copy of the form at the time it was copied. After changing fields, the design, Turnstile or the redirect URL, copy it again.

## Hidden fields

Hidden inputs are stored like any other answer, which is useful to tell forms or pages apart:

```html
<input type="hidden" name="source" value="pricing-page">
```

Nisuform already records the page each submission came from, so a hidden field for the URL isn't needed.

## Checkbox groups

```html
<fieldset>
  <legend>Topics</legend>
  <label><input type="checkbox" name="topics[]" value="Design"> Design</label>
  <label><input type="checkbox" name="topics[]" value="Development"> Development</label>
</fieldset>
```

Saved as `topics: "Design, Development"`. Post `FormData`, not `Object.fromEntries`, which keeps only one value. A single consent checkbox should use `value="yes"`.

## Redirects after submit

Redirects only apply to browser form posts, never to `fetch` requests that ask for JSON. With `fetch`, redirect in your own code after `result.ok`.

For HTML forms on Pro, either set **Redirect** in the form's settings (with MCP: `update_form` with `completionMode: "redirect"` and `redirectUrl`), or choose the destination per form:

```html
<input type="hidden" name="_redirect" value="https://example.com/thanks">
```

`_redirect` must be an absolute URL on the same site as the page that sent the form, or on one of the form's allowed websites. Otherwise it is ignored. On Free, visitors see the Nisuform thank-you page.

## Submitting from a server

Posting from a backend works, but prefer the visitor's browser:

- Rate limits are per IP, so every request from one server shares its limit: 5 a minute per form and 30 a minute across forms on Free, 50 and 300 on Pro and Business.
- Nisuform records the server's IP, location and user agent instead of the visitor's.
- Allowed websites don't apply, because servers send no `Origin` or `Referer`.
