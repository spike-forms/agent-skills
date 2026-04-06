---
name: spike-forms
description: Integrate HTML forms with Spike Forms — a form backend that handles submissions, email notifications, webhooks, and file uploads. No backend code needed.
globs:
  - "**/*.html"
  - "**/*.tsx"
  - "**/*.jsx"
  - "**/*.vue"
  - "**/*.svelte"
  - "**/*.astro"
---

# Spike Forms Integration

Spike Forms is a form backend service. Point your HTML form's `action` to a Spike endpoint and it handles everything — submissions, email notifications, auto-responders, webhooks, and file uploads.

## Quick Start

Every Spike form has an endpoint: `https://api.spike.ac/f/{org_code}/{form_slug}`

### HTML Form

```html
<form action="https://api.spike.ac/f/YOUR_ORG/YOUR_FORM" method="POST">
  <input type="text" name="name" placeholder="Name" required />
  <input type="email" name="email" placeholder="Email" required />
  <textarea name="message" placeholder="Message"></textarea>
  <button type="submit">Send</button>
</form>
```

### React / Next.js

```tsx
export function ContactForm() {
  const [status, setStatus] = useState("");

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    setStatus("sending");
    const formData = new FormData(e.currentTarget);

    try {
      const res = await fetch("https://api.spike.ac/f/YOUR_ORG/YOUR_FORM", {
        method: "POST",
        body: formData,
      });
      setStatus(res.ok ? "success" : "error");
      if (res.ok) e.currentTarget.reset();
    } catch {
      setStatus("error");
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" name="name" placeholder="Name" required />
      <input type="email" name="email" placeholder="Email" required />
      <textarea name="message" placeholder="Message" />
      <button type="submit" disabled={status === "sending"}>
        {status === "sending" ? "Sending..." : "Send"}
      </button>
      {status === "success" && <p>Message sent!</p>}
      {status === "error" && <p>Something went wrong.</p>}
    </form>
  );
}
```

### JSON API

```typescript
const response = await fetch("https://api.spike.ac/f/YOUR_ORG/YOUR_FORM", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    name: "John Doe",
    email: "john@example.com",
    message: "Hello!",
  }),
});
```

### With File Uploads

```html
<form action="https://api.spike.ac/f/YOUR_ORG/YOUR_FORM" method="POST" enctype="multipart/form-data">
  <input type="text" name="name" required />
  <input type="email" name="email" required />
  <input type="file" name="resume" />
  <button type="submit">Submit</button>
</form>
```

Files are stored in Cloudflare R2. The submission data will contain `file:filename.ext` for each uploaded file.

## Special Fields

These hidden fields control Spike's behavior:

| Field | Description |
|-------|-------------|
| `_next` | URL to redirect to after submission |
| `_subject` | Custom email notification subject |
| `_replyto` | Set the reply-to email address |
| `_gotcha` | Honeypot field — if filled, submission is silently rejected |

Example:
```html
<input type="hidden" name="_next" value="https://yoursite.com/thank-you" />
<input type="hidden" name="_subject" value="New contact from {{name}}" />
```

## Auto-Create Forms

If you submit to an endpoint where the form doesn't exist yet, Spike auto-creates it. The org owner gets an email notification. This means you can start collecting submissions immediately without creating the form in the dashboard first.

## UTM Tracking

Spike automatically captures UTM parameters from the referrer URL or from submission fields:
- `utm_source` / `_utm_source`
- `utm_campaign` / `_utm_campaign`

## Webhooks

Configure webhooks in the form settings to receive a POST request for every submission:

```json
{
  "event": "submission.created",
  "timestamp": "2026-01-01T00:00:00.000Z",
  "form": { "id": "...", "name": "Contact", "slug": "contact" },
  "submission": {
    "id": "...",
    "data": { "name": "John", "email": "john@example.com", "message": "Hello" },
    "createdAt": "2026-01-01T00:00:00.000Z"
  }
}
```

Webhooks include HMAC-SHA256 signatures when a secret is configured:
- `X-Spike-Signature`: HMAC signature of `{timestamp}.{body}`
- `X-Spike-Timestamp`: Unix timestamp

## API Authentication

For programmatic access, use an API key (prefix `sk_`):

```typescript
const response = await fetch("https://api.spike.ac/forms", {
  headers: { Authorization: "Bearer sk_your_api_key" },
});
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/forms` | List all forms |
| `POST` | `/forms` | Create a form |
| `GET` | `/forms/:id` | Get a form |
| `PATCH` | `/forms/:id` | Update a form |
| `DELETE` | `/forms/:id` | Delete a form and its submissions |
| `GET` | `/forms/:id/submissions` | List submissions (paginated) |
| `DELETE` | `/forms/:id/submissions` | Bulk delete submissions |
| `GET` | `/forms/:id/export` | Export submissions as JSON |
| `POST` | `/f/:org_code/:slug` | Submit a form (public) |
| `GET` | `/health` | Health check |

## Dashboard

- Dashboard: `https://app.spike.ac`
- API: `https://api.spike.ac`
- Docs: `https://docs.spike.ac`

## Key Concepts

- **Organization**: A workspace that contains forms. Has a `short_code` used in endpoints.
- **Form**: An endpoint that receives submissions. Has a `slug` used in the URL.
- **Submission**: A form response stored in Cloudflare D1. Contains the form data as JSON.
- **Auto-create**: Forms are automatically created when a submission hits a new slug.
- **File uploads**: Files go to Cloudflare R2, referenced as `file:filename` in submission data.
