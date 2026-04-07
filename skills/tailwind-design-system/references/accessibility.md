# Accessibility Patterns

## Focus Management

### Visible Focus Rings

Every interactive element needs a visible focus indicator. Use `focus-visible` (not `focus`) so it only shows for keyboard users, not mouse clicks.

```html
<!-- Standard focus ring -->
<button class="focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2">
  Button
</button>

<!-- Focus ring on dark backgrounds -->
<button class="focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-white focus-visible:ring-offset-2 focus-visible:ring-offset-gray-900">
  Dark mode button
</button>
```

### Skip Navigation Link

Place this as the first focusable element in `<body>`:

```typescript
// app/layout.tsx
<a
  href="#main-content"
  className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-white focus:text-brand-600 focus:rounded-lg focus:shadow-lg focus:ring-2 focus:ring-brand-500"
>
  Skip to main content
</a>

{/* Later in the page */}
<main id="main-content" tabIndex={-1}>
  {children}
</main>
```

### Focus Trap for Modals

When a modal is open, focus should stay inside it. The native `<dialog>` element handles this automatically:

```typescript
// components/AccessibleModal.tsx
"use client";

import { useEffect, useRef } from "react";

export function AccessibleModal({
  open,
  onClose,
  title,
  children,
}: {
  open: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}) {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;
    if (open) {
      dialog.showModal();
    } else {
      dialog.close();
    }
  }, [open]);

  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;
    const handleClose = () => onClose();
    dialog.addEventListener("close", handleClose);
    return () => dialog.removeEventListener("close", handleClose);
  }, [onClose]);

  return (
    <dialog
      ref={dialogRef}
      className="backdrop:bg-black/50 bg-white dark:bg-gray-900 rounded-lg shadow-xl p-6 max-w-lg w-full"
      aria-labelledby="dialog-title"
    >
      <h2 id="dialog-title" className="text-lg font-semibold mb-4">
        {title}
      </h2>
      {children}
      <button
        onClick={onClose}
        className="absolute top-4 right-4 p-1 rounded hover:bg-gray-100 dark:hover:bg-gray-800"
        aria-label="Close dialog"
      >
        <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
        </svg>
      </button>
    </dialog>
  );
}
```

## ARIA Patterns

### Landmarks

```html
<header role="banner">Site header</header>
<nav aria-label="Main navigation">Navigation</nav>
<main role="main">Page content</main>
<aside aria-label="Related links">Sidebar</aside>
<footer role="contentinfo">Site footer</footer>
```

### Live Regions (for dynamic updates)

```html
<!-- Announce changes to screen readers -->
<div aria-live="polite" aria-atomic="true">
  {statusMessage}
</div>

<!-- For urgent updates (errors) -->
<div role="alert">
  {errorMessage}
</div>

<!-- For form errors -->
<p role="alert" className="text-sm text-red-600">
  This field is required
</p>
```

### Accessible Form Pattern

```typescript
// components/AccessibleForm.tsx
"use client";

import { useState } from "react";

export function AccessibleForm() {
  const [errors, setErrors] = useState<Record<string, string>>({});

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const form = new FormData(e.currentTarget);
    const newErrors: Record<string, string> = {};

    if (!form.get("name")) newErrors.name = "Name is required";
    if (!form.get("email")) newErrors.email = "Email is required";

    setErrors(newErrors);

    if (Object.keys(newErrors).length > 0) {
      // Focus the first field with an error
      const firstErrorField = document.getElementById(Object.keys(newErrors)[0]);
      firstErrorField?.focus();
      return;
    }

    // Submit the form
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      <div className="space-y-4">
        <div>
          <label htmlFor="name" className="block text-sm font-medium mb-1">
            Name <span aria-hidden="true" className="text-red-500">*</span>
          </label>
          <input
            id="name"
            name="name"
            type="text"
            required
            aria-required="true"
            aria-invalid={errors.name ? "true" : undefined}
            aria-describedby={errors.name ? "name-error" : undefined}
            className="w-full rounded-lg border border-gray-300 dark:border-gray-600 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-brand-500"
          />
          {errors.name && (
            <p id="name-error" className="mt-1 text-sm text-red-600" role="alert">
              {errors.name}
            </p>
          )}
        </div>

        <div>
          <label htmlFor="email" className="block text-sm font-medium mb-1">
            Email <span aria-hidden="true" className="text-red-500">*</span>
          </label>
          <input
            id="email"
            name="email"
            type="email"
            required
            aria-required="true"
            aria-invalid={errors.email ? "true" : undefined}
            aria-describedby={errors.email ? "email-error" : "email-help"}
            className="w-full rounded-lg border border-gray-300 dark:border-gray-600 px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-brand-500"
          />
          {errors.email ? (
            <p id="email-error" className="mt-1 text-sm text-red-600" role="alert">
              {errors.email}
            </p>
          ) : (
            <p id="email-help" className="mt-1 text-sm text-gray-500">
              We will never share your email.
            </p>
          )}
        </div>

        <button
          type="submit"
          className="px-4 py-2 bg-brand-600 text-white rounded-lg font-medium hover:bg-brand-700 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2"
        >
          Submit
        </button>
      </div>
    </form>
  );
}
```

