---
name: component-storybook
description: "Use this skill whenever the user mentions Storybook, component library, component catalog, component documentation, design system docs, visual testing, visual regression, Chromatic, component isolation, component stories, 'document my components', 'component playground', 'preview components', design tokens, story, stories, 'show me all my components', or ANY task related to building a component documentation system — even if they don't explicitly say 'Storybook'. This skill sets up isolated component development and visual testing."
---

# Component Storybook

Build a component documentation system and visual testing pipeline for your Next.js App Router + TypeScript + Tailwind CSS + shadcn/ui project.

## Stack Context

This skill targets the following stack:

- **Framework**: Next.js 14+ with App Router
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS v3/v4
- **Component Library**: shadcn/ui (Radix UI primitives)
- **Deployment**: Vercel
- **Storybook**: v8+
- **Visual Testing**: Chromatic

Storybook runs as a separate dev server alongside your Next.js app. It renders components in isolation — no routing, no server components, no data fetching. Just your UI, by itself, in every possible state.

## Storybook Setup for Next.js App Router

### Installation

Run the Storybook initializer. It detects Next.js automatically:

```bash
npx storybook@latest init
```

This scaffolds the `.storybook/` config directory and adds scripts to `package.json`. If it asks about the framework, pick `nextjs`.

After init, install the Storybook addon for Tailwind CSS support:

```bash
npm install -D @storybook/addon-styling-webpack @storybook/addon-themes
```

### Configuration Files

Storybook needs two config files in `.storybook/`:

**`.storybook/main.ts`** — tells Storybook where to find stories and what addons to load:

```ts
import type { StorybookConfig } from "@storybook/nextjs";

const config: StorybookConfig = {
  stories: ["../src/**/*.mdx", "../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"],
  addons: [
    "@storybook/addon-onboarding",
    "@storybook/addon-essentials",
    "@chromatic-com/storybook",
    "@storybook/addon-interactions",
    "@storybook/addon-themes",
  ],
  framework: {
    name: "@storybook/nextjs",
    options: {},
  },
  staticDirs: ["../public"],
};
export default config;
```

**`.storybook/preview.ts`** — global decorators, parameters, and CSS imports:

```ts
import type { Preview } from "@storybook/react";
import "../src/app/globals.css";

const preview: Preview = {
  parameters: {
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
    nextjs: {
      appDirectory: true,
    },
  },
};

export default preview;
```

The `globals.css` import loads your Tailwind styles into every story. Without it your components render unstyled.

### Running Storybook

```bash
npm run storybook
```

Opens at `http://localhost:6006`. Hot-reloads when you edit stories or components.

Build a static Storybook for deployment:

```bash
npm run build-storybook
```

Outputs to `storybook-static/`. Add this to `.gitignore`.

## Writing Your First Story

Stories live next to the components they document. Create a story file with the `.stories.tsx` extension.

### Button Component Example

Say you have a Button component at `src/components/ui/button.tsx` (the standard shadcn/ui Button). Here is a complete story file:

```tsx
// src/components/ui/button.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./button";

const meta = {
  title: "UI/Button",
  component: Button,
  parameters: {
    layout: "centered",
  },
  tags: ["autodocs"],
  argTypes: {
    variant: {
      control: "select",
      options: [
        "default",
        "destructive",
        "outline",
        "secondary",
        "ghost",
        "link",
      ],
    },
    size: {
      control: "select",
      options: ["default", "sm", "lg", "icon"],
    },
  },
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Default: Story = {
  args: {
    children: "Click me",
    variant: "default",
  },
};

export const Destructive: Story = {
  args: {
    children: "Delete",
    variant: "destructive",
  },
};

export const Outline: Story = {
  args: {
    children: "Outline",
    variant: "outline",
  },
};

export const Ghost: Story = {
  args: {
    children: "Ghost",
    variant: "ghost",
  },
};

export const Small: Story = {
  args: {
    children: "Small",
    size: "sm",
  },
};

export const Large: Story = {
  args: {
    children: "Large",
    size: "lg",
  },
};

export const WithIcon: Story = {
  args: {
    children: (
      <>
        <svg
          xmlns="http://www.w3.org/2000/svg"
          width="16"
          height="16"
          viewBox="0 0 24 24"
          fill="none"
          stroke="currentColor"
          strokeWidth="2"
        >
          <path d="M5 12h14M12 5l7 7-7 7" />
        </svg>
        Next
      </>
    ),
  },
};

export const Loading: Story = {
  args: {
    children: "Loading...",
    disabled: true,
  },
};
```

Open Storybook and you will see a "UI / Button" entry in the sidebar with every variant listed. The `tags: ["autodocs"]` line auto-generates a documentation page from your component props.

## Component Story Patterns

