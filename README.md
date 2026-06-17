# Job Search Agent

An AI-powered job search tool built for Tech & Data roles. Search for jobs, track applications, draft outreach emails, and filter by visa sponsorship — all from a single webpage.

**Live site:** https://katiyaranamika-rgb.github.io/job-search-agent

---

## What it does

### Find Jobs
- Search by job title, experience level, locations, and skills
- Add multiple locations at once (type and press Enter to add each one)
- Filter by company stage: Public, Private, or Pre-IPO
- Set a minimum salary threshold
- Toggle H1B / visa sponsorship filter to only see companies known to sponsor (see Visa Sponsorship section below)
- Each result shows a match score, salary range, tech stack tags, and a 2-sentence role summary
- Every job card includes a colour-coded visa sponsorship banner (green = likely, red = unlikely, amber = unknown) with a one-line reason
- Click "Company insights" on any card to see a company snapshot (founded, team size, funding, mission) and 3 recent news/highlights bullets

### Saved Searches
- Fill in any combination of filters and click **"Save current"** to save them by name
- Saved searches appear as chips above the search form — click any chip to reload all filters instantly
- Delete saved searches you no longer need with the × button
- All saved searches persist in your browser's local storage across visits

### Application Tracker
- Track every job you apply to with company, role, status, date, and notes
- One-click "Track" button on every job card adds it directly to the tracker
- Update status: Applied → Interviewing → Offer received → Rejected
- Summary stats show total applications, interviews, and offers at a glance
- Data saved in browser local storage — persists between visits

### Outreach Emails
- Fill in your name, background, target company, and role
- The AI drafts a ready-to-send cold outreach email under 180 words
- Mentions visa sponsorship naturally and confidently when relevant
- Jump directly to email drafting from any job card or tracker entry

---

## Tech stack

| Layer | Technology |
|-------|-----------|
| Frontend | Plain HTML, CSS, JavaScript (no framework) |
| AI | Anthropic Claude API (claude-sonnet-4-6) |
| Proxy | Cloudflare Workers |
| Hosting | GitHub Pages |
| Storage | Browser localStorage |

---

## Architecture

```
User browser (GitHub Pages)
        │
        │  POST /v1/messages
        ▼
Cloudflare Worker (job-agent-proxy.katiyaranamika.workers.dev)
        │
        │  POST /v1/messages  +  x-api-key header
        ▼
Anthropic Claude API
```

### Why a Cloudflare Worker is needed (CORS issue)

Browsers enforce a security rule called **CORS (Cross-Origin Resource Sharing)**. It blocks websites from directly calling third-party APIs — including the Anthropic API — unless that API explicitly allows it. The Anthropic API does not allow direct browser calls for security reasons.

**The problem:** Calling `https://api.anthropic.com` directly from the GitHub Pages site caused a `NetworkError: Failed to fetch` error in the browser. The API call never reached Anthropic.

**The solution:** A Cloudflare Worker acts as a proxy (middleman) between the site and the Anthropic API:
1. The site sends the request to the Cloudflare Worker instead of directly to Anthropic
2. The Worker runs server-side (not in a browser), so CORS rules don't apply to it
3. The Worker forwards the request to Anthropic, adding the secret API key from an environment variable
4. Anthropic responds to the Worker, which returns the result to the site with permissive CORS headers

This keeps the API key secret (never exposed in the frontend code) and solves the browser restriction.

---

## File structure

```
job-search-agent/
├── index.html      ← the entire app (HTML + CSS + JS in one file)
└── README.md       ← this documentation
```

---

## Cloudflare Worker setup

The Worker code lives separately in Cloudflare (not in this repo). Here is the full Worker script for reference:

```javascript
export default {
  async fetch(request, env) {

    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        }
      });
    }

    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    try {
      const body = await request.json();

      const response = await fetch('https://api.anthropic.com/v1/messages', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': env.ANTHROPIC_API_KEY,
          'anthropic-version': '2023-06-01',
        },
        body: JSON.stringify(body),
      });

      const data = await response.json();

      return new Response(JSON.stringify(data), {
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*',
        }
      });

    } catch (err) {
      return new Response(JSON.stringify({ error: { message: err.message } }), {
        status: 500,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*',
        }
      });
    }
  }
};
```

**Environment variable required:**
- Name: `ANTHROPIC_API_KEY`
- Value: your Anthropic API key from console.anthropic.com
- Set under: Cloudflare Dashboard → Workers & Pages → job-agent-proxy → Settings → Variables → Add variable (mark as Encrypted)

---

## How to update the site

1. Go to `github.com/katiyaranamika-rgb/job-search-agent`
2. Click `index.html`
3. Click the pencil (edit) icon
4. Select all (Ctrl+A) and replace with the new file content
5. Click **Commit changes**

GitHub Pages will redeploy automatically within 1–2 minutes.

---

## Limitations

- **Job listings are AI-generated**, not pulled from live job boards. They are realistic and useful for research but should be verified on actual company career pages.
- **Company news bullets** are based on the AI's training data (knowledge cutoff early 2025), not live news feeds. For live news, a backend server with a news API would be required.
- **Visa sponsorship assessments** are based on the AI's knowledge of each company's historical H1B patterns. Always confirm sponsorship availability directly with the employer.
- **Local storage** means tracked applications and saved searches are tied to your browser. Clearing browser data will erase them.

---

## Built with

- [Anthropic Claude](https://anthropic.com) — AI backbone
- [Cloudflare Workers](https://workers.cloudflare.com) — serverless proxy
- [GitHub Pages](https://pages.github.com) — free static hosting
- [Tabler Icons](https://tabler.io/icons) — icon set
