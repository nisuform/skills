<p align="center">
  <img src="assets/logo.png" alt="Nisuform" width="96" height="96">
</p>

<h1 align="center">Nisuform skills</h1>

<p align="center">
  Agent skills and a Claude Code plugin for <a href="https://nisuform.com">Nisuform</a>, the form backend for any website.
</p>

Nisuform gives every form a public endpoint, `https://api.nisuform.com/s/{key}`. Your site's form posts straight to it, and Nisuform filters spam, stores submissions, emails you and forwards them to Slack, Sheets or a webhook. No server code, API routes or environment variables.

These skills teach your coding agent to wire up Nisuform correctly the first time: the honeypot, the fill-time check, the JSON response, captchas, webhooks and the REST API.

## Skills

| Skill | Use it to |
| --- | --- |
| [`nisuform-forms`](skills/nisuform-forms/SKILL.md) | Add a contact, waitlist, feedback, RSVP or custom form to HTML, React, Next.js, Vue, Nuxt, Svelte, SvelteKit, Astro, Webflow, Framer or WordPress. Also covers Turnstile, reCAPTCHA, file uploads, redirects and troubleshooting. |
| [`nisuform-webhooks`](skills/nisuform-webhooks/SKILL.md) | Receive and verify signed `submission.created` webhooks in Next.js, Express, Hono, Cloudflare Workers, FastAPI or Flask. |
| [`nisuform-api`](skills/nisuform-api/SKILL.md) | Manage forms and submissions from scripts and backends with the REST API and an `nfa_` key. |
| [`nisuform-migrate`](skills/nisuform-migrate/SKILL.md) | Move forms from Formspree, Web3Forms, FormSubmit, Netlify Forms, Getform, Formcarry, Basin, EmailJS or a custom email route. |

The Claude Code plugin also connects the [Nisuform MCP server](https://nisuform.com/docs/mcp), so Claude can create forms, get their embed code and read submissions in your account. You sign in with your Nisuform account the first time a tool runs.

## Install

### Claude Code

```sh
claude plugin marketplace add nisuform/skills
claude plugin install nisuform@nisuform
```

Or from inside a session:

```text
/plugin marketplace add nisuform/skills
/plugin install nisuform@nisuform
```

Then run `/mcp`, pick **nisuform** and sign in to let Claude work with your account.

### Other agents

Install the skills into Cursor, Codex, GitHub Copilot, Gemini CLI, OpenCode and other agents that read `SKILL.md` files:

```sh
npx skills add nisuform/skills
```

Or copy the folders in [`skills/`](skills) into your agent's skills directory, for example `.claude/skills/` or `.agents/skills/` in your project.

To connect the MCP server to other tools, see [MCP server setup](https://nisuform.com/docs/mcp).

## Try it

```text
Add a contact form to the contact page that sends me an email for every message.
```

```text
Move our Formspree forms to Nisuform.
```

```text
Add an endpoint that receives Nisuform webhooks and saves each submission to the database.
```

```text
Write a script that exports this month's unread submissions to CSV.
```

## Get a form endpoint

Create a free account at [app.nisuform.com](https://app.nisuform.com). The Free plan includes 100 submissions a month and 3 forms, with no credit card. Each form's endpoint is on its **Setup** tab, or Claude can create one for you through the MCP server.

## Links

- [Documentation](https://nisuform.com/docs)
- [Docs for agents](https://nisuform.com/llms-full.txt)
- [Pricing](https://nisuform.com/pricing)
- [Support](https://nisuform.com/support)

## License

[MIT](LICENSE)