### Args and Controls

Args are the props passed to your component. Storybook auto-generates controls in the panel for each arg:

```tsx
export const Playground: Story = {
  args: {
    children: "Play with me",
    variant: "default",
    size: "default",
    disabled: false,
  },
};
```

Users can tweak every prop in the Storybook UI without touching code.

### Decorators

Decorators wrap your stories. Use them for layout, providers, or theming:

```tsx
const meta = {
  title: "UI/Card",
  component: Card,
  decorators: [
    (Story) => (
      <div className="max-w-md mx-auto p-8">
        <Story />
      </div>
    ),
  ],
} satisfies Meta<typeof Card>;
```

### Multiple Variants in One View

Use a render function to show all variants at once:

```tsx
export const AllVariants: Story = {
  render: () => (
    <div className="flex flex-wrap gap-4">
      <Button variant="default">Default</Button>
      <Button variant="destructive">Destructive</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="secondary">Secondary</Button>
      <Button variant="ghost">Ghost</Button>
      <Button variant="link">Link</Button>
    </div>
  ),
};
```

### Composing Stories

Build complex stories from simpler ones:

```tsx
export const CardWithActions: Story = {
  render: () => (
    <Card>
      <CardHeader>
        <CardTitle>Are you sure?</CardTitle>
        <CardDescription>This action cannot be undone.</CardDescription>
      </CardHeader>
      <CardFooter className="flex justify-between">
        <Button variant="outline">Cancel</Button>
        <Button variant="destructive">Delete</Button>
      </CardFooter>
    </Card>
  ),
};
```

## Design Tokens in Storybook

### Tailwind Theme Integration

Your Tailwind config defines your design tokens. Storybook should reflect them. Create a design tokens story:

```tsx
// src/stories/DesignTokens.stories.tsx
import type { Meta, StoryObj } from "@storybook/react";

const meta = {
  title: "Design System/Tokens",
  parameters: {
    layout: "fullscreen",
  },
} satisfies Meta<{}>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Colors: Story = {
  render: () => (
    <div className="p-8 space-y-8">
      <h2 className="text-2xl font-bold">Colors</h2>
      <div className="grid grid-cols-4 gap-4">
        {[
          ["--background", "bg-background", "Background"],
          ["--foreground", "bg-foreground", "Foreground"],
          ["--primary", "bg-primary", "Primary"],
          ["--secondary", "bg-secondary", "Secondary"],
          ["--muted", "bg-muted", "Muted"],
          ["--accent", "bg-accent", "Accent"],
          ["--destructive", "bg-destructive", "Destructive"],
          ["--border", "bg-border", "Border"],
        ].map(([, className, label]) => (
          <div key={label} className="space-y-2">
            <div className={`${className} h-16 rounded-lg border`} />
            <p className="text-sm font-medium">{label}</p>
          </div>
        ))}
      </div>
    </div>
  ),
};

export const Typography: Story = {
  render: () => (
    <div className="p-8 space-y-4">
      <h2 className="text-2xl font-bold">Typography Scale</h2>
      <p className="text-xs">text-xs — The quick brown fox</p>
      <p className="text-sm">text-sm — The quick brown fox</p>
      <p className="text-base">text-base — The quick brown fox</p>
      <p className="text-lg">text-lg — The quick brown fox</p>
      <p className="text-xl">text-xl — The quick brown fox</p>
      <p className="text-2xl">text-2xl — The quick brown fox</p>
      <p className="text-3xl">text-3xl — The quick brown fox</p>
      <p className="text-4xl">text-4xl — The quick brown fox</p>
    </div>
  ),
};

export const Spacing: Story = {
  render: () => (
    <div className="p-8 space-y-4">
      <h2 className="text-2xl font-bold">Spacing Scale</h2>
      {[1, 2, 3, 4, 6, 8, 12, 16, 24].map((size) => (
        <div key={size} className="flex items-center gap-4">
          <span className="w-12 text-sm text-muted-foreground">
            {size * 4}px
          </span>
          <div
            className="h-4 bg-primary rounded"
            style={{ width: `${size * 16}px` }}
          />
        </div>
      ))}
    </div>
  ),
};
```

This gives you a living style guide that always matches your actual Tailwind config.

## Testing in Storybook

### Interaction Tests with Play Functions

Play functions let you script user interactions directly inside a story. They run in the browser and use Testing Library under the hood:

```tsx
import { expect, userEvent, within } from "@storybook/test";

export const ClickInteraction: Story = {
  args: {
    children: "Count: 0",
  },
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);
    const button = canvas.getByRole("button");

    await userEvent.click(button);
    await expect(button).toHaveTextContent("Count: 1");
  },
};
```

### Form Interaction Test

