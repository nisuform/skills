# File uploads

File uploads need the Pro or Business plan, and **Accept file uploads** turned on in the form's **Settings**, **Submissions** (with MCP: `update_form` with `allowUploads: true`, and optionally `uploadLimitMb`). Without both, files are skipped, the rest of the submission is stored, and any request over 200 KB fails with `413` and `payload_too_large`.

## Markup

The form must send `multipart/form-data`:

```html
<form action="https://api.nisuform.com/s/YOUR_FORM_KEY" method="POST" enctype="multipart/form-data">
  <input type="email" name="email" required>
  <input type="file" name="attachment" accept="image/jpeg,image/png,image/webp,image/gif,application/pdf">
  <input type="text" name="_gotcha" tabindex="-1" autocomplete="off" style="display:none">
  <button type="submit">Send</button>
</form>
```

Use `multiple`, or several file inputs, for more than one file. With `fetch`, post `new FormData(form)` and set no `Content-Type` header, so the browser adds the multipart boundary. Never send files as JSON or base64.

## Limits

| | Limit |
| --- | --- |
| Files per submission | 5 |
| Types | JPEG, PNG, WebP, GIF and PDF, detected from the file contents |
| Size per file | The form's limit, 10 MB by default, up to 100 |
| Whole request | 25 MB, all files and fields together |

Check size and type in the browser too, so visitors get feedback before uploading.

## Errors

| Status | `error` | Cause |
| --- | --- | --- |
| `400` | `too_many_files` | More than 5 files |
| `413` | `attachment_too_large` | A file is over the per-file limit |
| `413` | `payload_too_large` | The request is over 25 MB, or over 200 KB when uploads are off |
| `415` | `attachment_type_rejected` | Not a JPEG, PNG, WebP, GIF or PDF |
| `500` | `upload_failed` | Storage failed. Nothing was saved, so the visitor can retry. |

## Where files go

Files are stored privately and downloaded from the inbox through links that expire after 15 minutes. They are not attached to notification emails, and not sent to webhooks, integrations or CSV exports. With the REST API, `GET /v1/submissions/{id}/files/{index}` returns a fresh download link.
