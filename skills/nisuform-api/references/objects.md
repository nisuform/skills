# Objects

## Contents

- [Form](#form)
- [Updating a form](#updating-a-form)
- [Fields](#fields)
- [Submission](#submission)
- [Submission list](#submission-list)
- [Account](#account)

## Form

```json
{
  "id": "2f6b7c1e-9a4d-4c3e-8b21-7d5e6f4a3b2c",
  "name": "Website contact",
  "type": "contact",
  "emoji": "✉️",
  "key": "k3yz8q2m4n6p0r5t7v9w",
  "endpoint": "https://api.nisuform.com/s/k3yz8q2m4n6p0r5t7v9w",
  "status": "active",
  "closedReason": null,
  "notifyEnabled": true,
  "notifyEmail": null,
  "fields": [
    { "id": "name", "type": "text", "label": "Name", "required": true },
    { "id": "email", "type": "email", "label": "Email", "required": true },
    { "id": "message", "type": "textarea", "label": "Message", "required": true }
  ],
  "createdAt": "2026-09-29T10:00:00.000Z",
  "updatedAt": "2026-09-29T10:00:00.000Z"
}
```

| Field | Meaning |
| --- | --- |
| `id` | Used in API paths. |
| `name`, `type`, `emoji` | Dashboard name, `contact`, `waitlist`, `feedback`, `rsvp` or `custom`, and icon. |
| `key`, `endpoint` | The public key and the submit URL, `https://api.nisuform.com/s/{key}`. |
| `status` | `active` or `paused`. |
| `closedReason` | `manual`, `max_submissions`, `auto_close_date`, or `null` while open. |
| `maxSubmissions`, `autoCloseAt` | Close after this many submissions, or at this time. |
| `closedMessage` | Pro. Text shown once closed. |
| `notifyEnabled`, `notifyEmail`, `notifySubject`, `notifyFromName` | Email notifications. `notifyEmail` and a custom sender name are Pro. The subject supports `{{form.name}}` and `{{field.<name>}}`. |
| `autoreplyEnabled`, `autoreplySubject`, `autoreplyBody` | Pro. Confirmation email to the submitter. The body supports `{{<name>}}` and `{{<name>.label}}`. |
| `completionMode` | `default`, `message` or `redirect`: what browser posts see after submitting. |
| `completionTitle`, `completionMessage`, `redirectUrl` | Pro. The custom thank-you page or redirect. |
| `fields` | Field schema used to validate and label answers, or `null` to accept anything. |
| `design` | The styled embed: `theme`, `accent`, `title`, `description`, `submitLabel`, `successMessage`. |
| `allowedOrigins` | Websites allowed to submit. Empty allows all. |
| `blockedIps`, `blockedCountries` | Dropped senders. |
| `captchaEnabled`, `captchaProvider`, `captchaSiteKey`, `captchaSecretKeySet` | Turnstile or reCAPTCHA. The secret is never returned. |
| `dontStoreIp` | Leave IP addresses out of submissions. |
| `allowUploads`, `uploadLimitMb` | Pro. File uploads and the per-file size limit. |

`GET /v1/forms` returns a shorter shape per form: `id`, `name`, `type`, `emoji`, `status`, `key`, `endpoint`, `createdAt` and `submissionCount` (spam and tests excluded).

## Updating a form

`PATCH /v1/forms/{id}` changes only the fields sent. `null` clears a field. A Pro setting on a Free team answers `402` and saves nothing.

| Field | Accepts |
| --- | --- |
| `name` | 1 to 200 characters |
| `type` | `contact`, `waitlist`, `feedback`, `rsvp`, `custom`. Keeps the fields. |
| `status` | `active` or `paused`. Resuming clears `closedReason`. |
| `emoji` | Up to 32 characters, or `null` |
| `maxSubmissions` | 1 to 1,000,000, or `null` |
| `autoCloseAt` | ISO 8601 time, or `null` |
| `closedMessage` | Pro. 1 to 500 characters, or `null` |
| `notifyEnabled` | Boolean |
| `notifyEmail` | Pro. Email address, or `null` for the team owner |
| `notifySubject` | 1 to 500 characters |
| `notifyFromName` | 1 to 100 characters. Pro to change it from the default. |
| `autoreplyEnabled` | Boolean. Pro to turn on. |
| `autoreplySubject` | Pro. 1 to 500 characters, or `null` |
| `autoreplyBody` | Pro. 1 to 5,000 characters, or `null` |
| `completionMode` | `default`, `message`, `redirect` |
| `completionTitle` | Pro. 1 to 200 characters, or `null` |
| `completionMessage` | Pro. 1 to 2,000 characters, or `null` |
| `redirectUrl` | Pro. `http` or `https` URL. `null` or `""` removes it. |
| `fields` | Up to 30 fields, or `null`. Replaces the whole list, so read the form first and send every field to keep. |
| `design` | The whole design object. Replaces the current one. |
| `allowedOrigins` | Up to 50 origins. Paths are dropped. |
| `blockedIps` | Up to 1,000 exact IP addresses |
| `blockedCountries` | Up to 300 two-letter country codes |
| `captchaProvider` | `turnstile` or `recaptcha`. Changing it clears the saved keys unless new ones come in the same request. |
| `captchaSiteKey`, `captchaSecretKey` | Strings, or `null`. The secret is write only. |
| `captchaEnabled` | Boolean. Turning it on needs both keys saved before or in the same request. Add the widget to the site first. |
| `dontStoreIp` | Boolean |
| `allowUploads` | Boolean. Pro to turn on. |
| `uploadLimitMb` | 1 to 100. Pro to change it from 10. |

## Fields

Each field has an `id` (the `name` the HTML form sends), a `type`, a `label` and `required`.

```json
{
  "fields": [
    { "id": "email", "type": "email", "label": "Email", "required": true },
    { "id": "plan", "type": "select", "label": "Plan", "required": false, "options": ["Starter", "Team"] },
    { "id": "score", "type": "rating", "label": "How likely are you to recommend us?", "required": true, "max": 10 }
  ]
}
```

| Type | Accepts |
| --- | --- |
| `text`, `textarea` | Any text. Over `maxLength` is cut and flagged. `text` also takes `minLength` and `pattern`. |
| `email` | Must look like `name@example.com`. |
| `number` | A number within `min` and `max`. |
| `date` | A readable date like `2026-09-24`. |
| `select`, `radio` | One of `options` (1 to 100), exact match including case. |
| `checkbox` | A single box. `on`, `true`, `1` are saved as `yes`. |
| `rating` | A whole number from 1 to `max` (3 to 10, default 5). |
| `tel` | Digits, spaces and `+ ( ) . - /`, at least 3 digits. |
| `url` | A full `http://` or `https://` URL. |
| `file` | Only checked for `required`. |

All types also take `placeholder` (up to 200 characters) and `helpText` (up to 500). IDs start with a letter, use letters, digits, `-` and `_`, are up to 64 characters and unique. Failing a check never rejects a submission: it is stored with `data._invalid`.

## Submission

```json
{
  "id": "8d0f5a4e-3b9c-4f7a-9e21-6c4d2b1a0f93",
  "formId": "2f6b7c1e-9a4d-4c3e-8b21-7d5e6f4a3b2c",
  "data": { "name": "Jane Doe", "email": "jane@example.com", "message": "Hello!" },
  "meta": {
    "ip": "203.0.113.7",
    "userAgent": "Mozilla/5.0 ...",
    "referrer": "https://example.com/contact",
    "country": "DE",
    "deviceClass": "desktop"
  },
  "read": false,
  "starred": false,
  "isSpam": false,
  "isTest": false,
  "createdAt": "2026-09-29T09:29:59.412Z"
}
```

- `data` values are all strings. `data._invalid`, when present, is a JSON string listing failed checks, like `[{"fieldId":"email","reason":"invalid_email"}]`.
- `meta.ip` is missing when the form doesn't store IPs. `meta.deviceClass` is `mobile`, `desktop`, `tablet` or `bot`. `meta.files` lists uploads with `name`, `mime` and `size`, and its index is the `{index}` for the download endpoint.

## Submission list

`GET /v1/submissions` returns:

| Field | Meaning |
| --- | --- |
| `items` | Submissions on this page, each with a short `preview`. |
| `page`, `pageSize`, `pageCount` | Paging. A page past the end returns the last page. |
| `total` | Submissions matching the filter. |
| `unreadCount` | Unread in the form, or in all forms without `formId`, whatever the filter. |
| `asOf` | When the list was read. Pass it to filter-based updates and deletes. |

## Account

`GET /v1/me` returns `id`, `email`, `username`, `plan` (`free`, `pro`, `business`), `usage` (`used` and `limit` this month, reset on the 1st in UTC), `formLimit` (`null` for unlimited), `activeTeamId`, and `teams` with only the key's team.
