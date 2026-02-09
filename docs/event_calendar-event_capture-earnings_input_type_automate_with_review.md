the calendar is both a user-facing feature and an operational trigger. Let me break this into the two problems: knowing what's coming, and capturing what happened.

------

**1. Building the event calendar: sourcing upcoming dates**

The good news is that most of these dates are announced well in advance and come from structured sources:

**Earnings calls** (this is the gap in your spec — worth adding)

- Earnings dates are published on investor relations pages and aggregated by services like Nasdaq's earnings calendar, Seeking Alpha, or data feeds from providers like Wall Street Horizon or Refinitiv. Most are known 2–4 weeks out, sometimes a quarter ahead.
- For your Phase 1 companies (OpenAI is private, so no earnings), you'd track Microsoft, Google/Alphabet, Amazon/AWS, and later Apple, Meta, NVIDIA.

**Developer conferences / keynotes**

- Dates are announced months ahead on official event pages (Build, I/O, re:Invent, WWDC, GTC, Connect).
- These are low-volume, high-signal — you can manually seed them and set a periodic check for new announcements.

**Product launch events / livestreams**

- Less predictable, especially for OpenAI. These often get 1–2 weeks notice via blog posts or social.
- This is where monitoring matters more than a static calendar.

**Practical approach:**

- Maintain a curated `UpcomingEvent` table with source URLs, expected dates, and a confidence level (confirmed vs. rumored).
- Seed quarterly from IR calendars and official event pages.
- Supplement with an RSS/feed monitor on key blogs and newsrooms for surprise announcements.
- Surface this calendar in the product — analysts would genuinely value a single view of "what's coming across all tracked companies."

------

**2. Capturing information post-event: methods by speed**

This is where your operational cadence lives. Several approaches, roughly ordered by latency:

**Near-real-time (minutes to hours)**

- **RSS / webhook monitors on official sources** — AWS "What's New," Microsoft 365 Roadmap, Google Cloud release notes, OpenAI blog. These are structured feeds. When a new entry appears, ingest it into your `raw_release_items` pipeline automatically.
- **Livestream transcript capture** — For keynotes, pull the auto-generated captions (YouTube provides these) or use a speech-to-text service on the livestream. This gives you raw promise candidates within minutes of the event ending. Your spec already describes this for WWDC — extend it to all events.
- **Official recap / "Book of News" scraping** — Microsoft and AWS publish structured recap pages same-day. Monitor the URL pattern and trigger ingestion as soon as it's live.

**Same-day (hours)**

- **LLM-assisted extraction** — Once you have the transcript or recap text, run your Prompt 1 (split into atomic commitments) immediately. This is your fastest path to a draft scorecard.
- **Press release monitoring** — Company newsrooms and PR wires (Business Wire, PR Newswire). These often go live during or right after the keynote.

**Next-day (24–48h)**

- **Manual review and correction** — Human editor reviews the LLM-extracted commitments, fixes scope, deduplicates, filters out ineligible items (minor tweaks/bugfixes), and publishes the scorecard. This is the quality gate.
- **Earnings transcript ingestion** — Earnings call transcripts are available within hours from providers (Seeking Alpha, The Motley Fool for free; FactSet, Bloomberg for paid). You'd extract product/feature claims from the Q&A and prepared remarks, then match them against existing commitments or flag new ones.

**Ongoing (weekly)**

- **Evidence matching** — Your existing weekly refresh process. Re-scan shipping sources for all non-GA commitments.

------

**3. Earnings as a new input type (the gap you identified)**

Your spec doesn't cover earnings, but they're important for two reasons: companies make forward-looking product claims on calls, and they sometimes quietly confirm or walk back prior commitments. I'd suggest:

- Add `EARNINGS` as an `event_type` (you already have it in the enum but don't use it).
- Treat the earnings transcript as an announcement source — extract product claims the same way you'd extract from a keynote recap.
- Also use earnings transcripts as *evidence* for existing commitments — if a CEO says "we shipped X to all enterprise customers last quarter," that's an `EARNINGS_IR` evidence record against an existing commitment.

------

**4. Recommended workflow: automated with review**

The sweet spot is probably **auto-ingest → LLM draft → human review → publish**, with different SLAs:

| Event type           | Ingest trigger                         | Draft scorecard          | Published (reviewed) |
| -------------------- | -------------------------------------- | ------------------------ | -------------------- |
| Major keynote        | Transcript + recap page, auto-detected | Within 2h (LLM)          | Within 24–48h        |
| Earnings call        | Transcript availability alert          | Within 4h (LLM)          | Within 48h           |
| Product launch post  | RSS/blog monitor                       | Within 1h (LLM)          | Within 24h           |
| Weekly release notes | Scheduled scrape                       | Automatic classification | Same-day review      |

The key principle: **never publish without human review**, but get the draft ready automatically so the human is editing, not creating from scratch. That's how you hit the "scorecard within 48h" target from your business plan.