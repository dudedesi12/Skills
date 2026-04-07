# IDEAS.md Template

Copy this template into the root of every project repo.

---

```markdown
# Idea Parking Lot

> Auto-managed by idea-parking-lot skill. Do not edit manually.
> Last updated: {date}

## Ready to Build (ICE Score >= 24)

<!-- Ideas scored 24+ go here. These are green-lit and waiting for capacity. -->

## In Progress

<!-- The idea currently being worked on. Only ONE at a time. -->

## Backlog (ICE Score 15-23)

<!-- Good ideas that aren't urgent. Review weekly. -->

## Ice Box (ICE Score < 15)

<!-- Cool but not worth the effort right now. Review monthly. -->

## Shipped

<!-- Completed ideas with ship date and PR/commit link. -->

## Killed (with reason)

<!-- Archived ideas. Always include the reason for killing. -->
```

---

## Idea Entry Format

Every idea entry follows this exact format:

```markdown
### [IDEA-{NNN}] {Short descriptive title}
- **Added:** {YYYY-MM-DD}
- **Problem:** {One sentence — what pain point does this solve?}
- **Who benefits:** {Which users or what part of the business}
- **ICE Score:** I:{n} C:{n} E:{n} = **{total}**
- **Dependencies:** {Other ideas or skills needed, or "None"}
- **Estimated effort:** {Quick win / 1 session / 2-3 sessions / Multi-week}
- **Status:** {Ready / In Progress / Backlog / Ice Box / Shipped / Killed}
- **Tags:** {Relevant skills like supabase-power-user, stripe-saas}
- **Notes:** {Any additional context, constraints, or links}
```

## Shipped Entry Format

When an idea ships, update it:

```markdown
### [IDEA-{NNN}] {Short descriptive title}
- **Added:** {YYYY-MM-DD}
- **Shipped:** {YYYY-MM-DD}
- **PR:** #{PR number} or commit {hash}
- **Status:** Shipped
- **Notes:** {What was actually built, any deviations from original idea}
```

## Killed Entry Format

When an idea is killed, archive it:

```markdown
### [IDEA-{NNN}] {Short descriptive title}
- **Added:** {YYYY-MM-DD}
- **Killed:** {YYYY-MM-DD}
- **Reason:** {Why — duplicate, out of scope, tech not ready, user need changed}
- **Status:** Killed
```

## ID Assignment

- IDs are sequential: IDEA-001, IDEA-002, IDEA-003...
- Never reuse an ID, even for killed ideas
- Check the highest existing ID before assigning a new one
