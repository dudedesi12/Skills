---
name: tailwind-design-system
description: "Use this skill whenever the user mentions UI, design, styling, CSS, Tailwind, colors, fonts, typography, components, buttons, cards, modals, forms, tables, navigation, sidebar, header, footer, dark mode, responsive, mobile layout, desktop layout, animations, transitions, hover effects, shadcn, Radix, accessibility, ARIA, focus states, brand colors, design tokens, toast notifications, dropdown, tabs, accordion, or ANY visual/styling task — even if they don't explicitly say 'Tailwind' or 'design'. This skill creates consistent, beautiful UI."
---

# Tailwind Design System

Build consistent, accessible UI for Next.js App Router applications using Tailwind CSS, shadcn/ui, and Radix primitives.

## Tailwind Configuration with Brand Tokens

Set up your design system foundation with custom colors, fonts, and spacing.

```typescript
// tailwind.config.ts
import type { Config } from "tailwindcss";

const config: Config = {
  darkMode: "class",
  content: [
    "./src/pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        brand: {
          50: "#f0f7ff",
          100: "#e0effe",
          200: "#b9dffe",
          300: "#7cc4fd",
          400: "#36a6fa",
          500: "#0c8ceb",
          600: "#006fc9",
          700: "#0158a3",
          800: "#064b86",
          900: "#0b3f6f",
          950: "#07284a",
        },
        surface: {
          DEFAULT: "#ffffff",
          secondary: "#f8fafc",
          dark: "#0f172a",
          "dark-secondary": "#1e293b",
        },
        text: {
          primary: "#0f172a",
          secondary: "#475569",
          muted: "#94a3b8",
          "dark-primary": "#f8fafc",
          "dark-secondary": "#cbd5e1",
        },
      },
      fontFamily: {
        sans: ["Inter", "system-ui", "sans-serif"],
        mono: ["JetBrains Mono", "Fira Code", "monospace"],
      },
      fontSize: {
        "display-lg": ["3.5rem", { lineHeight: "1.1", letterSpacing: "-0.02em" }],
        "display": ["2.5rem", { lineHeight: "1.2", letterSpacing: "-0.02em" }],
        "heading": ["1.75rem", { lineHeight: "1.3", letterSpacing: "-0.01em" }],
        "subheading": ["1.25rem", { lineHeight: "1.4" }],
        "body": ["1rem", { lineHeight: "1.6" }],
        "small": ["0.875rem", { lineHeight: "1.5" }],
        "caption": ["0.75rem", { lineHeight: "1.4" }],
      },
      spacing: {
        "page": "1rem",
        "section": "4rem",
        "component": "1.5rem",
      },
      borderRadius: {
        "brand": "0.625rem",
      },
      animation: {
        "fade-in": "fadeIn 0.3s ease-out",
        "slide-up": "slideUp 0.3s ease-out",
        "slide-down": "slideDown 0.3s ease-out",
        "scale-in": "scaleIn 0.2s ease-out",
      },
      keyframes: {
        fadeIn: {
          "0%": { opacity: "0" },
          "100%": { opacity: "1" },
        },
        slideUp: {
          "0%": { opacity: "0", transform: "translateY(10px)" },
          "100%": { opacity: "1", transform: "translateY(0)" },
        },
        slideDown: {
          "0%": { opacity: "0", transform: "translateY(-10px)" },
          "100%": { opacity: "1", transform: "translateY(0)" },
        },
        scaleIn: {
          "0%": { opacity: "0", transform: "scale(0.95)" },
          "100%": { opacity: "1", transform: "scale(1)" },
        },
      },
    },
  },
  plugins: [],
};

export default config;
```

### Loading Custom Fonts

```typescript
// app/layout.tsx
import { Inter, JetBrains_Mono } from "next/font/google";
import "./globals.css";

const inter = Inter({
  subsets: ["latin"],
  variable: "--font-sans",
  display: "swap",
});

const jetbrainsMono = JetBrains_Mono({
  subsets: ["latin"],
  variable: "--font-mono",
  display: "swap",
});

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" className={`${inter.variable} ${jetbrainsMono.variable}`}>
      <body className="font-sans bg-surface text-text-primary dark:bg-surface-dark dark:text-text-dark-primary antialiased">
        {children}
      </body>
    </html>
  );
}
```

## Reusable Component Patterns

See `references/components.md` for 15+ copy-paste ready components: Button, Card, Modal, Form, Table, Nav, Sidebar, Toast, Badge, Avatar, Input, Select, Tabs, Accordion, and Alert.

Key patterns used across all components:
- **Variants via Record type** — map variant names to Tailwind class strings
- **forwardRef** for components that need ref access (Button, Input, Select)
- **Dark mode** — every component includes `dark:` variants
- **Focus rings** — `focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500`
- **Loading states** — spinner SVG with `animate-spin` for async actions
- **Error states** — red borders and `role="alert"` for validation messages

## Responsive Design

Use Tailwind's mobile-first breakpoints. Write base styles for mobile, then add larger breakpoints.

```
sm:  640px   — large phones / small tablets
md:  768px   — tablets
lg:  1024px  — small laptops
xl:  1280px  — desktops
2xl: 1536px  — large screens
```

### Responsive Layout Example

```typescript
// components/PageLayout.tsx
export function PageLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen">
      {/* Mobile: stack vertically. Desktop: sidebar + content */}
      <div className="flex flex-col lg:flex-row">
        {/* Sidebar: hidden on mobile, visible on desktop */}
        <aside className="hidden lg:block lg:w-64 lg:shrink-0 border-r border-gray-200 dark:border-gray-700 p-4">
          <nav aria-label="Main navigation">
            {/* Navigation links */}
          </nav>
        </aside>

        {/* Main content area */}
        <main className="flex-1 p-page md:p-6 lg:p-8 max-w-5xl">
          {children}
        </main>
      </div>
    </div>
  );
}
```

