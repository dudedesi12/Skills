---
name: accessibility-champion
description: "Use this skill whenever the user mentions accessibility, a11y, WCAG, screen reader, VoiceOver, NVDA, ARIA, aria-label, aria-live, keyboard navigation, focus management, tab order, skip link, color contrast, alt text, semantic HTML, landmarks, heading hierarchy, focus trap, focus ring, reduced motion, prefers-reduced-motion, axe-core, accessibility audit, 'can blind users use this', 'keyboard only', 'is my app accessible', or ANY accessibility/inclusion task — even if they don't explicitly say 'accessibility'. An immigration platform serves diverse users — accessibility is not optional."
---

# Accessibility Champion

You are building for people from every background, ability level, and device. Screen reader users, keyboard-only users, users with low vision, users on slow connections — they all need to use your immigration platform. WCAG 2.2 Level AA is the target.

## Stack Context

- Next.js App Router (TypeScript)
- Tailwind CSS (with shadcn/ui)
- Supabase (auth, database)
- Vercel (deployment)

## 1. WCAG 2.2 Quick Reference

WCAG has three levels: A (minimum), AA (target), AAA (ideal). **Target Level AA for everything.**

| Principle | What It Means | Key Checks |
|-----------|--------------|------------|
| **Perceivable** | Users can see/hear content | Alt text, color contrast, captions, text resize |
| **Operable** | Users can interact with UI | Keyboard nav, focus visible, no time traps, skip links |
| **Understandable** | Users can understand content | Clear language, predictable UI, error help |
| **Robust** | Works with assistive tech | Valid HTML, ARIA roles, semantic elements |

**The big numbers to remember:**
- **4.5:1** — Minimum contrast ratio for normal text
- **3:1** — Minimum contrast ratio for large text (18px+ bold or 24px+)
- **44x44px** — Minimum touch target size
- **2.5 seconds** — Maximum time before a user can dismiss a timeout warning

## 2. Semantic HTML

Use the right HTML element for the job. Assistive tech reads the element type, not the CSS.

```tsx
// BAD — divs with click handlers
<div onClick={handleClick} className="button-style">Submit</div>
<div className="header">Welcome</div>
<div className="nav-item">Dashboard</div>

// GOOD — semantic elements
<button onClick={handleClick}>Submit</button>
<h1>Welcome</h1>
<nav aria-label="Main navigation">
  <a href="/dashboard">Dashboard</a>
</nav>
```

### Landmarks

Screen readers let users jump between landmarks. Use these:

```tsx
<header>        {/* Site header — logo, nav */}
<nav>           {/* Navigation links */}
<main>          {/* Primary content — ONE per page */}
<aside>         {/* Sidebar, related content */}
<footer>        {/* Site footer */}
<section>       {/* Thematic grouping with a heading */}
<article>       {/* Self-contained content */}
```

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <a href="#main-content" className="sr-only focus:not-sr-only focus:absolute focus:z-50 focus:bg-white focus:p-4">
          Skip to main content
        </a>
        <header>
          <nav aria-label="Main navigation">{/* nav links */}</nav>
        </header>
        <main id="main-content">{children}</main>
        <footer>{/* footer content */}</footer>
      </body>
    </html>
  );
}
```

### Heading Hierarchy

Headings must follow a logical order. Never skip levels.

```tsx
// BAD
<h1>Dashboard</h1>
<h3>Recent Activity</h3>  {/* Skipped h2! */}
<h5>Details</h5>           {/* Skipped h3, h4! */}

// GOOD
<h1>Dashboard</h1>
<h2>Recent Activity</h2>
<h3>Details</h3>
```

## 3. ARIA Roles and Properties

**Rule 1: Don't use ARIA if a native HTML element does the job.**

```tsx
// BAD — unnecessary ARIA
<button role="button">Click me</button>
<a role="link" href="/page">Link</a>

