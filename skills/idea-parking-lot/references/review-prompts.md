# Idea Review Conversation Templates

Use these templates for weekly and monthly idea reviews.

## Weekly Quick Review (5 minutes)

```
Time for your weekly idea review! Here's the current state:

Total ideas: {N}
  Ready to Build: {N}
  In Progress: {N}
  Backlog: {N}
  Ice Box: {N}
  Shipped this week: {N}
  Killed this week: {N}

Top 3 ready ideas:
1. IDEA-{NNN} ({score}) — {title}
2. IDEA-{NNN} ({score}) — {title}
3. IDEA-{NNN} ({score}) — {title}

Stale ideas (30+ days untouched): {N}

Quick actions:
- Want to start one of the ready ideas?
- Any ideas to kill or reprioritize?
- New context that changes any scores?
```

## Monthly Deep Review (15 minutes)

```
Monthly idea review time! Let's do a thorough sweep.

### Stats This Month
- Ideas added: {N}
- Ideas shipped: {N}
- Ideas killed: {N}
- Average time from idea to ship: {N} days
- Current backlog size: {N}

### Ready to Build — Confirm Priority Order
{List all Ready ideas with scores. Ask user to confirm or reorder.}

### Backlog — Any Promotions?
{List all Backlog ideas. Ask if any should be promoted based on new context.}

### Ice Box — Still Relevant?
{List all Ice Box ideas. Quick yes/no on each.}

### Stale Ideas (30+ days)
{List stale ideas. Force a decision: keep, reprioritize, or kill.}

### Dependencies Check
{List ideas that depend on recently shipped features.
These might be unblocked now.}

### New Context
- Any new user feedback that changes priorities?
- Any new tech or tools that make something easier?
- Any competitor moves that change urgency?
```

## After Shipping a Feature

```
{Feature name} is shipped! Nice work.

This unblocks these parked ideas:
- IDEA-{NNN} ({score}) — {title} — was waiting on {dependency}
- IDEA-{NNN} ({score}) — {title} — was waiting on {dependency}

You also have {N} other ready ideas.

Want to:
1. Jump into one of the unblocked ideas?
2. Review the full backlog?
3. Take a break and come back later?
```

## When User Seems Unfocused

```
I've noticed you've started {N} new things in the last {timeframe}
without finishing any of them. That's totally normal — creative brain!

Here's what's open right now:
1. {Task} — {progress}%
2. {Task} — {progress}%
3. {Task} — {progress}%

The fastest path to a dopamine hit is finishing the one closest to done.
That's #{N} at {progress}%. Want to knock it out?
```

## When No Ideas Are Ready

```
Your idea board is looking clean:
- {N} ideas in backlog
- {N} in ice box
- Nothing scored "Ready to Build" right now

Options:
1. Reprioritize backlog — maybe context has changed and something scores higher now
2. Build something new — free to explore without backlog guilt
3. Improve what's shipped — polish existing features based on user feedback
```

## First Time Setup

```
I'm setting up your Idea Parking Lot! This is where every idea gets
captured, scored, and organized so nothing gets lost and nothing
derails your current work.

I've created IDEAS.md in your project root.

How it works:
- When you have an idea, I'll capture it with a quick score
- Ideas get organized by priority (Ready / Backlog / Ice Box)
- I'll remind you to review weekly
- When you finish a task, I'll surface the best next idea

The scoring uses ICE: Impact + Confidence + Ease (each 1-10).
Score of 24+ = Ready to Build. Under 15 = Ice Box.

You have zero ideas parked right now. Let's build something!
```
