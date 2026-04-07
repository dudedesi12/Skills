# Commit Message Conventions

How to write commit messages that actually help you understand what you did when you look back in a week.

---

## The Format

```
type(scope): short description
```

That is it. Three parts:

- **type** = what kind of change (feat, fix, chore, docs, style, refactor)
- **scope** = what part of the app you changed (auth, dashboard, api, db, landing)
- **description** = what you did, in plain English, lowercase, no period at the end

## The Six Types

### feat — You added something new

A new feature, page, component, or capability that users can see or interact with.

```
feat(auth): add Google OAuth login button
feat(dashboard): add revenue chart with date range filter
feat(api): add endpoint for exporting user data as CSV
feat(landing): add pricing comparison table
feat(chat): add real-time message notifications
feat(profile): add avatar upload with drag and drop
feat(search): add full-text search across all posts
feat(billing): add annual subscription option
feat(onboarding): add 3-step welcome wizard for new users
feat(api): add rate limiting to public endpoints
```

### fix — You fixed something broken

A bug that was causing incorrect behavior, crashes, or errors.

```
fix(auth): prevent infinite redirect loop on expired session
fix(dashboard): show correct timezone for event timestamps
fix(api): return 404 instead of 500 for missing resources
fix(checkout): calculate tax correctly for EU customers
fix(upload): handle files over 5MB without crashing
fix(nav): close mobile menu when route changes
fix(form): validate email format before submission
fix(db): fix N+1 query in user list endpoint
fix(seo): add missing og:image meta tags on blog posts
fix(auth): handle null user in session middleware
```

### chore — Maintenance and housekeeping

Dependencies, configuration, build setup, tooling. Not a user-facing change.

```
chore(deps): upgrade next to 15.1.0
chore(deps): update tailwindcss to v4
chore(config): add prettier config for consistent formatting
chore(ci): add GitHub Actions workflow for deploy preview
chore(deps): remove unused lodash dependency
chore(config): update tsconfig strict mode settings
chore(env): add STRIPE_WEBHOOK_SECRET to env example
chore(deps): run npm audit fix
chore(config): add path aliases for clean imports
chore(git): update gitignore to exclude Supabase temp files
```

### docs — Documentation changes

README updates, code comments, JSDoc, inline documentation.

```
docs(readme): add local development setup instructions
docs(api): add JSDoc comments to all route handlers
docs(readme): add deploy to Vercel button
docs(contributing): add pull request template
docs(env): document all required environment variables
docs(api): add OpenAPI spec for public endpoints
docs(readme): add architecture diagram
docs(changelog): add v2.1 release notes
```

### style — Visual/UI changes only

CSS, Tailwind classes, colors, fonts, spacing. No logic changes.

```
style(landing): increase hero section font size to 4xl
style(dashboard): switch card backgrounds to slate-50
style(global): update brand colors to new palette
style(nav): add hover animation to menu items
style(footer): reduce padding on mobile viewports
style(buttons): make primary button corners more rounded
style(landing): add gradient background to CTA section
style(dashboard): align chart labels to left
style(form): increase input field height to 48px
style(typography): switch body font to Inter
```

### refactor — Code restructure, same behavior

Reorganizing code, renaming things, extracting functions. The app works exactly the same before and after.

```
refactor(auth): extract session logic into useSession hook
refactor(api): move validation into shared middleware
refactor(db): replace raw SQL with Supabase query builder
refactor(components): split Dashboard into smaller components
refactor(utils): consolidate date formatting functions
refactor(api): use consistent error response format
refactor(types): generate database types from Supabase schema
refactor(auth): replace context provider with Zustand store
refactor(routes): reorganize route groups by feature
refactor(api): extract common CRUD operations into base handler
```

---

## Picking the Right Scope

The scope tells you WHERE in the app the change happened. Use the folder or feature name.

