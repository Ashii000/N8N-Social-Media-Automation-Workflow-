# N8N Social Media Automation Workflow

An end‑to‑end, AI‑driven **n8n** workflow that turns a single news link into fully researched, AI‑written, AI‑illustrated, human‑approved social media posts — automatically published to **LinkedIn, X (Twitter), Facebook, and Instagram**.

Drop a news link into a Google Sheet → the workflow researches it, summarizes it, generates a matching AI image, writes a platform‑specific caption for each network, sends it to you for one‑tap approval by email, and — once approved — logs it and publishes it live.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Workflow Architecture](#workflow-architecture)
- [Tech Stack & Integrations](#tech-stack--integrations)
- [Detailed Workflow Breakdown](#detailed-workflow-breakdown)
- [Google Sheet Structure](#google-sheet-structure)
- [Setup & Installation](#setup--installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Customization](#customization)
- [Notes & Limitations](#notes--limitations)
- [Author](#author)

---

## Overview

This repository contains a single exportable n8n workflow, **`Social Media Post.json`**, that automates the entire content pipeline for a company or personal brand's social presence:

**Research → Summarize → Illustrate → Write (x4 platforms) → Approve (human‑in‑the‑loop) → Publish → Log.**

It is designed so that a non‑technical team member only has to do two things: (1) paste a news/article link into a Google Sheet, and (2) tap **Approve** or **Reject** on an email for each platform's draft. Everything else — research, writing, image generation, and publishing — is fully automated.

## Key Features

- **Zero‑touch trigger** — a Google Sheets Trigger polls every minute for a newly added row containing a `News Link`.
- **AI web research** — the article is fetched and analyzed via the **Tavily Search API** (advanced search depth) rather than a simple scrape, giving the AI richer context to summarize from.
- **AI summarization** — a GPT‑4o‑mini powered LangChain agent condenses the article into a tight, ≤60‑word, fact‑focused summary (no fluff, no filler).
- **AI image generation** — a second agent turns that summary into a structured, photorealistic image‑generation prompt (subject / action / setting / details), which is sent to an image model (`google/nano-banana` via the Kie.ai API) to produce a ready‑to‑post visual.
- **Four dedicated copywriting agents** — separate GPT‑4o‑mini agents, each with its own expert system prompt and brand‑voice rules, write a caption tailored to that platform's format and etiquette:
  - **Instagram** — 1–2 sentence hook‑driven caption, max 2 emojis, 2–3 hashtags
  - **Facebook** — full hook/context/story/CTA structure, 5–15 hashtags, paragraph‑spaced formatting
  - **LinkedIn** — 1,000–1,500 character thought‑leadership post with data points, a discussion question, and 5–10 hashtags
  - **X / Twitter** — punchy 100–150 character post with 3–5 hashtags, no links
- **Human‑in‑the‑loop approval** — before anything goes live, each platform's draft (caption + generated image) is emailed via Gmail's **Send‑and‑Wait (double approval)** node, so a real person approves or rejects every post per platform.
- **Automatic logging** — every approved post's caption and image URL is appended to its own tab in the master Google Sheet (Instagram / Facebook / LinkedIn / X‑Twitter posts), creating a running content archive.
- **Automatic publishing** — approved posts are pushed live to each platform through the **Upload‑Post API**, with the generated image downloaded and attached automatically.

## Workflow Architecture

```
Google Sheet (new "News Link" row)
        │
        ▼
   Limit (last item)
        │
        ▼
  Edit Fields → build research query
        │
        ▼
  Tavily Search API  ──►  Summary Agent (GPT-4o-mini)
                                  │
                                  ▼
                     Create Image Prompt Agent (GPT-4o-mini)
                                  │
                                  ▼
                     Kie.ai Image Generation (nano-banana)
                                  │
                                  ▼
                        Wait (webhook callback)
                                  │
                                  ▼
                    Code node → extract image URL
                                  │
        ┌─────────────┬──────────┼───────────────┬──────────────┐
        ▼             ▼          ▼                ▼              
  Instagram Agent  Facebook Agent  LinkedIn Agent   X/Twitter Agent
        │             │             │                │
        ▼             ▼             ▼                ▼
  Gmail: Send & Wait (double approval) — per platform
        │             │             │                │
        ▼             ▼             ▼                ▼
      If approved?  If approved?  If approved?    If approved?
        │             │             │                │
        ▼             ▼             ▼                ▼
  Append to Sheet  Append to Sheet Append to Sheet Append to Sheet
        │             │             │                │
        ▼             ▼             ▼                ▼
  Download Image   Download Image  Download Image  Download Image
        │             │             │                │
        ▼             ▼             ▼                ▼
  Post to Instagram Post to Facebook Post to LinkedIn Post to X/Twitter
        (via Upload-Post API, one branch per platform)
```

## Tech Stack & Integrations

| Component | Purpose |
|---|---|
| **n8n** | Workflow orchestration engine |
| **Google Sheets** | Trigger source (new article links) + content log/archive (per‑platform tabs) |
| **Tavily Search API** | AI‑oriented web research/retrieval for the source article |
| **OpenAI `gpt-4o-mini`** (via `@n8n/n8n-nodes-langchain`) | Summarization + 4x platform‑specific copywriting agents + image‑prompt engineering agent |
| **Kie.ai API** (`google/nano-banana` model) | Text‑to‑image generation for the post visual |
| **Gmail node — Send and Wait (double approval)** | Human‑in‑the‑loop review/approval gate, one email per platform |
| **Upload‑Post API** | Final multi‑platform publishing (Instagram, Facebook, LinkedIn, X/Twitter) |

## Detailed Workflow Breakdown

1. **Google Sheets Trigger** — polls every minute; fires when a new row is added to the `Article` tab of the *Social media posts* spreadsheet.
2. **Limit** — keeps only the most recently added row, so the pipeline processes one article at a time.
3. **Edit Fields** — builds the research query: `Summarise and analyse sentiment for this article: {{News Link}}`.
4. **HTTP Request (Tavily)** — sends the query to `api.tavily.com/search` with `search_depth: advanced` to retrieve rich article content.
5. **Summary Agent** — a LangChain agent (GPT‑4o‑mini) reduces the retrieved content to a ≤60‑word factual summary.
6. **Create Image Prompt** — a second agent converts that summary into a structured photorealistic image prompt (subject, action, setting, details, fixed style suffix: *natural, realistic, 8K, taken on iPhone, --ar 16:9*).
7. **HTTP Request → Kie.ai** — submits the prompt as an image‑generation job (`google/nano-banana` model) with a callback URL.
8. **Wait (webhook)** — pauses the workflow until Kie.ai's callback delivers the finished image.
9. **Code node** — parses the callback payload and extracts the final image URL.
10. **Four parallel content branches** (Instagram, Facebook, LinkedIn, X/Twitter):
    - A dedicated GPT‑4o‑mini agent writes the platform‑appropriate caption from the article + summary.
    - A **Gmail "Send and Wait" (double approval)** email is sent containing the caption and the generated image link.
    - An **If** node checks whether the recipient approved the post.
    - On approval: the caption + image URL are **appended to that platform's tab** in the Google Sheet, the image is **downloaded**, and the post is **published live** via the Upload‑Post API.
    - On rejection: the branch simply stops — nothing is logged or published.

## Google Sheet Structure

The workflow expects one Google Sheet ("Social media posts") with the following tabs:

| Tab | Columns | Purpose |
|---|---|---|
| `Article` | `News Link` (and any notes) | Input — drop a new article/news URL here to trigger a run |
| `Instagram posts` | `Posts`, `Image` | Log of approved, published Instagram captions + images |
| `Facebook posts` | `Posts`, `Image` | Log of approved, published Facebook captions + images |
| `Linkedin Posts` | `Posts`, `Image` | Log of approved, published LinkedIn captions + images |
| `X/Twitter posts` | `Posts`, `Image` | Log of approved, published X/Twitter captions + images |

## Setup & Installation

1. **Import the workflow**
   - Open your n8n instance → **Workflows → Import from File** → select `Social Media Post.json`.
2. **Connect credentials** (n8n → Credentials):
   - Google Sheets OAuth2 (Trigger + all 4 "Append row" nodes)
   - Gmail OAuth2 (all 4 "Send and Wait" approval nodes)
   - OpenAI API key (all `lmChatOpenAi` nodes — summary, image‑prompt, and the 4 platform agents)
3. **Add your API keys** in the relevant HTTP Request nodes (replace the `YOUR_API_KEY` placeholders):
   - Tavily Search API key (`HTTP Request` node)
   - Kie.ai API key (`HTTP Request1` node)
   - Upload‑Post API key + your Upload‑Post username (all 4 `Post To …` nodes)
4. **Point it at your own Google Sheet**
   - Update the `documentId` / `sheetName` fields on the Trigger and all 4 "Append row in sheet" nodes to match your copy of the spreadsheet (structure per the table above).
5. **Expose a public webhook URL** for the `Wait` node so Kie.ai's callback can resume the workflow (n8n Cloud does this automatically; self‑hosted instances need a reachable `WEBHOOK_URL`).
6. **Activate the workflow.**

## Configuration

| What | Where to change it |
|---|---|
| Research query phrasing | `Edit Fields` node |
| Summary length/style | `Summary Agent` → system message |
| Image style/aspect ratio | `Create Image Prompt` → system message (currently fixed to realistic, 8K, 16:9) |
| Image model | `HTTP Request1` → `model` field (currently `google/nano-banana`) |
| Per‑platform tone, length, hashtag count | Each platform's Agent node (`Instagram Agent`, `Facebook Agent`, `LinkedIn Agent`, `X/Twitter Agent`) → system message |
| Approval email subject/body | Each `Send a message…` Gmail node |
| Destination sheet/tab per platform | Each `Append row in sheet…` node |

## Usage

1. Add a new row with a `News Link` to the `Article` tab of your Google Sheet.
2. Within a minute, the workflow triggers automatically and runs research → summary → image generation → the four caption‑writing branches.
3. You'll receive **up to 4 separate approval emails** (one per platform) — each containing the AI‑generated caption and a link to the generated image.
4. Tap **Approve** on the platforms you want published; tap **Reject** (or ignore) the ones you don't.
5. Approved posts are automatically logged to the Google Sheet and published live to the corresponding platform.

## Customization

- **Add more platforms**: duplicate one of the four branches (Agent → Send & Wait → If → Append row → Download Image → Post) and point the `platform[]` parameter in the final HTTP Request node to the new network.
- **Swap the image model**: change the `model` value in the Kie.ai `HTTP Request1` node to any model supported by Kie.ai's `createTask` endpoint.
- **Change the research source**: replace the Tavily `HTTP Request` node with any other search/scraping API — the rest of the pipeline only depends on receiving article text.
- **Single‑approval instead of double**: adjust `approvalType` on the Gmail nodes from `double` to `single` if you don't need separate approve/reject buttons.

## Notes & Limitations

- All API keys in the exported JSON are placeholders (`YOUR_API_KEY`) — they must be replaced or, preferably, moved into n8n Credentials before running.
- The Google Sheet `documentId`/`sheetName` values are hardcoded to the original author's sheet and must be updated to point to your own spreadsheet.
- The `Wait` node depends on a publicly reachable webhook, so self‑hosted n8n instances need a properly configured `WEBHOOK_URL`/reverse proxy for the image‑generation callback to resume the workflow.
- Publishing depends on the third‑party **Upload‑Post** service being connected to your actual Instagram, Facebook, LinkedIn, and X accounts.

## Author

**Aushique Hussain** — AI Developer & Automation Engineer (n8n, Python, LLM workflows)
[LinkedIn](https://linkedin.com/in/ashiq-mari-5abb33277)
