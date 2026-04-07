# Responsive Design Patterns

## Breakpoint Reference

| Prefix | Min Width | Typical Device |
|--------|-----------|----------------|
| (none) | 0px | Mobile (default) |
| `sm:` | 640px | Large phone / small tablet |
| `md:` | 768px | Tablet |
| `lg:` | 1024px | Small laptop |
| `xl:` | 1280px | Desktop |
| `2xl:` | 1536px | Large desktop |

Always write mobile styles first, then add breakpoints for larger screens.

## Common Layout Switches

### Stack to Row

```html
<!-- Stacks on mobile, side-by-side on tablet+ -->
<div class="flex flex-col md:flex-row gap-4">
  <div class="md:w-1/2">Left</div>
  <div class="md:w-1/2">Right</div>
</div>
```

### 1-2-3 Column Grid

```html
<!-- 1 column on mobile, 2 on tablet, 3 on desktop -->
<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
  <div>Card 1</div>
  <div>Card 2</div>
  <div>Card 3</div>
</div>
```

### Sidebar Layout

```html
<!-- Stacked on mobile, sidebar on desktop -->
<div class="flex flex-col lg:flex-row min-h-screen">
  <aside class="w-full lg:w-64 lg:shrink-0 border-b lg:border-b-0 lg:border-r border-gray-200 dark:border-gray-700 p-4">
    Sidebar
  </aside>
  <main class="flex-1 p-4 lg:p-8">
    Main content
  </main>
</div>
```

### Full-Width to Contained

```html
<!-- Full width on mobile, max-width container on desktop -->
<div class="w-full max-w-none lg:max-w-4xl lg:mx-auto px-4">
  Content
</div>
```

## Responsive Typography

```html
<h1 class="text-2xl sm:text-3xl lg:text-4xl font-bold">
  Scales with screen
</h1>
<p class="text-sm md:text-base">
  Body text
</p>
```

## Responsive Spacing

```html
<!-- Tighter padding on mobile, more room on desktop -->
<section class="px-4 py-8 md:px-8 md:py-16 lg:px-12 lg:py-24">
  Content
</section>
```

## Show/Hide by Breakpoint

```html
<!-- Mobile only -->
<div class="block md:hidden">Mobile menu button</div>

<!-- Desktop only -->
<nav class="hidden md:flex">Desktop navigation</nav>

<!-- Tablet and up -->
<div class="hidden sm:block">Visible on 640px+</div>
```

## Responsive Navigation Pattern

```typescript
// components/ResponsiveNav.tsx
"use client";

import { useState } from "react";
import Link from "next/link";

const links = [
  { href: "/", label: "Home" },
  { href: "/features", label: "Features" },
  { href: "/pricing", label: "Pricing" },
  { href: "/about", label: "About" },
];

export function ResponsiveNav() {
  const [menuOpen, setMenuOpen] = useState(false);

  return (
    <header className="border-b border-gray-200 dark:border-gray-700">
      <div className="max-w-7xl mx-auto px-4 h-16 flex items-center justify-between">
        <Link href="/" className="font-bold text-lg">
          MyApp
        </Link>

        {/* Desktop nav */}
        <nav className="hidden md:flex items-center gap-6" aria-label="Main navigation">
          {links.map((link) => (
            <Link
              key={link.href}
              href={link.href}
              className="text-sm text-gray-600 hover:text-gray-900 dark:text-gray-400 dark:hover:text-gray-100 transition-colors"
            >
              {link.label}
            </Link>
          ))}
        </nav>

        {/* Mobile menu button */}
        <button
          className="md:hidden p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-800"
          onClick={() => setMenuOpen(!menuOpen)}
          aria-expanded={menuOpen}
          aria-label="Toggle menu"
        >
          <svg className="w-6 h-6" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
            {menuOpen ? (
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" />
            ) : (
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M4 6h16M4 12h16M4 18h16" />
            )}
          </svg>
        </button>
      </div>

      {/* Mobile menu */}
      {menuOpen && (
        <nav className="md:hidden border-t border-gray-200 dark:border-gray-700 px-4 py-2" aria-label="Mobile navigation">
          {links.map((link) => (
            <Link
              key={link.href}
              href={link.href}
              className="block py-3 text-sm text-gray-600 hover:text-gray-900 dark:text-gray-400 dark:hover:text-gray-100"
              onClick={() => setMenuOpen(false)}
            >
              {link.label}
            </Link>
          ))}
        </nav>
      )}
    </header>
  );
}
```

## Responsive Table Pattern

Tables can overflow on mobile. Wrap them in a scrollable container:

```html
<div class="overflow-x-auto -mx-4 sm:mx-0">
  <div class="inline-block min-w-full align-middle">
    <table class="min-w-full">
      <!-- table content -->
    </table>
  </div>
</div>
```

Or switch to a card layout on mobile:

```typescript
// components/ResponsiveTable.tsx
interface Item {
  id: string;
  name: string;
  email: string;
  role: string;
}

export function ResponsiveTable({ items }: { items: Item[] }) {
  return (
    <>
      {/* Desktop table */}
      <div className="hidden md:block overflow-x-auto rounded-lg border border-gray-200 dark:border-gray-700">
        <table className="w-full text-sm">
          <thead className="bg-gray-50 dark:bg-gray-800">
            <tr>
              <th className="px-4 py-3 text-left font-medium">Name</th>
              <th className="px-4 py-3 text-left font-medium">Email</th>
              <th className="px-4 py-3 text-left font-medium">Role</th>
            </tr>
          </thead>
          <tbody className="divide-y divide-gray-200 dark:divide-gray-700">
            {items.map((item) => (
              <tr key={item.id}>
                <td className="px-4 py-3">{item.name}</td>
                <td className="px-4 py-3">{item.email}</td>
                <td className="px-4 py-3">{item.role}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      {/* Mobile cards */}
      <div className="md:hidden space-y-3">
        {items.map((item) => (
          <div
            key={item.id}
            className="bg-white dark:bg-gray-900 rounded-lg border border-gray-200 dark:border-gray-700 p-4 space-y-2"
          >
            <p className="font-medium">{item.name}</p>
            <p className="text-sm text-gray-500">{item.email}</p>
            <p className="text-sm text-gray-500">{item.role}</p>
          </div>
        ))}
      </div>
    </>
  );
}
```

## Responsive Image

```typescript
import Image from "next/image";

// Full width on mobile, constrained on desktop
<div class="w-full md:max-w-md lg:max-w-lg mx-auto">
  <Image
    src="/hero.jpg"
    alt="Hero image"
    width={800}
    height={400}
    sizes="(max-width: 768px) 100vw, (max-width: 1024px) 50vw, 33vw"
    className="rounded-lg w-full h-auto"
    priority
  />
</div>
```

## Responsive Padding Cheat Sheet

| Element | Mobile | Tablet | Desktop |
|---------|--------|--------|---------|
| Page padding | `px-4` | `px-6` | `px-8` |
| Section spacing | `py-8` | `py-12` | `py-20` |
| Card padding | `p-4` | `p-6` | `p-6` |
| Gap between items | `gap-3` | `gap-4` | `gap-6` |
