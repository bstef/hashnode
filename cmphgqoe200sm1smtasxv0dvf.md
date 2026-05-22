---
title: "I Built a Self-Updating Electricity Dashboard from Gmail Notifications — Here’s How It Got Messy"
datePublished: 2026-05-22T22:00:12.029Z
cuid: cmphgqoe200sm1smtasxv0dvf
slug: i-built-a-self-updating-electricity-dashboard-from-gmail-notifications-here-s-how-it-got-messy

---

I have a habit of over-engineering solutions to small problems. My electric utility, PSE&G, sends me threshold notification emails every time my daily usage exceeds 15 kWh. I had a folder full of them and no easy way to see trends. Was summer 2025 really that bad? (Spoiler: yes, 59 kWh in a single day.) Was my usage creeping up in winter?

Instead of exporting a spreadsheet or logging into the PSE&G portal, I did what any reasonable infrastructure engineer would do: I built a live dashboard hosted on Cloudflare Pages that automatically pulls data from my Gmail, parses it, and renders a full chart and table. Zero servers. Zero cron jobs. Just a Cloudflare Pages Function, a KV cache, and some OAuth plumbing.

It’s called **Inwood Electricity** — named after my Proxmox homelab node — and it lives at [inwood-electricity.pages.dev](https://inwood-electricity.pages.dev).

Here’s the full story of how it came together, the deployment rabbit holes I fell into, and the subtle Gmail API bug that nearly broke everything.

* * *

## The Idea

The PSE&G notification emails all follow the same format:

> A MyMeter threshold setting for your account, \*\*\*\*1908, has been met. Meter #000303411361 has daily total consumption above 15 kWh. Actual use for this period was **21.81 kWh** at **05-20-26** 12:00 A.

Every single email has a date and a kWh value sitting right there in the snippet. No need to open the email body — the snippet alone has everything.

So the plan was straightforward:

1.  Use the Gmail API to fetch all emails from `MyMeter@email.pseg.com`
    
2.  Parse the kWh value and date from each snippet with a regex
    
3.  Cache the result in Cloudflare KV so we’re not hammering Gmail on every page load
    
4.  Serve a static dashboard that fetches the JSON and renders it
    

Simple enough. Except nothing is ever simple.

* * *

## The Architecture

Before writing a line of code, I mapped out what I wanted:

```plaintext
PSE&G Email → Gmail
       ↓
Cloudflare Pages Function (/api/meter-data)
  • OAuth2 refresh token → Gmail API
  • Fetch all MyMeter emails
  • Parse date + kWh + type from snippets
  • Cache in Cloudflare KV (6 hours)
       ↓
public/index.html (static)
  • Fetches /api/meter-data on load
  • Bar chart, monthly summary, sortable table
  • Live/Cached status indicator
  • ?refresh=1 to bust cache
```

No backend server. No database. No scheduled jobs. The Cloudflare Pages Function runs on-demand when someone loads the page, checks the KV cache, and only calls Gmail if the cache is stale. The whole thing costs exactly $0 to run.

* * *

## Building the Dashboard

The frontend is a single `public/index.html` — pure HTML, CSS, and vanilla JS. No framework, no build step. It fetches `/api/meter-data` on load and renders three sections:

*   **Stat cards** — total alerts, peak day, average, lowest, monthly record, busiest month
    
*   **Bar chart** — every daily reading plotted chronologically, color-coded by intensity
    
*   **Sortable table** — all records with type badges, mini bars, and a “+/− threshold” column
    

The color tiers made the trends immediately obvious:

| Range | Color | Meaning |
| --- | --- | --- |
| 15–20 kWh | 🟢 Teal | Barely over threshold |
| 20–35 kWh | 🟡 Yellow | Moderate |
| 35–50 kWh | 🟠 Orange | High |
| 50+ kWh | 🔴 Red | Peak summer heat |

One glance at the bar chart and you can see the summer 2025 spike — late June through July was brutal, with multiple days above 50 kWh and a peak of **59.11 kWh on July 6**.

* * *

## The Gmail API Worker

The Pages Function lives at `functions/api/meter-data.js`. Cloudflare automatically routes requests to `/api/meter-data` through this file.

### OAuth2 — Getting an Access Token

Gmail requires an OAuth2 access token. Since this is a personal app accessing my own inbox, I used the OAuth Playground to get a long-lived refresh token, then stored it as an encrypted environment variable in Cloudflare. On every request, the Worker exchanges the refresh token for a short-lived access token:

```javascript
async function getAccessToken(env) {
  const res = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      client_id: env.GMAIL_CLIENT_ID,
      client_secret: env.GMAIL_CLIENT_SECRET,
      refresh_token: env.GMAIL_REFRESH_TOKEN,
      grant_type: 'refresh_token',
    }),
  });
  const data = await res.json();
  if (!data.access_token) throw new Error(`OAuth error: ${JSON.stringify(data)}`);
  return data.access_token;
}
```

Three environment variables, set once in the Cloudflare dashboard as encrypted secrets. Done.

### Fetching the Emails

```javascript
const GMAIL_SEARCH_QUERY = 'from:MyMeter@email.pseg.com subject:"MyMeter Threshold Notification"';

async function fetchMessageList(accessToken) {
  const allMessages = [];
  let pageToken = null;
  let pages = 0;

  do {
    const params = new URLSearchParams({
      q: GMAIL_SEARCH_QUERY,
      maxResults: '500',
    });
    if (pageToken) params.set('pageToken', pageToken);

    const res = await fetch(
      `https://gmail.googleapis.com/gmail/v1/users/me/messages?${params}`,
      { headers: { Authorization: `Bearer ${accessToken}` } }
    );
    const data = await res.json();
    if (!data.messages) break;

    allMessages.push(...data.messages);
    pageToken = data.nextPageToken || null;
    pages++;
  } while (pageToken && pages < 4);

  return allMessages;
}
```

Each page of results costs exactly one subrequest. With ~200 emails total that’s a single page — one subrequest to get all the data.

### Parsing the Snippets

This is the heart of it. Two regexes do all the work:

```javascript
function parseReadings(messages) {
  const readings = [];
  const seen = new Set();

  for (const msg of messages) {
    const snippet = msg.snippet || '';

    // "Actual use for this period was 21.81 kWh"
    const kwhMatch = snippet.match(/was\s+([\d.]+)\s+kWh/i);
    if (!kwhMatch) continue;
    const kwh = parseFloat(kwhMatch[1]);

    // "at 05-20-26 12:00"
    const dateMatch = snippet.match(/at\s+(\d{2}-\d{2}-\d{2})\s+\d{2}:\d{2}/i);
    if (!dateMatch) continue;
    const [mm, dd, yy] = dateMatch[1].split('-');
    const date = `20${yy}-${mm}-${dd}`;

    // "daily total", "hourly total", "monthly total"
    let type = 'daily';
    if (/hourly\s+total/i.test(snippet))   type = 'hourly';
    else if (/monthly\s+total/i.test(snippet)) type = 'monthly';

    // Deduplicate
    const key = `${date}|${kwh}|${type}`;
    if (seen.has(key)) continue;
    seen.add(key);

    readings.push({ date, kwh, type });
  }

  readings.sort((a, b) => b.date.localeCompare(a.date));
  return readings;
}
```

Clean, simple, and it handles all three alert types PSE&G sends — daily, hourly, and monthly.

### KV Caching

```javascript
// Check cache first
const cached = await env.KV_METER_DATA.get(CACHE_KEY, { type: 'json' });
if (cached && !isStale(cached.fetchedAt, CACHE_TTL_SECONDS)) {
  return new Response(JSON.stringify({ ...cached, source: 'cache' }), { headers: corsHeaders });
}