### Responsive Grid

```typescript
// components/CardGrid.tsx
export function CardGrid({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4 md:gap-6">
      {children}
    </div>
  );
}
```

See `references/responsive.md` for more patterns.

## Dark Mode

Use Tailwind's `class` strategy (set in `tailwind.config.ts` above).

### Dark Mode Toggle

```typescript
// components/ThemeToggle.tsx
"use client";

import { useEffect, useState } from "react";

export function ThemeToggle() {
  const [dark, setDark] = useState(false);

  useEffect(() => {
    const stored = localStorage.getItem("theme");
    const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    const isDark = stored === "dark" || (!stored && prefersDark);
    setDark(isDark);
    document.documentElement.classList.toggle("dark", isDark);
  }, []);

  function toggle() {
    const next = !dark;
    setDark(next);
    localStorage.setItem("theme", next ? "dark" : "light");
    document.documentElement.classList.toggle("dark", next);
  }

  return (
    <button
      onClick={toggle}
      className="p-2 rounded-brand hover:bg-gray-100 dark:hover:bg-gray-800 transition-colors"
      aria-label={dark ? "Switch to light mode" : "Switch to dark mode"}
    >
      {dark ? (
        <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z" />
        </svg>
      ) : (
        <svg className="w-5 h-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" aria-hidden="true">
          <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M20.354 15.354A9 9 0 018.646 3.646 9.003 9.003 0 0012 21a9.003 9.003 0 008.354-5.646z" />
        </svg>
      )}
    </button>
  );
}
```

### Preventing Flash of Wrong Theme

Add an inline script to `<head>` in `app/layout.tsx` that reads `localStorage.getItem('theme')` and applies the `dark` class before paint. This prevents a flash of the wrong theme on load.

## Animations and Transitions

### Tailwind Transitions

```html
<!-- Hover scale -->
<div class="transition-transform duration-200 hover:scale-105">
  Hover me
</div>

<!-- Color transition -->
<button class="transition-colors duration-150 bg-brand-600 hover:bg-brand-700">
  Click me
</button>

<!-- Opacity fade -->
<div class="transition-opacity duration-300 opacity-0 data-[visible=true]:opacity-100">
  Fades in
</div>
```

### Framer Motion Basics

```bash
npm install framer-motion
```

```typescript
// components/AnimatedCard.tsx
"use client";

import { motion } from "framer-motion";

export function AnimatedCard({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3, ease: "easeOut" }}
      className="bg-white dark:bg-surface-dark-secondary rounded-brand shadow-md p-component"
    >
      {children}
    </motion.div>
  );
}
```

## shadcn/ui Integration

shadcn/ui gives you copy-paste components built on Radix primitives.

### Setup

```bash
npx shadcn@latest init
```

Choose these options:
- Style: Default
- Base color: Slate
- CSS variables: Yes

### Adding Components

```bash
npx shadcn@latest add button dialog input select tabs toast
```

### Customizing shadcn Components

After adding, components live in `components/ui/`. Edit them freely.

```typescript
// components/ui/button.tsx — customize the variants
// shadcn generates this file. Modify the variant classes
// to match your brand colors from tailwind.config.ts
```

## Accessibility

Every component must be keyboard-navigable and screen-reader friendly. See `references/accessibility.md` for full patterns.

### Key Rules

1. **All interactive elements need focus styles:**
```html
<button class="focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-brand-500 focus-visible:ring-offset-2">
  Accessible button
</button>
```

2. **Images need alt text:**
```typescript
<Image src={url} alt="Description of the image" width={400} height={300} />
```

3. **Forms need labels:** Every `<input>` must have a `<label>` with a matching `htmlFor`/`id`. See `references/components.md` for the FormField component.

4. **Color contrast:** Ensure at least 4.5:1 ratio for text. The brand color scale above is designed for contrast compliance.

5. **Skip navigation:** Add a skip link as the first element in `<body>` using `sr-only focus:not-sr-only` classes. See `references/accessibility.md` for the full pattern.

## Typography Scale

Use the custom font sizes defined in `tailwind.config.ts`:

```html
<h1 class="text-display-lg font-bold">Hero Heading</h1>
<h2 class="text-display font-bold">Page Title</h2>
<h3 class="text-heading font-semibold">Section Heading</h3>
<h4 class="text-subheading font-medium">Subsection</h4>
<p class="text-body">Body text goes here.</p>
<p class="text-small text-text-secondary">Supporting text.</p>
<span class="text-caption text-text-muted">Caption or metadata.</span>
```

## Layout Patterns

### Container with Max Width

```typescript
// components/Container.tsx
export function Container({ children, className = "" }: { children: React.ReactNode; className?: string }) {
  return (
    <div className={`mx-auto max-w-7xl px-page sm:px-6 lg:px-8 ${className}`}>
      {children}
    </div>
  );
}
```

### Two-Column Layout

```html
<div class="grid grid-cols-1 lg:grid-cols-[1fr_300px] gap-8">
  <main>Primary content</main>
  <aside>Sidebar</aside>
</div>
```

### Sticky Header

```typescript
// components/Header.tsx
export function Header() {
  return (
    <header className="sticky top-0 z-40 bg-white/80 dark:bg-surface-dark/80 backdrop-blur-sm border-b border-gray-200 dark:border-gray-800">
      <div className="mx-auto max-w-7xl px-page sm:px-6 lg:px-8 h-16 flex items-center justify-between">
        <span className="text-subheading font-semibold">MyApp</span>
        <nav className="flex items-center gap-4" aria-label="Main navigation">
          {/* Nav links */}
        </nav>
      </div>
    </header>
  );
}
```
