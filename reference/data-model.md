# Data model — three cards, one graph

Miles works on top of three connected JSON artifacts. Together they form a small graph that grows with the user's meetings over time.

```
┌──────────────────────────┐
│      project-card        │  ←── one per project (project.json)
│     "the anchor"         │      contains: purpose, scope, status,
│                          │                 stakeholders[], risks
└────────────┬─────────────┘
             │
             │ stakeholders[].contact_id ────────┐
             │                                    │
             ▼                                    │
┌──────────────────────────┐                     │
│      meeting-card        │                     │
│   "the reflection"       │  ←── one per        │
│                          │      session        │
│   contains: user_intent, │                     │
│   analysis (strengths,   │                     │
│   growth_points,         │                     │
│   reflection_questions,  │                     │
│   pattern_read),         │                     │
│   user_reflections[]     │                     │
└────────────┬─────────────┘                     │
             │                                    │
             │ attendees[].contact_id ───────────┤
             │ previous_meeting_card_id          │
             │   (longitudinal chain)            │
             ▼                                    ▼
                    ┌──────────────────────────┐
                    │      contact-card        │  ←── one per person
                    │     "the people"         │      (centralized in contacts/)
                    │                          │
                    │   contains: name, role,  │
                    │   behavioral_obs[],      │
                    │   involved_in[]          │
                    └──────────────────────────┘
```

## Relations and cardinality

Three relationship types, each with specific cardinality:

| Relation | Cardinality | Implementation |
|---|---|---|
| project ↔ meeting | **1-to-many** | Each meeting-card has exactly one `project_id`. A project accumulates many meetings over time. |
| project ↔ contact | **many-to-many** | Each contact-card lists its projects in `involved_in[]`. Each project-card lists its key stakeholders in `stakeholders[]` (with `contact_id` pointers). Same person can be involved in multiple projects; same project has multiple people. |
| meeting ↔ contact | **many-to-many** | Each meeting-card lists its attendees in `attendees[]` (with `contact_id` pointers). Same person attends multiple meetings; same meeting has multiple attendees. |

Contact-cards exist for **every** person who appears in the user's working context — not just the people Miles has observed something interesting about. Two consequences:

- A contact-card can exist for a stakeholder who is on the project-card but has not yet attended any meeting (e.g., the steering-group sponsor who only comes to the quarterly meeting). Their card is identity-only until they show up, and Miles uses the gap itself as a signal ("the sponsor has not been in any of the last three steerings").
- A contact-card persists after the person departs (`active: false`, `departed_at: YYYY-MM-DD`); historical meeting-cards still reference them by id, so the longitudinal view stays intact.

## Why three artifacts, not one big record

Each artifact answers a different question, and each has a different lifecycle:

| Artifact | Question it answers | Lifecycle |
|---|---|---|
| **project-card** | "What is this project for and where is it now?" | Created once per project; updated when status, stakeholders, or risks meaningfully change. |
| **meeting-card** | "What happened in this specific meeting and what did the user reason from it?" | Created once per coaching session; immutable analysis, append-only `user_reflections[]`. |
| **contact-card** | "Who is this person across all my meetings with them?" | Created when a person first appears across sessions; `behavioral_observations[]` accumulates over time. |

A single mega-record per session would force duplication. Mark Thompson appears in seven steering meetings — his behavioral patterns belong in *one* place that all seven reference, not seven copies that drift. The contact-card carries the cross-meeting view; the meeting-card carries the in-session view; the project-card carries the why-are-we-here view.

## ID conventions

All ids are uppercase, hyphenated, and pattern-validated by the schemas. They are designed to be readable, short, and to encode the relationship between artifacts.

### Project id

```
{customer_code}-{project_code}
```

- `customer_code`: 2–10 uppercase letters, the customer's short code (e.g., `ACME`, `NBRG`, `STOWLINE`).
- `project_code`: 2–7 uppercase letters, the project's short code (e.g., `WMS`, `CRM`, `ETL`).