| Files changed in... | Scope |
|---------------------|-------|
| `app/auth/`, login/signup pages | `auth` |
| `app/dashboard/`, dashboard components | `dashboard` |
| `app/api/`, route handlers | `api` |
| `app/(landing)/`, marketing pages | `landing` |
| `components/ui/` | `ui` |
| `lib/`, utility functions | `utils` |
| `supabase/`, database migrations | `db` |
| `middleware.ts` | `middleware` |
| `package.json`, config files | `deps` or `config` |
| Multiple areas | pick the primary one |

---

## How to Auto-Generate a Commit Message

Look at what changed and follow this decision process:

```bash
git diff --stat
```

### Step 1: What type?

- Added a new file or feature? -> `feat`
- Fixed something broken? -> `fix`
- Only changed CSS/Tailwind classes? -> `style`
- Moved code around, renamed things? -> `refactor`
- Updated packages or config? -> `chore`
- Updated comments or docs? -> `docs`

### Step 2: What scope?

Look at the file paths:
- `src/app/auth/login/page.tsx` -> scope is `auth`
- `src/app/api/users/route.ts` -> scope is `api`
- `src/components/dashboard/Chart.tsx` -> scope is `dashboard`

### Step 3: What description?

Describe WHAT you did, not HOW. Use present tense, lowercase, no period.

- Good: `add login page with email validation`
- Bad: `Added the login page. Used Zod for validation.`
- Bad: `updated stuff`

---

## Rejected Commit Messages

Never accept these messages. Always rewrite them:

| Bad message | Why it is bad | Better version |
|------------|--------------|----------------|
| `update` | Update what? | `feat(dashboard): add date range filter to chart` |
| `fix` | Fix what? | `fix(auth): prevent redirect loop on expired token` |
| `changes` | What changes? | `refactor(api): extract validation into middleware` |
| `stuff` | Meaningless | `chore(deps): update next and tailwind` |
| `wip` | Not a save point | `feat(checkout): add cart total calculation (partial)` |
| `asdf` | Keyboard smash | `style(landing): adjust hero section spacing` |
| `test` | Test what? | `feat(auth): add login form with email validation` |
| `minor changes` | Not descriptive | `fix(nav): correct active link highlight on mobile` |
| `fixed bug` | Which bug? | `fix(upload): handle empty file selection gracefully` |
| `updated files` | Which files, why? | `refactor(utils): consolidate date helpers into one file` |
| `final version` | There is no final | `feat(landing): complete pricing page with all tiers` |
| `please work` | Mood is not a message | `fix(deploy): add missing env var to build config` |

---

## Multiple Changes in One Commit

If you changed multiple things, use the primary change as the message:

```bash
# Changed auth login page AND fixed a typo in the nav
# The auth change is the primary work
git commit -m "feat(auth): add login page with email validation"
```

If the changes are truly unrelated, make separate commits:

```bash
git add src/app/auth/
git commit -m "feat(auth): add login page with email validation"

git add src/components/Nav.tsx
git commit -m "fix(nav): correct spelling of 'Dashboard' in menu"
```

---

## Commit Message Length

- **Type + scope + description** should fit in 72 characters or fewer.
- If you need more detail, add a blank line and then a body:

```
feat(billing): add Stripe webhook handler for subscription events

Handles subscription.created, subscription.updated, and
subscription.deleted events. Updates user tier in Supabase
and sends confirmation email via Resend.
```

The first line is what shows up in `git log --oneline`. The body is for extra context when someone clicks into the commit.

---

## Quick Reference Card

```
feat(scope): add [new thing]           <- new feature
fix(scope): prevent/handle/correct     <- bug fix
chore(scope): update/upgrade/remove    <- maintenance
docs(scope): add/update                <- documentation
style(scope): adjust/change/update     <- visual only
refactor(scope): extract/move/rename   <- code reorganization
```

Scopes: auth, dashboard, api, landing, ui, utils, db, middleware, deps, config, nav, billing, chat, profile, search, onboarding, seo, upload, form, checkout
