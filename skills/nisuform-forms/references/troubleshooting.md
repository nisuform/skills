# Troubleshooting

Start by separating Nisuform from the site. With MCP tools, `send_test_submission` checks the form's email and integrations without the site. If the test arrives, the problem is in how the site sends the form. Then look at the real request in the browser's network tab, or reproduce it with `curl` and `Accept: application/json` to see the JSON body.

## Error codes

Rejected submissions answer `{ "ok": false, "error": "...", "message": "..." }` with a matching status. Browser form posts get a page with the same reason.

| Status | `error` | Fix |
| --- | --- | --- |
| `400` | `too_many_files` | Send at most 5 files. |
| `402` | `quota_exceeded` | The team used its monthly submissions (100 on Free). Upgrade or wait for the 1st of the month, UTC. |
| `403` | `origin_not_allowed` | Add the site's exact origin, with or without `www`, to the form's allowed websites. |
| `403` | `captcha_failed` | See "Every submission fails with captcha_failed" in [spam-protection.md](spam-protection.md). |
| `404` | `form_not_found` | Wrong key, a rotated key or a deleted form. Copy the endpoint again. |
| `405` | `method_not_allowed` | The form uses `GET`, or the endpoint was opened in a browser. Use `method="POST"`. |
| `413` | `payload_too_large` | Over 200 KB, or 25 MB with uploads on. Large files need uploads turned on (Pro). |
| `413` | `attachment_too_large` | A file is over the per-file limit. |
| `415` | `unsupported_media_type` | Send JSON, URL-encoded or multipart. A JSON body must be valid JSON and an object. |
| `415` | `attachment_type_rejected` | Only JPEG, PNG, WebP, GIF and PDF. |
| `422` | `empty_submission` | Every field was empty, or no input had a `name`. |
| `423` | `form_closed_manual` | The form is paused. Resume it in **Settings**, **General**, or `update_form` with `status: "active"`. |
| `423` | `form_max_submissions` | Raise or clear **Maximum submissions**. |
| `423` | `form_auto_close` | Clear or move the **Close on** date. |
| `429` | `rate_limited` | Too many submissions from one IP or to one form. Wait a minute. |
| `500` | `upload_failed`, `internal_error` | Retry. If it repeats, contact support@nisuform.com with the `x-request-id` response header. |

## The response is `200` but the submission isn't in the inbox

- `{ "ok": true }` without a `submissionId` means it was dropped on purpose: `_gotcha` had a value, `_elapsed` was under 2000, or the IP or country is blocked. A honeypot hidden with a class that didn't load, or with `type="hidden"`, gets filled by autofill and password managers. Hide it with inline `style="display:none"`. Make sure `_elapsed` is measured from when the form was shown, not from the submit click.
- With a `submissionId`, check the inbox's **Spam** filter: more than 5 links, link markup like `<a href=` or `[url=`, or an exact repeat within 24 hours lands there.
- Check that the endpoint in the code matches the form and team you are looking at.

## Some answers are missing

- The input has no `name`, or it is `disabled`.
- The name starts with `_`. Rename it.
- `Object.fromEntries(new FormData(form))` kept only the last checkbox of a group. Post `FormData`.
- File inputs need `enctype="multipart/form-data"`, and uploads turned on (Pro).

## The browser console shows a CORS error

- Send only `Content-Type` and `Accept`. Remove `Authorization`, `X-` and other custom headers.
- Don't use `credentials: 'include'`.
- A typo in the URL answers `404`, which browsers can report as CORS.
- `mode: 'no-cors'` hides the response. Remove it.

## The page reloads or shows a Nisuform page instead of the success message

The submit handler didn't call `event.preventDefault()`, threw before it, or the script was stripped (WordPress does this for some users). The submission still worked.

## No email notification

- Check the spam folder of the receiving address.
- On Free, notifications go to the team owner's account email. On Pro, check **Send to** in **Settings**, **Email**.
- **Email me new submissions** may be off (`notifyEnabled`).
- Spam folder submissions never send email.

## The redirect doesn't happen

Redirects are Pro only, never apply to `fetch` requests that ask for JSON, and `_redirect` must be on the sending site or an allowed website.

## The site shows an old version of the form

The styled embed is a copy. After changing fields or design in the builder, or turning on Turnstile, get the embed code again and replace it.
