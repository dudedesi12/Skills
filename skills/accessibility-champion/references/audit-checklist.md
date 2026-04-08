# Accessibility Audit Checklist — WCAG 2.2 Level AA

Go through every item before launching. Test each page in your app.

## Perceivable (Can Users See/Hear Content?)

### Text Alternatives (1.1)
- [ ] All informative images have descriptive `alt` text
- [ ] Decorative images have `alt=""`
- [ ] Complex images (charts, diagrams) have detailed descriptions
- [ ] Icon-only buttons have `aria-label`
- [ ] SVG icons used for decoration have `aria-hidden="true"`
- [ ] CAPTCHA has an alternative (audio or text-based)

### Time-Based Media (1.2)
- [ ] Videos have captions
- [ ] Audio content has transcripts
- [ ] Live video has real-time captions (if applicable)

### Adaptable (1.3)
- [ ] Headings follow logical hierarchy (h1 → h2 → h3, no skips)
- [ ] Lists use `<ul>`, `<ol>`, `<dl>` — not styled divs
- [ ] Tables have `<th>` headers with `scope` attributes
- [ ] Form fields are linked to labels via `htmlFor`/`id`
- [ ] Page structure uses landmarks: `<header>`, `<nav>`, `<main>`, `<footer>`
- [ ] Reading order makes sense when CSS is disabled
- [ ] Input purpose is identified (autocomplete attributes)

### Distinguishable (1.4)
- [ ] Normal text has 4.5:1 contrast ratio against background
- [ ] Large text (18px+ bold or 24px+) has 3:1 contrast ratio
- [ ] UI components (borders, icons) have 3:1 contrast ratio
- [ ] Color is never the only way to convey information
- [ ] Text can be resized to 200% without loss of content
- [ ] No horizontal scrolling at 320px viewport width (reflow)
- [ ] Custom focus indicators have 3:1 contrast against adjacent colors
- [ ] Content doesn't require a specific orientation (landscape/portrait)
- [ ] Non-text contrast meets 3:1 for UI components

## Operable (Can Users Interact with UI?)

### Keyboard Accessible (2.1)
- [ ] All interactive elements are reachable with Tab key
- [ ] All interactive elements are activatable with Enter or Space
- [ ] Focus order follows visual layout (left-to-right, top-to-bottom)
- [ ] No keyboard traps (focus can always escape)
- [ ] Custom keyboard shortcuts use modifier keys (Ctrl, Alt, Shift)
- [ ] Focus is visible on all interactive elements

### Enough Time (2.2)
- [ ] Session timeouts can be extended or warned about
- [ ] Auto-updating content can be paused or stopped
- [ ] No content flashes more than 3 times per second

### Navigable (2.3)
- [ ] Skip-to-content link is the first focusable element
- [ ] Page titles are descriptive and unique per page
- [ ] Link text describes the destination (no "click here")
- [ ] Multiple ways to find pages (nav, search, sitemap)
- [ ] Headings and labels are descriptive
- [ ] Focus is visible at all times

### Input Modalities (2.5)
- [ ] Touch targets are at least 44x44 pixels
- [ ] Click/touch actions can be cancelled (mouseup/touchend, not mousedown)
- [ ] Visible labels match accessible names (`aria-label` ≈ visible text)
- [ ] Motion-based interactions have alternatives (shake to undo → button)

## Understandable (Can Users Understand Content?)

### Readable (3.1)
- [ ] Page language is declared (`<html lang="en">`)
- [ ] Language changes within content are marked (`<span lang="zh">`)
- [ ] Abbreviations are expanded on first use
- [ ] Technical terms are defined or linked to glossary

### Predictable (3.2)
- [ ] Focus changes don't cause unexpected context changes
- [ ] Input changes don't auto-submit forms (unless warned)
- [ ] Navigation is consistent across pages
- [ ] Components are identified consistently (same icon = same action)

### Input Assistance (3.3)
- [ ] Error messages identify which field has the error
- [ ] Error messages suggest how to fix the problem
- [ ] Required fields are marked before submission
- [ ] Form instructions are visible before the form
- [ ] Users can review and confirm before final submission
- [ ] Errors are linked to their fields (aria-describedby)

## Robust (Does It Work with Assistive Tech?)

### Compatible (4.1)
- [ ] HTML validates (no duplicate IDs, proper nesting)
- [ ] All interactive elements have accessible names
- [ ] Status messages use `role="status"` or `aria-live`
- [ ] Custom components have correct ARIA roles and states
- [ ] Dynamic content updates are announced to screen readers

## Page-Specific Checks

### Login/Signup Pages
- [ ] Form fields have labels (not just placeholders)
- [ ] Password field has show/hide toggle with accessible label
- [ ] Error messages appear next to the relevant field
- [ ] Success state is announced ("Account created" via aria-live)
- [ ] Social login buttons have descriptive labels ("Sign in with Google")

### Dashboard
- [ ] Data tables have proper headers
- [ ] Charts have text alternatives
- [ ] Loading states are announced
- [ ] Empty states are communicated to screen readers
- [ ] Filters and sorting controls are keyboard accessible

### Assessment/Form Pages
- [ ] Multi-step forms indicate progress ("Step 2 of 5")
- [ ] Required fields are marked consistently
- [ ] Error summary appears at the top of the form after submission
- [ ] Each error in the summary links to the relevant field
- [ ] Conditional fields are managed with aria-expanded

### Search Results
- [ ] Number of results is announced on load
- [ ] "No results" is communicated to screen readers
- [ ] Filtering updates are announced via aria-live
- [ ] Pagination controls are keyboard accessible

## Quick Test Commands

```bash
# Run axe-core tests
npx playwright test tests/e2e/accessibility/

# Check for missing alt text
grep -rn '<img' --include="*.tsx" --include="*.ts" src/ app/ components/ | grep -v 'alt='

# Check for missing form labels
grep -rn '<input' --include="*.tsx" --include="*.ts" src/ app/ components/ | grep -v 'id='

# Check for click handlers on non-interactive elements
grep -rn 'onClick' --include="*.tsx" src/ app/ components/ | grep '<div\|<span'

# Check heading hierarchy
grep -rn '<h[1-6]' --include="*.tsx" src/ app/ components/ | sort
```

## Tools

- **axe DevTools** — Browser extension for live page testing
- **Lighthouse** — Built into Chrome DevTools, includes accessibility audit
- **WAVE** — Browser extension that overlays issues on the page
- **Colour Contrast Analyser** — Desktop app for checking contrast ratios
- **NVDA** — Free screen reader for Windows
- **VoiceOver** — Built-in screen reader for Mac (Cmd+F5)