// GOOD — native elements already have the right role
<button>Click me</button>
<a href="/page">Link</a>
```

**When you DO need ARIA:**

```tsx
// Loading state — announce to screen readers
<div role="status" aria-live="polite">
  {isLoading ? "Loading results..." : `${results.length} results found`}
</div>

// Icon-only button — needs a label
<button aria-label="Close dialog" onClick={onClose}>
  <XIcon className="h-5 w-5" />
</button>

// Custom component — needs role and states
<div
  role="tablist"
  aria-label="Assessment sections"
>
  <button role="tab" aria-selected={activeTab === "profile"} aria-controls="panel-profile">
    Profile
  </button>
  <button role="tab" aria-selected={activeTab === "results"} aria-controls="panel-results">
    Results
  </button>
</div>
<div role="tabpanel" id="panel-profile" aria-labelledby="tab-profile">
  {/* Tab content */}
</div>
```

See `references/aria-patterns.md` for complete ARIA patterns for modals, tabs, accordions, and comboboxes.

## 4. Keyboard Navigation

Every interactive element must be reachable and usable with keyboard only.

### Focus Order

```tsx
// Tab order follows DOM order. Use tabIndex carefully:
<button>First</button>     {/* tabIndex=0 (default) */}
<button>Second</button>    {/* tabIndex=0 (default) */}
<button>Third</button>     {/* tabIndex=0 (default) */}

// tabIndex values:
// 0   = In normal tab order (default for interactive elements)
// -1  = Focusable by script only (useful for modals, skip targets)
// 1+  = NEVER USE — breaks natural tab order
```

### Skip Link

Always add a skip link as the first focusable element:

```tsx
// components/skip-link.tsx
export function SkipLink() {
  return (
    <a
      href="#main-content"
      className="fixed left-0 top-0 z-[100] -translate-y-full bg-blue-600 px-4 py-2 text-white transition-transform focus:translate-y-0"
    >
      Skip to main content
    </a>
  );
}
```

### Focus Trap for Modals

When a modal is open, keyboard focus must stay inside it:

```tsx
// hooks/use-focus-trap.ts
"use client";
import { useEffect, useRef } from "react";

export function useFocusTrap(isOpen: boolean) {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isOpen || !ref.current) return;

    const element = ref.current;
    const focusableElements = element.querySelectorAll<HTMLElement>(
      'a[href], button:not([disabled]), input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'
    );
    const firstFocusable = focusableElements[0];
    const lastFocusable = focusableElements[focusableElements.length - 1];

    firstFocusable?.focus();

    function handleKeyDown(e: KeyboardEvent) {
      if (e.key !== "Tab") return;
      if (e.shiftKey) {
        if (document.activeElement === firstFocusable) {
          e.preventDefault();
          lastFocusable?.focus();
        }
      } else {
        if (document.activeElement === lastFocusable) {
          e.preventDefault();
          firstFocusable?.focus();
        }
      }
    }

    element.addEventListener("keydown", handleKeyDown);
    return () => element.removeEventListener("keydown", handleKeyDown);
  }, [isOpen]);

  return ref;
}
```

### Escape Key for Dismissible Elements

```tsx
useEffect(() => {
  function handleEscape(e: KeyboardEvent) {
    if (e.key === "Escape" && isOpen) onClose();
  }
  document.addEventListener("keydown", handleEscape);
  return () => document.removeEventListener("keydown", handleEscape);
}, [isOpen, onClose]);
```

## 5. Color Contrast and Visual Design

### Contrast Ratios

```
Normal text (< 18px bold or < 24px): 4.5:1 minimum
Large text (≥ 18px bold or ≥ 24px):  3:1 minimum
UI components and graphical objects:  3:1 minimum
```

### Tailwind Colors That Pass

```tsx
// GOOD — high contrast on white background
<p className="text-gray-900">Primary text</p>     {/* ~15:1 */}
<p className="text-gray-700">Secondary text</p>   {/* ~10:1 */}
<p className="text-gray-600">Tertiary text</p>    {/* ~7:1  */}