```tsx
export const FormSubmission: Story = {
  play: async ({ canvasElement }) => {
    const canvas = within(canvasElement);

    await userEvent.type(canvas.getByLabelText("Email"), "test@example.com");
    await userEvent.type(canvas.getByLabelText("Password"), "password123");
    await userEvent.click(canvas.getByRole("button", { name: "Sign In" }));

    await expect(canvas.getByText("Welcome!")).toBeInTheDocument();
  },
};
```

### Running Tests

Run all interaction tests from the CLI:

```bash
npx test-storybook
```

Add to your CI pipeline alongside your regular test suite.

## Visual Regression with Chromatic

Chromatic catches visual changes by comparing screenshots of every story between builds.

### Setup

```bash
npm install -D chromatic
```

Get your project token from [chromatic.com](https://www.chromatic.com/) and add it to your environment:

```bash
npx chromatic --project-token=<your-token>
```

### CI Integration with GitHub Actions

```yaml
# .github/workflows/chromatic.yml
name: Chromatic
on: push

jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - uses: chromaui/action@latest
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
```

Every push uploads snapshots. PRs get a status check showing what changed visually. Reviewers approve or reject visual diffs directly in Chromatic's UI.

See `references/chromatic-setup.md` for the full pipeline configuration.

## Documentation Pages

### MDX Docs Alongside Stories

Create rich documentation pages using MDX:

```mdx
{/* src/components/ui/button.mdx */}
import { Meta, Story, Canvas, Controls } from "@storybook/blocks";
import * as ButtonStories from "./button.stories";

<Meta of={ButtonStories} />

# Button

Buttons trigger actions. Use the right variant for the right context.

## Default

The standard button for primary actions:

<Canvas of={ButtonStories.Default} />

## Controls

Play with all available props:

<Controls />

## Variants

<Canvas of={ButtonStories.AllVariants} />

## Usage Guidelines

- Use `default` for primary actions
- Use `destructive` for delete/remove actions
- Use `outline` for secondary actions
- Use `ghost` for tertiary or toolbar actions
- Use `link` when you want it to look like a hyperlink
```

### Auto-generated Docs

The `tags: ["autodocs"]` tag on your meta object tells Storybook to auto-generate a docs page. It pulls prop types from TypeScript and shows every exported story in a single scrollable page. You get this for free — no MDX required.

## Organizing Stories

### Folder Structure

Keep stories next to the components they document:

```
src/
  components/
    ui/
      button.tsx
      button.stories.tsx
      card.tsx
      card.stories.tsx
      input.tsx
      input.stories.tsx
    features/
      auth/
        login-form.tsx
        login-form.stories.tsx
      dashboard/
        stats-card.tsx
        stats-card.stories.tsx
  stories/
    DesignTokens.stories.tsx
    Introduction.mdx
```

### Naming Conventions

The `title` field in your meta controls the sidebar hierarchy. Use slashes for nesting:

```tsx
// Appears under "UI" group
title: "UI/Button"

// Appears under "Features > Auth" group
title: "Features/Auth/LoginForm"

// Appears under "Design System" group
title: "Design System/Tokens"

// Appears under "Pages" group
title: "Pages/Dashboard"
```

### Story Naming

Export names become story names in the sidebar. Use PascalCase:

```tsx
export const Default: Story = { ... };        // "Default"
export const WithIcon: Story = { ... };       // "With Icon"
export const LargeDisabled: Story = { ... };  // "Large Disabled"
```

Storybook auto-inserts spaces before capital letters. `WithIcon` becomes "With Icon" in the UI.

## Rules

1. **Stories live next to components.** Place `button.stories.tsx` in the same directory as `button.tsx`. Never in a separate top-level stories folder (except for design system overview pages).

2. **Every exported component gets a story.** If it is in your component library, it has a story. No exceptions. Undocumented components are invisible components.

3. **Import globals.css in preview.ts.** Without it, Tailwind classes do not work in Storybook. Your components will render as unstyled HTML.

4. **Use `satisfies Meta<typeof Component>` for type safety.** This gives you autocomplete on args and catches prop mismatches at compile time. Never use plain object types.

5. **Tag stories with `["autodocs"]` by default.** Auto-generated docs pages are free documentation. Only skip this tag if you are writing custom MDX docs instead.

6. **Test interactions with play functions, not manual clicking.** If a story has interactive behavior (forms, toggles, modals), add a play function. These run in CI and catch regressions.

7. **Add Chromatic to your CI pipeline on day one.** Visual regression testing catches CSS bugs that unit tests miss. Do not wait until your component library is "big enough" — start immediately.

8. **Keep story files under 200 lines.** If a story file gets long, split it. One file per component variant group. For example: `button.stories.tsx` for basic variants, `button-with-icons.stories.tsx` for icon combinations.
