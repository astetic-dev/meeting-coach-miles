# Sample workspace — Acme Logistics WMS replacement

This folder is **not part of the coach folder**. It is a worked example of what a user's working data looks like after a few sessions with the coach.

## Why this exists

The meeting coach reads and writes three artifacts:

- **project.json** — the project-card; one per project, the anchor every meeting analysis hangs off
- **contacts/CONTACT-{ORG}-{slug}.json** — one per person who shows up across meetings
- **meetings/MTG-{project}-{date}-{nn}.json** — one per meeting reflection session

These artifacts are JSON files validated by the schemas in `coach/reference/schemas/`. They form a small connected graph: project ←→ contacts ←→ meetings.

This sample shows them in a realistic shape so a reviewer can:

1. See what "after three sessions with the coach" actually looks like
2. Read the analysis style on `meetings/MTG-ACME-WMS-2026-05-10-01.json` and `2026-05-17-01.json` and judge whether it coaches or lectures
3. Drop this whole folder into a Claude project alongside the coach folder and try follow-up sessions cold

## The fictional setup

**Acme Logistics B.V.** is replacing their 15-year-old in-house WMS with **Stowline** (SaaS), live in three warehouses (Rotterdam, Antwerp, Düsseldorf), targeting go-live October 1.

**Sarah Chen** is an external project manager facilitating the engagement. She has been keeping meeting-cards on her two recurring meetings:

- **Steering** with Mark Thompson (sponsor) and Priya Natarajan (customer PMO)
- **Vendor sync** with Jasmine Ali (Stowline PM); David Okonkwo (Rotterdam warehouse manager) sometimes joins

The two meeting-cards in `meetings/` are her last two reflection sessions with the coach.

## How a reviewer can use this in five minutes

1. Open `project.json` and skim — that is the project context the coach pulls into every session.
2. Open `meetings/MTG-ACME-WMS-2026-05-17-01.json` and read `analysis.growth_points` and `analysis.pattern_read` end to end.
3. Ask yourself: is this coaching, or is this consulting? If it reads as "next time do X" — the coach is failing. If it reads as "here is the moment, here is what I noticed, what do you think was happening?" — the coach is working.
4. Drop the snippet in `meetings/transcripts/2026-05-17-vendor-sync-snippet.md` into a fresh chat with the coach to generate your own version and compare.

## What's intentionally NOT in here

- Action items, minutes, agendas — those are out of scope by design
- Full transcripts — only excerpts; the coach reads transcripts inline at session time, not from disk