## Keyboard Navigation

### Custom Keyboard Handlers

For custom interactive components (dropdowns, menus), handle these keys:

| Key | Action |
|-----|--------|
| `Enter` / `Space` | Activate the focused item |
| `Escape` | Close the dropdown/modal |
| `ArrowDown` | Move focus to next item |
| `ArrowUp` | Move focus to previous item |
| `Home` | Move focus to first item |
| `End` | Move focus to last item |
| `Tab` | Move focus to next focusable element |

```typescript
// Example: keyboard-navigable list
function handleKeyDown(e: React.KeyboardEvent, items: string[], activeIndex: number, setActiveIndex: (i: number) => void) {
  switch (e.key) {
    case "ArrowDown":
      e.preventDefault();
      setActiveIndex(Math.min(activeIndex + 1, items.length - 1));
      break;
    case "ArrowUp":
      e.preventDefault();
      setActiveIndex(Math.max(activeIndex - 1, 0));
      break;
    case "Home":
      e.preventDefault();
      setActiveIndex(0);
      break;
    case "End":
      e.preventDefault();
      setActiveIndex(items.length - 1);
      break;
  }
}
```

## Color Contrast

WCAG AA requires:
- **Normal text:** 4.5:1 contrast ratio
- **Large text (18px+ bold or 24px+):** 3:1 contrast ratio
- **UI components and icons:** 3:1 contrast ratio

### Safe Color Combinations

| Background | Text Color | Tailwind Classes |
|-----------|------------|-----------------|
| White | Gray 700+ | `bg-white text-gray-700` |
| Gray 50 | Gray 700+ | `bg-gray-50 text-gray-700` |
| Gray 900 | Gray 300 or lighter | `bg-gray-900 text-gray-300` |
| Brand 600 | White | `bg-brand-600 text-white` |
| Red 600 | White | `bg-red-600 text-white` |

### Screen Reader Utilities

```html
<!-- Hide visually but keep for screen readers -->
<span class="sr-only">Open menu</span>

<!-- Hide from screen readers (decorative) -->
<svg aria-hidden="true">...</svg>

<!-- Announce to screen readers -->
<div aria-live="polite">3 items in cart</div>
```

## Accessible Icon Buttons

Icon-only buttons must have a label:

```html
<!-- Option 1: aria-label -->
<button aria-label="Delete item" class="p-2 rounded hover:bg-gray-100">
  <svg aria-hidden="true" class="w-5 h-5">...</svg>
</button>

<!-- Option 2: sr-only text -->
<button class="p-2 rounded hover:bg-gray-100">
  <svg aria-hidden="true" class="w-5 h-5">...</svg>
  <span class="sr-only">Delete item</span>
</button>
```

## Testing Checklist

1. **Keyboard:** Tab through the entire page. Can you reach and operate every interactive element?
2. **Screen reader:** Test with VoiceOver (Mac) or NVDA (Windows). Are all elements announced correctly?
3. **Zoom:** Zoom to 200%. Does the layout still work?
4. **Color:** Turn on high-contrast mode. Is everything readable?
5. **Motion:** Enable "Reduce motion" in OS settings. Are animations respected?

```css
/* Respect user's motion preference */
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

Add this to your `globals.css` file.
