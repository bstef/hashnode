---
title: "From a Spreadsheet Idea to a Live Church App: Building Skyline Pulse"
seoTitle: "Feel the pulse of your church worship"
datePublished: 2026-07-09T20:52:07.991Z
cuid: cmrdzg13r00000aht8jja714i
slug: from-a-spreadsheet-idea-to-a-live-church-app-building-skyline-pulse
cover: https://cdn.hashnode.com/uploads/covers/6a0519d6a0c15402778c59a1/38251b25-6907-4d97-aecf-9f811e435304.png
tags: cloudflare, supabase, claude, cloudflare-pages, claude-code, worshiptech

---

A few weeks ago I set out to solve a small, specific problem at Skyline Church NJ: after every service, a handful of numbers that mattered — attendance, how unified worship felt, how engaged the room was, how the sound actually landed in the back — lived nowhere but in someone's head or a text thread. By the next Sunday, nobody could tell you how the 9:30 service compared to the 11:00, or whether "loud" this week was actually louder than "loud" three weeks ago.

So I built **Skyline Pulse** — a lightweight service ledger for our worship team, now live at [skylinechurchpulse.pages.dev](https://skylinechurchpulse.pages.dev).

## What it does

Pulse turns a post-service debrief into a 60-second habit instead of a memory exercise:

*   **Weekly Ledger** — attendance, worship unity, audience engagement, and loudness (in dB) for every service
    
*   **Calendar & Day View** — a month grid for planning ahead, or a focused day view for Sunday morning itself
    
*   **Room Experience** — sound clarity in the back of the room, spiritual response, how cleanly transitions landed
    
*   **Service Flow** — did the lighting fit the song, did the talking segments and sermon land as planned
    
*   **Planning Center Sync** — Plans pull automatically from our Planning Center Services Service Types, split correctly across multi-service Sundays (like 9:30/11:00), with sermon artwork attached
    
*   **Analytics** — rolling trend charts across every measurement, plus a breakdown of who's logging, so the data stays honest
    
*   **CSV export**, three themes (Light, Dark, Ocean), and a "Logged by" field that remembers who's contributing without requiring any login
    

No accounts. No app to install. One shared link, and the team is logging real data within a minute of walking off stage.

## Why I built it this way

The stack is intentionally small and boring, because "boring" is what keeps running without me babysitting it:

*   **One HTML file** — no framework, no build step
    
*   **Supabase** (Postgres + Row Level Security) as the backend, called straight from the browser
    
*   **Cloudflare Pages** for static hosting, auto-deploying on every push to `main`
    
*   **Pages Functions** as a small serverless proxy, so the Planning Center API token never touches the client
    
*   **Chart.js** for analytics, themed live off the same CSS custom properties as the rest of the app
    

I built the first version by hand, then used **Claude Code** to keep iterating — adding the calendar views, the Planning Center integration, the theming system, and the analytics layer. It's a good example of what's possible right now for a solo builder: a real, integrated, production tool for a real organization, built and shipped in the margins of a week, on entirely free-tier infrastructure.

## What's next

The repo's public if you want to see how it's put together: [github.com/bstef/skylinechurchpulse](https://github.com/bstef/skylinechurchpulse). There's also a full features walkthrough at [skylinechurchpulse.pages.dev/docs](https://skylinechurchpulse.pages.dev/docs).

If you lead worship, run production, or just care about turning "I think it went well" into something you can actually see trending over time — I'd love to hear how your team tracks this stuff, or what you'd want a tool like this to do next.

#WorshipTech #ChurchTech #CloudflarePages #Supabase #ClaudeCode #BuildInPublic