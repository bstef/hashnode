---
title: "From AWS to Cloudflare: Why I Migrated My Web Infrastructure"
datePublished: 2026-06-22T23:25:25.051Z
cuid: cmqpufo9f00000bkkb68xezn7
slug: from-aws-to-cloudflare-why-i-migrated-my-web-infrastructure
cover: https://cdn.hashnode.com/uploads/covers/6a0519d6a0c15402778c59a1/4df9223d-5370-41fc-a845-c9298edb5b99.png
tags: website, cloudflare, cloudformation, aws, cloud-computing, migration, infrastructure, awsamplify, cloudflare-worker, cloudflare-pages

---

![](https://cdn.hashnode.com/uploads/covers/6a0519d6a0c15402778c59a1/e0eaa133-b18e-4076-b795-f289771e0cc8.png align="center")

## The Question That Started It All

Like many engineers, I built my early web projects on AWS. It worked — but over time I started asking: *am I paying for complexity I don't actually need?*

After migrating multiple projects off AWS and onto Cloudflare, the answer was yes.

* * *

## What I Was Running on AWS

**AWS Amplify** is Amazon's managed web hosting platform. It connects to your GitHub repo, builds your app automatically on every code push, and serves it globally. It's genuinely capable — but charges per build minute, per GB stored, and per GB served. Costs that start small and grow quietly.

**AWS Route 53** is Amazon's DNS service — the technology that translates `yourcompany.com` into an actual server address. Reliable and deeply integrated with AWS, but it costs $0.50 per domain per month plus per-query fees, and lives inside one of the most complex dashboards in enterprise software.

* * *

## What Cloudflare Replaces Them With

**Cloudflare Pages** → replaces Amplify. Same GitHub integration, same auto-deploy on push, same global delivery. Free tier covers unlimited requests and bandwidth for static sites with no build minute charges.

**Cloudflare DNS** → replaces Route 53. Completely free, consistently one of the fastest DNS resolvers in the world, and takes 10 minutes to set up. DDoS protection and SSL included by default.

**Cloudflare Workers** → lightweight serverless compute that runs at the edge, closer to your users. Far simpler to configure than the AWS Lambda equivalent.

* * *

## The Real Comparison

|  | AWS | Cloudflare |
| --- | --- | --- |
| **Cost** | Metered at every layer | Generous free tier |
| **Complexity** | Built for enterprise scale | Built for speed and simplicity |
| **Security** | Separate services, separate cost | Included by default |
| **Best for** | Complex backend systems | Front-end and edge workloads |

**Where AWS still wins:** Large-scale backend architectures with databases, queues, containers, and ML pipelines. If your app has serious infrastructure behind it, AWS is still the more complete ecosystem.

The honest framing: **Cloudflare is winning the front end. AWS still owns the back end.**

* * *

## What Migration Actually Looks Like

It's less dramatic than it sounds. The high-level process:

1.  **Audit your projects** — Static HTML sites migrate in minutes. Framework apps (React, Vue) take slightly more configuration.
    
2.  **Connect to Cloudflare Pages** — Authorize GitHub, set your build command and output directory, deploy.
    
3.  **Add a** `_redirects` **file** — One line, required for single-page apps so routing works correctly. Amplify handled this automatically; Cloudflare needs it explicitly.
    
4.  **Migrate DNS off Route 53** — Update nameservers, configure records in Cloudflare. Usually propagates within minutes.
    
5.  **Decommission AWS resources** — Critical step. AWS charges for idle resources until you explicitly delete them.
    

* * *

## What I'd Tell Anyone Considering This

*   Static sites are the easiest, highest-value migration — pure HTML/CSS/JS deploys with zero configuration
    
*   DNS migration alone eliminates Route 53 fees across every domain you own
    
*   Cloudflare's dashboard is refreshingly simple compared to the AWS console — that simplicity has real operational value
    
*   Know your framework's build output directory before you start (`dist` for Vite, `build` for Create React App) — it's the most common friction point
    
*   Evaluate Cloudflare's paid tiers if you're running high-traffic production apps — the free tier has real limits at scale
    

Cloud infrastructure decisions have real business consequences on cost, developer productivity, and operational complexity. For front-end hosting, the calculus has shifted — and Cloudflare is worth a serious look.

Happy to connect with anyone navigating similar decisions — whether you're a developer evaluating your stack or a business leader trying to right-size cloud spend.

*— Bobby Stef, Senior Infrastructure & Systems Engineer*

* * *

#Cloudflare #AWS #CloudMigration #WebDevelopment #Infrastructure #DevOps #CloudflarePages #TechLeadership #PeacockEngineering