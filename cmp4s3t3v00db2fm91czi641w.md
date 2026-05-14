---
title: "🚀 Claude AI + Zoom Automation: Building an AI Meeting Agent"
seoTitle: "Using Zoom MCP with Claude AI"
seoDescription: "Automating meetings with AI agents using Claude + Zoom APIs"
datePublished: 2026-05-14T00:57:20.156Z
cuid: cmp4s3t3v00db2fm91czi641w
slug: claude-ai-zoom-automation-building-an-ai-meeting-agent
cover: https://cdn.hashnode.com/uploads/covers/6a0519d6a0c15402778c59a1/528b86dc-5e09-4f79-b183-353b3774884b.jpg
tags: ai, zoom, ai-tools, mcp, mcp-server, mcp-host, mcp-client, zoommcp

---

## TL;DR

I built a system that connects Claude AI with Zoom to automatically:

*   Summarize meetings
    
*   Extract action items
    
*   Trigger follow-ups
    

* * *

## 🧠 The Idea

Meetings create a lot of friction:

*   Notes get lost
    
*   Action items are forgotten
    
*   Follow-ups take time
    

So I built a system where AI handles the entire workflow.

* * *

## ⚙️ Architecture

```plaintext
Zoom → Webhook → Backend → Claude → Action Engine → Tools
```

* * *

## 🔌 Zoom Webhook Example

```javascript
import express from "express";

const app = express();
app.use(express.json());

app.post("/zoom/webhook", async (req, res) => {
  if (req.body.event === "recording.completed") {
    const url = req.body.payload.object.recording_files[0].download_url;
    await processMeeting(url);
  }
  res.sendStatus(200);
});

app.listen(3000);
```

* * *

## 🧠 Claude Processing

```javascript
async function analyzeMeeting(transcript) {
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "x-api-key": process.env.CLAUDE_API_KEY,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      model: "claude-3-opus",
      messages: [{
        role: "user",
        content: `Summarize and extract action items:\n${transcript}`
      }]
    })
  });

  return res.json();
}
```

* * *

## 📌 Structured Output

```json
{
  "summary": "...",
  "decisions": ["..."],
  "actions": [
    { "task": "...", "owner": "..." }
  ]
}
```

* * *

## 📬 Follow-Up Automation

```javascript
async function sendFollowUp(summary, actions) {
  console.log(summary, actions);
}
```

* * *

## 🤖 Why This Matters

This is more than automation.

It’s the shift from: AI as a tool → AI as an operator

* * *

## 🔥 Use Cases

*   Devs → auto-create tickets
    
*   Teams → auto-sync decisions
    
*   Founders → scale across meetings
    

* * *

## 🧭 Future

AI agents that:

*   Join meetings
    
*   Make decisions
    
*   Execute tasks
    

* * *

## 💬 Final Thought

The goal isn’t better meetings.

It’s meetings that do the work for you.