// ... fetch from Gmail ...

// Write to cache
await env.KV_METER_DATA.put(CACHE_KEY, JSON.stringify(payload), {
  expirationTtl: CACHE_TTL_SECONDS, // 6 hours
});
```

The dashboard shows whether it’s serving live or cached data in the status pill at the top right. Hit `?refresh=1` to force a fresh fetch anytime.

* * *

## The Deployment Rabbit Hole

Here’s where things got interesting. I figured deploying to Cloudflare Pages would take 10 minutes. It took considerably longer.

**Problem 1: Workers vs Pages**

Cloudflare has merged Workers and Pages into one UI but they’re still fundamentally different products. I kept accidentally creating Worker projects instead of Pages projects. Workers don’t serve static files from a `public/` folder. Pages do. The “Looking to deploy Pages? **Get started**” link buried at the bottom of the Create screen was the answer.

**Problem 2: The deploy command**

The new Cloudflare Pages UI requires a deploy command. But for a static site with no build step, there is no deploy command — Cloudflare handles it automatically. Except the UI won’t let you save without one. The fix: `echo "deploy"`. A no-op shell command that always succeeds and does nothing.

**Problem 3:** `wrangler.toml` **placeholder IDs**

I had placeholder text in `wrangler.toml` for the KV namespace IDs. Cloudflare saw this and blocked the KV binding from being set in the dashboard (“managed from toml”). The fix was creating the real KV namespaces first with `npx wrangler kv namespace create KV_METER_DATA`, then putting the real IDs in the file.

None of these were hard problems — they were just papercuts that added up.

* * *

## The Bug That Broke Everything

Once deployed, the dashboard loaded but showed zero data. The API was returning:

```json
{"readings": [], "fetchedAt": "...", "totalEmails": 0, "source": "cache"}
```

The OAuth was working fine. The KV cache was writing. But `totalEmails` was 0 every time.

My first approach had used Gmail’s batch HTTP API to fetch all message snippets in a single request. The idea: get a list of message IDs, then batch-fetch the snippet for each one in one multipart HTTP call.

The bug was subtle. Gmail’s batch API has a **hard limit of 100 requests per batch**. I had ~200 emails. When you exceed the limit, the API doesn’t throw an error — it just silently returns nothing. Empty response. Zero snippets parsed. Zero readings.

The fix turned out to be much simpler than the original approach. The `messages.list` API already returns the `snippet` field for each message in the list response. There was never a need for a second API call at all.

One `messages.list` request → 500 results with snippets included → parse inline → done. Two subrequests total (one for the OAuth token exchange, one for the message list). No batch API needed.

It’s a good reminder that sometimes the right fix isn’t a better implementation of the complex solution — it’s realizing the complex solution wasn’t necessary in the first place.

* * *

## What I Learned

**Cloudflare Pages Functions are genuinely great** for this kind of lightweight API work. No cold starts worth worrying about, KV is fast, and the free tier is more than enough for a personal dashboard.

**The Gmail API’s** `messages.list` **snippet is underrated.** Most tutorials tell you to fetch individual messages for content. For simple parsing tasks like this, the snippet in the list response is often all you need.

**Subrequest limits are sneaky.** Cloudflare Workers allow 50 subrequests per invocation. The batch API limit issue wasn’t a Cloudflare problem at all — it was a Gmail API constraint I hadn’t read carefully enough. Always check the limits on third-party APIs, not just your own runtime.

**OAuth setup for personal apps is tedious but one-time.** Adding yourself as a test user in Google Cloud’s Audience screen, going through the OAuth Playground, storing the refresh token as an encrypted secret — it’s annoying the first time. But once it’s done it just works indefinitely.

* * *

## What’s Next

A few things I want to add:

*   **Custom domain** — point `electricity.peacockengr.com` at it
    
*   **Home Assistant sensor** — have HA call `/api/meter-data` and pull the latest reading into my homelab dashboard
    
*   **Cloudflare Cron Trigger** — proactively check for high usage and push a notification instead of waiting for me to open the dashboard
    
*   **Cost estimation** — map kWh readings to estimated dollar amounts based on PSE&G’s rate schedule
    

The repo is at [github.com/bstef/inwood-electricity](https://github.com/bstef/inwood-electricity) if you want to fork it for your own utility provider. The snippet parsing regex is the only thing you’d need to change.

* * *

*Bobby Stef is a Senior Infrastructure & Systems Engineer based in New Jersey, running* [*Peacock Engineering*](https://peacockengr.com) *on the side. He writes about homelab projects, infrastructure tooling, and the occasional over-engineered solution to a small problem.*