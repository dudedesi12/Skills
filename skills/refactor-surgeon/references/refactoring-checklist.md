# Refactoring Checklist

Use this checklist before, during, and after every refactoring session.

## Pre-Refactoring (Do These First)

- [ ] **Commit current work** — Create a clean commit point so you can revert if needed. `git add -A && git commit -m "checkpoint before refactoring"`
- [ ] **Identify the target file(s)** — List every file you plan to touch with line counts.
- [ ] **Map all imports** — Search the codebase for every file that imports from your target. `grep -r "from.*target-file" --include="*.ts" --include="*.tsx"`
- [ ] **Map all exports** — List every export from the target file. `grep "^export" path/to/file.ts`
- [ ] **Run the build** — Confirm the project builds successfully before you start. `npm run build`
- [ ] **Run tests** — If tests exist, confirm they pass. `npm test`
- [ ] **Define the goal** — Write down what the file structure should look like after refactoring. Don't start until this is clear.
- [ ] **Check for `any` types** — Count them so you can track how many you fix. `grep -c ": any" path/to/file.ts`

## During Refactoring (Follow This Order)

### For Each File Split

- [ ] **One file at a time** — Never split two files in parallel.
- [ ] **Create the new file** — Write the extracted code in its new location.
- [ ] **Add re-exports** — In the old file, add `export { ... } from "./new-file"` for backward compatibility.
- [ ] **Build** — Run `npm run build` to confirm nothing broke with the re-exports.
- [ ] **Update consumers** — Change imports in every file that uses the moved code.
- [ ] **Build again** — Run `npm run build` after updating consumers.
- [ ] **Remove re-exports** — Delete the temporary re-exports from the old file.
- [ ] **Final build** — Run `npm run build` one more time.
- [ ] **Commit** — Create a commit for this single file split.

### For `any` Type Fixes

- [ ] **One `any` at a time** — Fix, build, move on.
- [ ] **Check usage** — Before defining a type, look at how the value is actually used to determine the correct type.
- [ ] **Prefer specific types over `unknown`** — `unknown` is better than `any`, but a specific interface is better than both.
- [ ] **Use Supabase generated types** — For database results, use `Database["public"]["Tables"]["table_name"]["Row"]`.
- [ ] **Use React event types** — `React.ChangeEvent<HTMLInputElement>`, `React.FormEvent<HTMLFormElement>`, etc.
- [ ] **Use `unknown` for catch blocks** — `catch (error: unknown)` then narrow with `error instanceof Error`.

### For Hook Extractions

- [ ] **Identify the state cluster** — Group related useState/useEffect calls.
- [ ] **Define the return type** — Write the interface before writing the hook.
- [ ] **Move state and effects** — Copy to the new hook file, import dependencies.
- [ ] **Return only what the component needs** — Don't expose internal state the component doesn't use.
- [ ] **Update the component** — Replace inline state with the hook call.
- [ ] **Build and test** — Confirm behavior is identical.

### For Component Extractions

- [ ] **Define props interface first** — Know what data flows in and out.
- [ ] **Keep props under 7** — If you need more, reconsider the boundary or group into objects.
- [ ] **Handle loading and empty states** — Each extracted component should handle its own loading/empty/error states.
- [ ] **Preserve the `"use client"` directive** — If the parent was a client component, the child needs it too (unless it's a pure display component).
- [ ] **Move related styles** — If using CSS modules, move the styles with the component.

## Post-Refactoring (Verify Everything)

### Build Verification

- [ ] **Clean build passes** — `rm -rf .next && npm run build`
- [ ] **No TypeScript errors** — Build output shows zero type errors.
- [ ] **No new warnings** — Compare build warnings before and after.

### Import Verification

- [ ] **No circular imports** — Check for import cycles. A imports B which imports A is a problem.
- [ ] **No orphaned re-exports** — Search for any temporary re-exports you forgot to remove.
- [ ] **No deep imports** — Consumers shouldn't import from `@/components/feature/sub/deep/file`. Use barrel exports.
- [ ] **All paths use `@/` alias** — No relative imports that go up 3+ levels (`../../../`).

### Code Quality

- [ ] **`any` count reduced** — Compare before and after: `grep -rc ": any" --include="*.ts" --include="*.tsx" src/ app/ lib/`
- [ ] **No file over 300 lines** — Check all new files: `wc -l path/to/new/*.ts`
- [ ] **Each file has one responsibility** — Read each new file and confirm it does one thing.
- [ ] **No dead code** — Remove any code that was copied but not needed in the new location.
- [ ] **No duplicate code** — Ensure extracted code doesn't still exist in the original file.

### Behavioral Verification

- [ ] **Run the dev server** — `npm run dev` and click through affected pages.
- [ ] **Check the browser console** — No new errors or warnings.
- [ ] **Run E2E tests** — If you have Playwright tests, run them: `npx playwright test`
- [ ] **Test edge cases** — Empty states, error states, loading states all still work.

### Git Hygiene

- [ ] **Small commits** — One commit per logical change (one file split = one commit).
- [ ] **Descriptive messages** — Example: `refactor: extract formatting utils from lib/utils.ts`
- [ ] **No unrelated changes** — Don't sneak in feature changes or bug fixes during a refactoring PR.
- [ ] **Clean diff** — Review `git diff` to ensure only intended changes are included.

## Quick Reference: Commit Message Formats

```
refactor: extract UserTable component from settings-page
refactor: split lib/utils.ts into formatting, validation, calculations
refactor: extract useUsers hook from users-page component
refactor: move types to dedicated type files
refactor: replace 12 'any' types with proper interfaces
refactor: create barrel exports for components/dashboard
refactor: remove dead code from lib/api-helpers
refactor: extract auth middleware wrapper (withAuth)
```

## Red Flags — Stop and Reconsider

- **More than 10 files changed** — Your refactoring might be too large. Break it into smaller PRs.
- **Tests are failing** — Don't push through. Fix the test or revert and try a different split.
- **You're changing behavior** — Refactoring should not change what the code does, only how it's organized.
- **You need to add new dependencies** — Refactoring shouldn't require new packages.
- **The new structure is more confusing** — If teammates would struggle to find things, the split isn't right. Sometimes a 400-line file is better than 8 files with 50 lines each.
- **Circular dependencies appear** — You split at the wrong boundary. Module A and B both depend on each other = wrong split.

## File Size Guidelines

| File Type | Target Size | Max Before Splitting |
|-----------|-------------|----------------------|
| Component | 50-200 lines | 300 lines |
| Custom Hook | 30-100 lines | 150 lines |
| Utility/Helper | 30-150 lines | 250 lines |
| API Route | 30-80 lines | 150 lines |
| Type Definitions | 20-100 lines | 200 lines |
| Zod Schemas | 20-80 lines | 150 lines |
| Service/Query | 50-150 lines | 250 lines |
| Constants | 20-80 lines | 150 lines |
| Test File | 50-200 lines | 400 lines |
