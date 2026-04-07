# Idea Lifecycle Commands

Every command with usage examples.

## `park` — Capture and Score a New Idea

The most common command. Captures an idea without interrupting current work.

**User triggers:**
- "I have an idea"
- "What if we added..."
- "We should also build..."
- "Save this for later"
- "Park this idea"

**Flow:**
1. Acknowledge the idea warmly
2. Ask: What problem? Who benefits? Rough scope?
3. Score with ICE (Impact, Confidence, Ease — each 1-10)
4. Write to IDEAS.md under the appropriate category
5. Redirect back to current task

**Example:**

User: "What if we added a feature where users can track their EOI queue position?"

Assistant response:
1. "Love that idea! Quick questions — what problem does this solve?"
2. User: "People keep refreshing the immigration site manually"
3. "Who benefits most?" User: "All 189/190 visa applicants"
4. "Rough scope?" User: "Scrape the site, show position in dashboard"
5. Score: I:9 C:8 E:8 = 25 (Ready to Build)
6. Write to IDEAS.md, redirect: "Parked as IDEA-007 (score 25). Back to the auth flow..."

## `review` — Show Current Idea Board

Displays IDEAS.md sorted by priority with actionable summary.

**User triggers:**
- "Review ideas"
- "What's in the backlog?"
- "What should I build next?"
- "Show me my ideas"

**Flow:**
1. Read IDEAS.md from project root
2. Summarize counts by category
3. Highlight top ready ideas, stale ideas, and dependency matches
4. Ask what action to take

**Example output:**

You have 12 parked ideas:
- 3 Ready to Build (top: IDEA-007, score 25)
- 1 In Progress (IDEA-004)
- 5 Backlog
- 3 Ice Box
- Stale: IDEA-002 (42 days old)
Want to start a ready idea, reprioritize, or kill stale ones?

## `promote` — Move an Idea Up

Moves an idea from Backlog to Ready, or Ice Box to Backlog.

**User triggers:**
- "Promote IDEA-005"
- "Move this idea to ready"
- "This idea is more important now"

**Flow:**
1. Find the idea in IDEAS.md
2. Ask if ICE scores should change (context may have shifted)
3. Rescore if needed
4. Move to the higher category
5. Update IDEAS.md

**Example:**

Promoting IDEA-005 (Dashboard analytics) from Backlog to Ready.
Current score: I:7 C:6 E:7 = 20
Has anything changed? New score: I:8 C:8 E:7 = 23
Still Backlog (needs 24+). Bump Ease? If scraper is done: E:9 = 25. Ready!

## `start` — Begin Working on an Idea

Moves an idea to "In Progress" and sets it as the active task.

**User triggers:**
- "Let's build IDEA-007"
- "Start the queue tracker"
- "I want to work on this idea"

**Flow:**
1. Check if there is a current active task
2. If yes: warn and ask to finish first or explicitly switch
3. If no: move idea to In Progress, update IDEAS.md
4. Begin implementation

**Guard rail — mid-task start attempt:**

You are currently building the auth flow (about 60% done).
Options:
1. Park IDEA-007 and finish auth first (recommended)
2. Pause auth and switch to IDEA-007
Which one?

## `ship` — Mark an Idea as Shipped

Celebrates completion and moves to the Shipped section.

**User triggers:**
- "IDEA-007 is done"
- "We shipped it"
- "Mark this as complete"

**Flow:**
1. Update idea status to Shipped
2. Add ship date and PR/commit link
3. Move to Shipped section in IDEAS.md
4. Check if shipping this unblocks other ideas
5. Surface unblocked ideas

**Example:**

IDEA-007 shipped! Updated IDEAS.md.
This unblocks:
- IDEA-011 (score 22) was waiting on the scraper you just built
- IDEA-015 (score 18) can now use the queue data
Want to promote either of these?

## `kill` — Archive an Idea

Removes an idea from active consideration with a documented reason.

**User triggers:**
- "Kill IDEA-003"
- "Remove that idea"
- "That idea is no longer relevant"
- "Duplicate of IDEA-007"

**Flow:**
1. Find the idea
2. Ask for the reason (duplicate, out of scope, tech not ready, need changed)
3. Move to Killed section with reason and date
4. Update IDEAS.md

**Valid kill reasons:**
- Duplicate of another idea
- Out of scope for this product
- Technology not ready or too complex
- User need has changed or disappeared
- Superseded by a better approach
- Not aligned with business goals

## `reprioritize` — Rescore All Active Ideas

Recalculates ICE scores based on new context.

**User triggers:**
- "Reprioritize ideas"
- "Scores are outdated"
- "Context has changed"
- "We shipped X, update priorities"

**Flow:**
1. List all active ideas (Ready + Backlog + Ice Box)
2. For each, show current score and ask if context changed
3. Rescore any that changed
4. Re-sort into categories based on new scores
5. Show before/after comparison

**Example:**

Reprioritizing 8 active ideas after shipping the scraper module...

Changed:
- IDEA-011: 20 -> 26 (Ease went from 5 to 9, scraper now available) — PROMOTED to Ready
- IDEA-003: 14 -> 8 (Killed competitor feature, Impact dropped) — stays Ice Box

Unchanged: IDEA-005, IDEA-008, IDEA-009, IDEA-012, IDEA-014, IDEA-016