# Anti-Derailment Patterns

Strategies for protecting focus and keeping the current sprint on track.

## The Shiny Object Problem

Vibe coders are creative people. Creativity means constant new ideas. The problem is not the ideas — it is acting on them immediately. Every context switch costs 15-30 minutes of re-focus time. Three switches in a session and you ship nothing.

## Pattern 1: The Gentle Redirect

**Trigger:** User says "I have an idea" or "what if we..." mid-task.

**Response:**
```
Love that idea! Let me capture it properly so we don't lose it.
[Ask 2-3 quick questions, score it, write to IDEAS.md]
Parked as IDEA-{NNN} with score {score}. Now back to {current task} —
we were working on {specific thing}.
```

**Key:** Acknowledge warmly, capture fast, redirect explicitly. Name the current task to re-anchor focus.

## Pattern 2: The Explicit Switch Request

**Trigger:** User says "let's build this now" or "forget what we're doing, this is more important."

**Response:**
```
You're currently building {current task} and you're about {progress}% through it.
Options:
1. Park this idea and finish {current task} first (recommended)
2. Explicitly switch — pause {current task} and start this instead

If you switch, I'll save your current progress so you can pick it back up later.
Which one?
```

**Key:** Make the cost of switching visible. Show progress on current task. Require explicit confirmation.

## Pattern 3: The Multi-Task Warning

**Trigger:** User has started 3+ tasks without finishing any.

**Response:**
```
Heads up — you have {N} open tasks right now:
1. {Task A} — {progress}
2. {Task B} — {progress}
3. {Task C} — {progress}

Starting another one will make it harder to finish any of them.
Let's pick ONE to finish first. Which one matters most right now?
```

**Key:** List all open tasks with progress. Gently force a priority decision.

## Pattern 4: The "Quick Detour" Trap

**Trigger:** User says "this will only take 5 minutes" or "let me just quickly..."

**Response:**
```
I hear you — but "5-minute detours" usually take 30+ minutes once you account for
context switching, testing, and edge cases. Let me park it with a high Ease score
so it's first in line when you're done with {current task}. Sound good?
```

**Key:** Name the cognitive bias. Offer a concrete alternative (high Ease = it will get built soon).

## Pattern 5: The Scope Creep Detector

**Trigger:** User starts adding features to the current task that were not in the original plan.

**Response:**
```
That's a great enhancement, but it's scope creep on {current task}.
The original plan was: {original scope}.
Want me to park this enhancement as a separate idea? You can build it
right after {current task} ships.
```

**Key:** Reference the original scope. Separate "enhancement" from "current task."

## Pattern 6: The Post-Ship Idea Review

**Trigger:** User just finished shipping a feature.

**Response:**
```
{Current task} is shipped! Nice work.

You have {N} ideas ready to build:
1. IDEA-{NNN} ({score}) — {title}
2. IDEA-{NNN} ({score}) — {title}
3. IDEA-{NNN} ({score}) — {title}

Want to review and pick one? Or take a break first?
```

**Key:** Only surface ideas AFTER shipping, not during. Give the option to rest.

## Pattern 7: The Dependency Alert

**Trigger:** User is building something that a parked idea depends on.

**Response:**
```
FYI — IDEA-{NNN} ({title}) depends on what you're building right now.
Once {current task} ships, you'll have everything needed to build it.
Want to queue it up next?
```

**Key:** Show that current work is unlocking future ideas. Motivation boost.

## Pattern 8: The Staleness Check

**Trigger:** An idea has been parked for 30+ days without any activity.

**Response:**
```
IDEA-{NNN} ({title}) has been parked for {N} days.
Is it still relevant? Options:
1. Keep it — still want to build this
2. Reprioritize — context has changed, rescore it
3. Kill it — no longer relevant
```

**Key:** Prevent idea debt. Old ideas that nobody acts on create mental overhead.

## The Golden Rules

1. **One active task at a time.** If you are building something, finish it before starting another.
2. **Ideas are not tasks.** Having an idea does not mean you should build it now.
3. **Capture everything, build selectively.** Writing it down means you will not forget. It does not mean you must act immediately.
4. **Context switches are expensive.** Every switch costs 15-30 minutes of re-focus time.
5. **Shipping beats starting.** One shipped feature is worth ten half-built ones.
6. **The backlog is your friend.** A well-organized backlog means the right idea surfaces at the right time.
