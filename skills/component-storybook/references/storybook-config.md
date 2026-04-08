# Storybook Configuration for Next.js + Tailwind + shadcn

## Installation

```bash
npx storybook@latest init --framework @storybook/nextjs
```

## .storybook/main.ts

```typescript
// .storybook/main.ts
import type { StorybookConfig } from "@storybook/nextjs";

const config: StorybookConfig = {
  stories: [
    "../components/**/*.stories.@(ts|tsx)",
    "../app/**/*.stories.@(ts|tsx)",
  ],
  addons: [
    "@storybook/addon-essentials",
    "@storybook/addon-interactions",
    "@storybook/addon-a11y",
    "@storybook/addon-themes",
  ],
  framework: {
    name: "@storybook/nextjs",
    options: {
      nextConfigPath: "../next.config.ts",
    },
  },
  staticDirs: ["../public"],
  typescript: {
    reactDocgen: "react-docgen-typescript",
  },
};

export default config;
```

## .storybook/preview.ts

```typescript
// .storybook/preview.ts
import type { Preview } from "@storybook/react";
import "../app/globals.css"; // Load Tailwind styles

const preview: Preview = {
  parameters: {
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
    layout: "centered",
    backgrounds: {
      default: "light",
      values: [
        { name: "light", value: "#ffffff" },
        { name: "dark", value: "#0f172a" },
        { name: "gray", value: "#f1f5f9" },
      ],
    },
    viewport: {
      viewports: {
        mobile: { name: "Mobile", styles: { width: "375px", height: "812px" } },
        tablet: { name: "Tablet", styles: { width: "768px", height: "1024px" } },
        desktop: { name: "Desktop", styles: { width: "1440px", height: "900px" } },
      },
    },
  },
  decorators: [
    (Story) => (
      <div className="font-sans antialiased">
        <Story />
      </div>
    ),
  ],
};

export default preview;
```

## Path Aliases

Storybook needs to resolve `@/` imports the same way Next.js does. The `@storybook/nextjs` framework handles this automatically by reading your `tsconfig.json`. If it doesn't work, add this to `.storybook/main.ts`:

```typescript
// .storybook/main.ts
import path from "path";

const config: StorybookConfig = {
  // ... other config
  webpackFinal: async (config) => {
    if (config.resolve) {
      config.resolve.alias = {
        ...config.resolve.alias,
        "@": path.resolve(__dirname, ".."),
      };
    }
    return config;
  },
};
```

## Dark Mode Decorator

```typescript
// .storybook/preview.ts — with dark mode
import { withThemeByClassName } from "@storybook/addon-themes";

const preview: Preview = {
  decorators: [
    withThemeByClassName({
      themes: {
        light: "",
        dark: "dark",
      },
      defaultTheme: "light",
    }),
    (Story) => (
      <div className="font-sans antialiased">
        <Story />
      </div>
    ),
  ],
};
```

## Font Loading

```typescript
// .storybook/preview.ts — load Google Fonts
import { Inter } from "next/font/google";

const inter = Inter({ subsets: ["latin"] });

const preview: Preview = {
  decorators: [
    (Story) => (
      <div className={`${inter.className} antialiased`}>
        <Story />
      </div>
    ),
  ],
};
```

## NPM Scripts

```json
{
  "scripts": {
    "storybook": "storybook dev -p 6006",
    "build-storybook": "storybook build -o storybook-static"
  }
}
```

## .gitignore Addition

```
storybook-static/
```
