---
title: "From a Podcast Archive to a Searchable Sermon Library: Building Skyline Echo
"
datePublished: 2026-10-13T01:12:57.774Z
cuid: cmv5zldw1000007l5b5yfcvx1
slug: from-a-podcast-archive-to-a-searchable-sermon-library-building-skyline-echo
cover: https://cdn.hashnode.com/uploads/covers/6a0519d6a0c15402778c59a1/7aa52df5-37f6-440a-92ed-e387b5efd7ad.png
tags: podcast, podcasts, searching-algorithms, podcast-transcription

---

A sermon can stick with you long after Sunday. Finding it again is a different story.

You remember a phrase or an idea, but not the episode title. Maybe it was a few weeks ago. Maybe it was last year. The audio exists, but finding the moment means opening episodes and moving around the progress bar until something sounds familiar.

I wanted a better way to revisit Skyline Church’s messages, so I started building **Skyline Echo**: a sermon library and timestamped search app for our church in Bloomfield, New Jersey.

The first version is live at [skylinechurch.app](https://skylinechurch.app/). Here is what it does today, how the transcription pipeline fits together, and what I still need to test.

## Make the archive useful before adding AI

The initial archive contains 481 podcast episodes. That is already a lot of messages to browse, even without transcripts.

Echo gives those episodes a dedicated home. Each entry has a readable date, the day of the week, its title, and access to the original podcast audio. Speaker credits appear when the podcast explicitly provides them.

There is also a calendar with month, week, and agenda views. You can switch years, jump to a date, or select a day to see its sermons. For someone who remembers “it was around Easter” more clearly than an episode title, browsing by date gives them another route into the archive.

Playback uses one persistent audio player. The playing sermon is highlighted, and a “Show in list” control brings its entry back into view. It is a small feature, but a progress bar at the bottom of the screen should not leave you wondering which episode is playing.

I used Skyline’s color logos and Instrument typefaces, with light and dark themes built around the church’s blue–green palette. The footer connects back to the church, Church Center, social channels, directions, and the podcast on Spotify and Apple Podcasts. A separate strip links to the project documentation and source code.

## The real goal: find a passage, then hear it

Browsing the archive is the foundation. The more interesting part is finding something inside a sermon.

For that, Echo needs actual transcript text and timestamps. Published transcripts are grouped into passages of roughly a minute, with some overlap so a thought is less likely to disappear across a passage boundary.

A search result can then show the words in context and start the original audio near where they were said. The audio remains on the podcast host; the app stores the transcript passages and the references needed to find the moment.

There are two search paths:

*   **Keyword search** uses SQLite FTS5 to find words in published passages.
    
*   **Contextual search** is an optional integration using embeddings to retrieve passages related to an idea, even when the exact wording differs.
    

Groq handles transcription. It does not provide the embeddings for this search integration. The optional meaning-based path uses OpenAI embeddings and Cloudflare Vectorize on the deployed app.

When that configuration is unavailable, the app visibly falls back to keyword search. I have not yet validated contextual relevance against a real transcribed sermon corpus.

## Put the website and audio processing in different places

The public app runs on **Cloudflare Workers with Static Assets and D1**. The frontend uses HTML, CSS, and JavaScript. D1 stores episode metadata and transcript passages.

The audio pipeline runs on my local infrastructure: **n8n installed directly in a Proxmox LXC**, with Python and FFmpeg alongside it.

The process is:

1.  Check the public podcast RSS feed and sync episode metadata.
    
2.  Select a bounded batch of episodes that have not been indexed.
    
3.  Download one episode and convert it into 16 kHz mono FLAC chunks, about ten minutes each.
    
4.  Send the chunks to Groq for timestamped transcription.
    
5.  Save the responses, combined transcript, and local search index.
    
6.  Upload the completed transcript and episode metadata to a protected Cloudflare endpoint.
    

Groq does the speech recognition on its servers. The LXC prepares the audio and coordinates the requests; it does not run a local Whisper model or need a GPU for this workflow.

Downloaded audio and converted chunks are temporary. Transcript JSON and checkpoint responses remain on the LXC. Processing is sequential, so an archive backfill does not need to download every sermon at once.

## Make retries part of the design

I do not want a weekly job that needs manual cleanup whenever one request fails.

The transcription runner saves completed chunk responses. If a later chunk fails, rerunning can reuse what was already completed. Checkpoints are separated by model and processing settings so responses from different configurations do not get mixed together.

Publishing is tracked separately. If Groq finished but the Cloudflare upload failed, the saved transcript can be uploaded again without another transcription call. Successful publication writes a local receipt, and the endpoint recognizes identical content on a repeated request.

Replacing transcript passages happens in a database transaction. If the insertion fails, the previous transcript remains available. The Cloudflare database assigns its own passage IDs rather than relying on IDs from an independent local database.

That separation also means the website can receive a new weekly sermon and its transcript without a frontend rebuild.

## Test the connection before paying for the archive

The first end-to-end connection test used a synthetic sample with timestamps. It went from n8n in the LXC to Cloudflare and returned a successful validation response.

It made no Groq calls and added no sample text to public search. The upload endpoint defaults to validation and rejects attempts to commit a payload explicitly marked as a sample.

That was enough to verify the publishing connection. It was not enough to prove transcription quality or timestamp accuracy.

**As of October 12, 2026, the live app has 481 episode records and zero published transcripts.** The real-sermon trial is still pending. The weekly schedule is disabled, and the main runner remains in dry-run mode.

I want to test one real sermon next, listen against its transcript, and check whether search actually returns useful passages. Then I can expand the trial before processing the full archive.

## A weekly rhythm once it is ready

The n8n template is configured for Mondays at 6 a.m. Eastern, after the new podcast should usually have been posted.

When deliberately enabled, it can check for untranscribed episodes and process a limited batch. The first-run boundary keeps it focused on recent episodes; an archive backfill is an explicit choice rather than an accidental consequence of turning on a schedule.

Website deployments do not start audio processing. Configuring credentials does not start it either. I want those actions to remain distinct while I validate the pipeline.

## What comes next

The next milestone is a real message moving through the complete path: podcast audio, Groq transcription, local checkpoints, Cloudflare publication, search, and timestamped playback.

After that, I want to evaluate natural-language searches, work through the archive in controlled batches, and enable the weekly job once the results are dependable. Listening history and additional browsing filters are also possible future additions.

I wrote about [Skyline Pulse](https://blog.benstef.com/from-a-spreadsheet-idea-to-a-live-church-app-building-skyline-pulse) as a way to remember how a service went. Echo addresses a related question: how do we return to something that was said?

The project is open source, and the overview page explains both the experience and the architecture:

*   [Open Skyline Echo](https://skylinechurch.app/)
    
*   [Project overview](https://bstef.com/skyline-sermon-search/)
    
*   [Source code](https://github.com/bstef/skyline-sermon-search)
    
*   [n8n setup](https://github.com/bstef/skyline-sermon-search/blob/main/integrations/n8n/README.md)
    
*   [Publishing guide](https://github.com/bstef/skyline-sermon-search/blob/main/integrations/n8n/PUBLISHING.md)
    

If you have tried to find a specific moment in a sermon or another long recording, I would be interested in how you searched for it. The words you remember—and the words you do not—are what will make this useful to test.