Examples: `ACME-WMS`, `NBRG-CRM`, `STOWLINE-MIGRATION`.

Project-cards have no user prefix because a project may be shared across users in larger setups.

### Meeting id

```
MTG-{customer_code}-{project_code}-{YYYY-MM-DD}-{NN}
```

- `MTG-` prefix marks the artifact type.
- The `{customer_code}-{project_code}` portion must match the linked project-card id exactly.
- `YYYY-MM-DD` is the meeting date.
- `NN` is a 2-digit ordinal for the same project + date — `01` for the first meeting that day, `02` for the second, etc.

Examples: `MTG-ACME-WMS-2026-05-17-01`, `MTG-NBRG-CRM-2026-05-24-01`.

### Contact id

```
CONTACT-{org_code}-{name-slug}
```

- `CONTACT-` prefix marks the artifact type.
- `org_code`: 2–10 uppercase letters; the person's organization. May be a customer code (`ACME`), a vendor code (`STOWLINE`), or a generic bucket (`INTERNAL`, `THIRDPARTY`).
- `name-slug`: lowercase, dashes, no diacritics.

Examples: `CONTACT-ACME-mark-thompson`, `CONTACT-STOWLINE-jasmine-ali`, `CONTACT-INTERNAL-sarah-chen`.

## File layout

A working workspace looks like this:

```
{project-folder}/
├── project.json                           ← the project-card
├── contacts/
│   ├── CONTACT-ACME-mark-thompson.json
│   ├── CONTACT-ACME-priya-natarajan.json
│   ├── CONTACT-STOWLINE-jasmine-ali.json
│   └── CONTACT-INTERNAL-{user-slug}.json
└── meetings/
    ├── MTG-ACME-WMS-2026-05-10-01.json    ← meeting-card
    ├── MTG-ACME-WMS-2026-05-17-01.json
    └── transcripts/                       ← optional; user-managed
        └── 2026-05-17-vendor-sync.md
```

Miles reads `project.json` first to anchor, then reads matching meeting-cards (sorted by date) for the longitudinal view, then reads contact-cards for attendees in the current transcript.

## How references resolve

When Miles encounters `attendees[].contact_id: "CONTACT-ACME-mark-thompson"` in a meeting-card, it reads `contacts/CONTACT-ACME-mark-thompson.json` to get:

- The current role (in case it changed since this meeting)
- Past behavioral observations Miles has saved
- Other meetings this person has been involved in

If `contact_id` is null or the file is missing, Miles falls back to the `name` field on the attendee and treats this person as new. Across two or three sessions, Miles will recognize the pattern and offer to create a contact-card for recurring attendees.

## What the user owns vs. what Miles writes

| Field | Owner | Notes |
|---|---|---|
| `project-card.*` | Co-authored at first session | Miles proposes JSON; user confirms. Updated by user as project evolves. |
| `meeting-card.user_intent` | User | Required input before Miles will analyze. |
| `meeting-card.user_self_assessment` | User | Optional; written before Miles reads the transcript. |
| `meeting-card.analysis.*` | Miles | The coach output. Immutable once written. |
| `meeting-card.user_reflections[]` | User | Append-only log. The user reasons here. |
| `contact-card.behavioral_observations[]` | Miles + user | Miles proposes; user can edit or reject. |
| `contact-card.notes` | User | Working notes, free-text. |

## Versioning

All three schemas use `schema_version: 1`. If the schemas evolve, future versions will be additive where possible (new optional fields) and breaking only when necessary. Breaking changes increment the schema_version; tooling validates against the version declared in each file.

## What this model deliberately does NOT track

- **Action items / decisions / minutes.** Out of scope for Miles — these are documentation, not reflection.
- **Time-on-task / effort logging.** Project-management tooling, not coaching.
- **Sentiment scores / numeric quality ratings.** Numeric scoring of facilitation quality is what the consultant-pretending-to-coach mode produces. Miles does not do numbers.
- **Action plans for next meeting.** Miles asks reflection questions; the user authors their own next-meeting intent in their own words, in the next session's `user_intent`.
