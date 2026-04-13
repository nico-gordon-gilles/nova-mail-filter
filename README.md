# Nova Mail Filter

> An AI-powered email processing pipeline that turns an inbox into a managed conversation system.

Nova Mail Filter runs every 5 minutes as a cron job. It reads incoming emails, classifies them, generates context-aware replies via a Gemini sub-agent, and pushes notifications — all without human intervention unless a reply needs approval.

---

## What It Does

```
Gmail inbox (nova.ai.main@gmail.com)
         ↓  every 5 minutes
   Mail Filter (Node.js cron job)
         ↓
   Classify sender
   ├── known sender? → load conversation history
   ├── first contact? → intro text + classify intent
   ├── bounce/spam?  → ignore
   └── banned?       → hardcoded rejection, no LLM
         ↓
   Gemini Sub-Agent
   ├── reads mail + last 4 messages history
   ├── generates structured response
   └── returns: { reply, category, confidence, action }
         ↓
   Push notification to Nico
         ↓
   Store in SQLite
```

---

## Architecture

```
┌─────────────────────────────────────────────┐
│              Gmail API (OAuth2)             │
└──────────────────┬──────────────────────────┘
                   │ poll every 5 min
                   ▼
┌─────────────────────────────────────────────┐
│           mail-filter.js (cron)             │
│                                             │
│  ┌─────────────┐    ┌────────────────────┐  │
│  │ Load Shed   │    │  Circuit Breaker   │  │
│  │ >50 pending │    │  3 failures → open │  │
│  └─────────────┘    └────────────────────┘  │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │         Gemini Sub-Agent            │    │
│  │   structured output · 30s timeout  │    │
│  │   sliding window: last 4 messages  │    │
│  │   max input: 15,000 chars          │    │
│  └─────────────────────────────────────┘    │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
            SQLite (panel.db)    +    Push → Nico
```

---

## Production Safeguards

The system is designed to run unattended without causing problems:

| Safeguard | Value | Purpose |
|---|---|---|
| Global runtime limit | 100s | cron job never runs over |
| Gemini timeout | 30s per call | no hanging API calls |
| Max mail body | 10KB | no prompt injection via giant emails |
| Max mail size | 1MB | skip attachments |
| Sliding window | last 4 messages | context without explosion |
| Max input chars | 15,000 | controlled token budget |
| Max new senders/run | 10 | no burst processing |
| Load shed threshold | 50 pending | skip LLM if queue is backed up |
| Pending TTL | 30 min | stale items auto-expire |
| Push rate limit | 10 min/sender | no notification spam |
| Circuit breaker | 3 failures → open | stop hammering broken APIs |

---

## Sub-Agent Design

The Gemini sub-agent receives a structured prompt and returns structured output:

```javascript
// Input to sub-agent
{
  system: "You are Nova, an AI assistant answering emails for Nico...",
  mail: {
    from: "sender@example.com",
    subject: "Question about project",
    body: "...",                    // max 10KB
  },
  history: [                        // last 4 exchanges
    { role: "user", content: "..." },
    { role: "assistant", content: "..." }
  ]
}

// Structured output
{
  reply: "...",
  category: "inquiry" | "spam" | "newsletter" | "important",
  confidence: 0.94,
  action: "reply" | "ignore" | "flag_for_nico"
}
```

---

## Sender Classification

```
First contact:
  → intro text prepended: "Hi, I am Nova, an AI assistant for Nico."
  → classify intent

Known sender:
  → load last 4 messages as context
  → reply in established tone

Bounce patterns (mailer-daemon, noreply, postmaster...):
  → silently ignored, no LLM call

Banned sender:
  → hardcoded rejection text, NO LLM call
  → reason: banned senders should not consume tokens
```

---

## Reliability Features

**Exponential backoff** on all Google API calls:
```javascript
async function withBackoff(fn, retries = 3) {
  let delay = 1000;
  for (let i = 0; i <= retries; i++) {
    try { return await fn(); }
    catch (e) {
      if (isRetryable(e.status)) {
        await sleep(delay);
        delay *= 2;
      } else throw e;
    }
  }
}
```

**Circuit breaker** — after 3 consecutive Gemini failures, the pipeline stops processing and waits for the next cron cycle. Prevents cascading failures from a flaky API.

**Heartbeat tracking** — each run writes a timestamp. If the last heartbeat is older than 130 seconds, the system flags itself as stale.

---

## Tech Stack

- Node.js (cron job, Gmail API integration)
- Google APIs (Gmail via OAuth2)
- Gemini API (structured output, sub-agent)
- SQLite with WAL mode (conversation history, state)
- Push notification system (Nova Bridge integration)

---

## Part of Nova

Nova Mail Filter is one component of **Nova** — an autonomous AI assistant system.

🌐 [neural-werk.de](https://neural-werk.de) · [GenesisBrain](https://github.com/nico-gordon-gilles/nova-genesisbrain) · [Autonomi](https://github.com/nico-gordon-gilles/nova-autonomi) · [STM](https://github.com/nico-gordon-gilles/nova-stm)