// BAD — insufficient contrast on white
<p className="text-gray-400">Hard to read</p>     {/* ~3:1 — FAILS */}
<p className="text-gray-300">Very hard</p>         {/* ~2:1 — FAILS */}
```

### Don't Rely on Color Alone

```tsx
// BAD — only color indicates error
<input className={error ? "border-red-500" : "border-gray-300"} />

// GOOD — color + icon + text
<div>
  <input
    className={error ? "border-red-500" : "border-gray-300"}
    aria-invalid={!!error}
    aria-describedby={error ? "email-error" : undefined}
  />
  {error && (
    <p id="email-error" className="mt-1 flex items-center gap-1 text-sm text-red-600">
      <AlertCircle className="h-4 w-4" />
      {error}
    </p>
  )}
</div>
```

### Reduced Motion

```tsx
// Respect user's motion preference
<div className="transition-transform motion-reduce:transition-none">
  {/* animated content */}
</div>
```

```css
/* In global CSS */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

## 6. Screen Reader Testing

### Quick VoiceOver Test (Mac)

1. Press **Cmd + F5** to turn on VoiceOver
2. Use **Tab** to move through interactive elements
3. Use **VO + Right Arrow** (Ctrl+Option+Right) to read through content
4. Listen for: proper headings, link text, button labels, form labels
5. Press **Cmd + F5** again to turn off

### What to Check

- [ ] Page title is announced when navigating
- [ ] Headings are in logical order (use VO + U to see heading list)
- [ ] Images have descriptive alt text (or empty alt for decorative)
- [ ] Form fields are labeled (click the label, cursor should move to input)
- [ ] Buttons and links describe their purpose
- [ ] Error messages are announced
- [ ] Loading states are announced
- [ ] Modal focus is trapped inside

## 7. Form Accessibility

Forms are where accessibility matters most on an immigration platform.

```tsx
// components/accessible-form-field.tsx
interface FormFieldProps {
  id: string;
  label: string;
  type?: string;
  required?: boolean;
  error?: string;
  helpText?: string;
}

export function FormField({ id, label, type = "text", required, error, helpText }: FormFieldProps) {
  const describedBy = [
    helpText ? `${id}-help` : null,
    error ? `${id}-error` : null,
  ].filter(Boolean).join(" ") || undefined;

  return (
    <div>
      <label htmlFor={id} className="block text-sm font-medium text-gray-700">
        {label}
        {required && <span className="text-red-500 ml-1" aria-hidden="true">*</span>}
        {required && <span className="sr-only">(required)</span>}
      </label>
      <input
        id={id}
        type={type}
        required={required}
        aria-invalid={!!error}
        aria-describedby={describedBy}
        className={`mt-1 w-full rounded border px-3 py-2 ${
          error ? "border-red-500 focus:ring-red-500" : "border-gray-300 focus:ring-blue-500"
        }`}
      />
      {helpText && (
        <p id={`${id}-help`} className="mt-1 text-sm text-gray-500">{helpText}</p>
      )}
      {error && (
        <p id={`${id}-error`} role="alert" className="mt-1 text-sm text-red-600">{error}</p>
      )}
    </div>
  );
}
```

### Error Summary

After form submission, show an error summary that links to each field:

```tsx
// components/error-summary.tsx
interface ErrorSummaryProps {
  errors: Record<string, string>;
}

export function ErrorSummary({ errors }: ErrorSummaryProps) {
  const entries = Object.entries(errors);
  if (entries.length === 0) return null;

  return (
    <div role="alert" className="rounded border border-red-300 bg-red-50 p-4">
      <h2 className="font-semibold text-red-800">
        There {entries.length === 1 ? "is 1 error" : `are ${entries.length} errors`} in your form
      </h2>
      <ul className="mt-2 list-disc pl-5 text-sm text-red-700">
        {entries.map(([field, message]) => (
          <li key={field}>
            <a href={`#${field}`} className="underline">{message}</a>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## 8. Dynamic Content

### Live Regions

