# ICE Scoring Framework

ICE stands for Impact, Confidence, Ease. Each dimension is scored 1-10. Total = I + C + E (max 30).

## Impact (1-10)

How much will this move the needle for users or the business?

| Score | Meaning | Example |
|-------|---------|---------|
| 1-2 | Cosmetic / nice-to-have | Change button color on settings page |
| 3-4 | Minor improvement | Add sorting to a table |
| 5-6 | Meaningful feature | Email notifications for status changes |
| 7-8 | High-value feature | Auto-fill forms from uploaded documents |
| 9-10 | Game-changer / core value | Real-time queue position tracking for applicants |

### Questions to determine Impact:
- How many users does this affect? (All users = high, niche = low)
- Does this solve a top-3 user pain point?
- Does this directly drive revenue or retention?
- Would users notice if we removed it?

## Confidence (1-10)

How sure are you this will actually work and users want it?

| Score | Meaning | Example |
|-------|---------|---------|
| 1-2 | Wild guess, no validation | "I think blockchain could help somehow" |
| 3-4 | Gut feeling, no data | "Users probably want dark mode" |
| 5-6 | Some signals | A few users mentioned it in feedback |
| 7-8 | Strong signals | Multiple support tickets, competitor has it |
| 9-10 | Validated demand | Users explicitly asked, data proves the need |

### Questions to determine Confidence:
- Have users asked for this?
- Do competitors have this feature?
- Do you have data showing the need?
- Have you built something similar before?

## Ease (1-10)

How easy is this to build with your current stack and skills?

| Score | Meaning | Example |
|-------|---------|---------|
| 1-2 | Months of work, new tech needed | Build a real-time video chat system |
| 3-4 | Multi-week, significant complexity | Custom payment splitting with escrow |
| 5-6 | Multi-session, moderate complexity | Dashboard with charts and filters |
| 7-8 | 1-2 sessions, known patterns | CRUD feature with Supabase + Next.js |
| 9-10 | Quick win, copy-paste patterns exist | Add a new API endpoint using existing patterns |

### Questions to determine Ease:
- Do you have a working pattern for this already?
- Does it need new infrastructure or third-party services?
- How many files/components need to change?
- Can you ship it in one focused session?

## Score Categories

| Total Score | Category | Action |
|-------------|----------|--------|
| 24-30 | Ready to Build | Ship it next. High impact, high confidence, easy to do. |
| 15-23 | Backlog | Good idea, park it. Build when capacity opens up. |
| 1-14 | Ice Box | Cool thought, but not worth the effort right now. Revisit later. |

## Scoring Examples for SaaS / Immigration Context

### Example 1: Real-time EOI Queue Position Tracker
- **Impact: 9** — Every applicant wants to know where they stand
- **Confidence: 8** — Users ask about this constantly in forums
- **Ease: 8** — Scraper + Supabase + simple UI, known patterns
- **Total: 25** — Ready to Build

### Example 2: AI-Powered Document Checklist Generator
- **Impact: 7** — Saves hours of manual checklist creation
- **Confidence: 6** — Some users mentioned it, not validated at scale
- **Ease: 5** — Needs Gemini integration + document parsing logic
- **Total: 18** — Backlog

### Example 3: Blockchain Document Verification
- **Impact: 4** — Sounds impressive but users care about speed, not blockchain
- **Confidence: 2** — No user has ever asked for this
- **Ease: 2** — Would need entirely new infrastructure
- **Total: 8** — Ice Box

### Example 4: Email Notification Preferences
- **Impact: 6** — Users get too many emails, want control
- **Confidence: 9** — Multiple unsubscribe complaints received
- **Ease: 9** — Simple preferences table + toggle UI
- **Total: 24** — Ready to Build

### Example 5: Multi-Language Support (Hindi/English)
- **Impact: 8** — Opens up to non-English-speaking users
- **Confidence: 7** — Large Hindi-speaking user base confirmed
- **Ease: 4** — Needs i18n framework, translate all content
- **Total: 19** — Backlog

## When to Rescore

Rescore ideas when:
- New user feedback arrives that validates or invalidates the idea
- A dependency gets shipped (Ease increases)
- A competitor launches the feature (Impact and Confidence increase)
- The tech stack changes (Ease changes)
- Business priorities shift (Impact changes)
- More than 30 days have passed since last score