Announce changes to screen readers without moving focus:

```tsx
// Announce search results count
<div aria-live="polite" aria-atomic="true" className="sr-only">
  {isSearching ? "Searching..." : `${results.length} results found`}
</div>

// Announce form submission success
<div role="status" aria-live="polite">
  {submitted && "Your assessment has been submitted successfully."}
</div>

// Announce urgent errors
<div role="alert" aria-live="assertive">
  {error && `Error: ${error}`}
</div>
```

### Loading Announcements

```tsx
// components/loading-announcement.tsx
export function LoadingAnnouncement({ isLoading, message = "Loading content" }: {
  isLoading: boolean;
  message?: string;
}) {
  return (
    <>
      {isLoading && (
        <div role="status" aria-live="polite" className="sr-only">
          {message}
        </div>
      )}
      {isLoading && (
        <div aria-hidden="true" className="flex items-center justify-center py-8">
          <div className="h-8 w-8 animate-spin rounded-full border-4 border-blue-500 border-t-transparent" />
        </div>
      )}
    </>
  );
}
```

## 9. Automated Testing with axe-core

### Playwright + axe-core for CI

```bash
npm install -D @axe-core/playwright
```

```typescript
// tests/e2e/accessibility/a11y.spec.ts
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test.describe("Accessibility", () => {
  const pages = ["/", "/login", "/signup", "/dashboard", "/assessment"];

  for (const path of pages) {
    test(`${path} has no accessibility violations`, async ({ page }) => {
      await page.goto(path);
      await page.waitForLoadState("networkidle");

      const results = await new AxeBuilder({ page })
        .withTags(["wcag2a", "wcag2aa", "wcag22aa"])
        .analyze();

      expect(results.violations).toEqual([]);
    });
  }
});
```

See `references/axe-core-setup.md` for full CI integration and custom rules.

## 10. Images and Media

```tsx
// Informative image — describe what it shows
<img src="/visa-process.png" alt="Four-step visa application process: eligibility check, document upload, assessment, and decision" />

// Decorative image — empty alt
<img src="/decorative-pattern.svg" alt="" />

// Complex image — use aria-describedby
<figure>
  <img src="/points-chart.png" alt="Points calculator results" aria-describedby="chart-desc" />
  <figcaption id="chart-desc">
    Bar chart showing points breakdown: Age 30, English 20, Experience 15, Qualification 15. Total: 80 points.
  </figcaption>
</figure>

// Icon with text — hide the icon from screen readers
<button>
  <CheckIcon aria-hidden="true" className="h-5 w-5" />
  Save Changes
</button>

// Icon without text — needs aria-label
<button aria-label="Delete assessment">
  <TrashIcon aria-hidden="true" className="h-5 w-5" />
</button>
```

## 11. Rules

1. **Use semantic HTML first** — div and span are last resorts for interactive elements.
2. **Every image needs alt text** — Descriptive for informative images, empty string for decorative.
3. **Every form field needs a label** — Connected via `htmlFor`/`id`, not just placeholder text.
4. **Test with keyboard** — Tab through every page. If you can't reach something, it's broken.
5. **Don't skip heading levels** — h1 → h2 → h3, never h1 → h3.
6. **Color is never the only indicator** — Always pair color with text, icons, or patterns.
7. **Announce dynamic changes** — Use `aria-live` for content that updates without page reload.
8. **Trap focus in modals** — When a modal opens, focus goes in. Tab stays inside. Escape closes it.
9. **Test with a screen reader** — Run VoiceOver (Mac) or NVDA (Windows) on your key pages at least once.
10. **Run axe-core in CI** — Automated checks catch 30-50% of issues. Manual testing catches the rest.

See `references/` for detailed patterns and checklists:
- `axe-core-setup.md` — Playwright + axe-core CI integration
- `aria-patterns.md` — Complete ARIA patterns for modals, tabs, accordions, comboboxes
- `audit-checklist.md` — 40+ item WCAG AA compliance checklist
