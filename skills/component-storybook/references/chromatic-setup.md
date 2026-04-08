# Chromatic Visual Regression Setup

## What Chromatic Does

Chromatic takes screenshots of every story on every PR and compares them to the baseline. If anything looks different, it flags the change for review before merging.

## Setup

### 1. Create Account

Go to [chromatic.com](https://www.chromatic.com) and sign in with GitHub. Add your repository.

### 2. Install

```bash
npm install -D chromatic
```

### 3. Get Project Token

From the Chromatic dashboard, copy your project token. Add it as a GitHub secret: `CHROMATIC_PROJECT_TOKEN`.

### 4. First Run (Create Baseline)

```bash
npx chromatic --project-token=<your-token>
```

This uploads all your stories and creates the initial baseline screenshots.

## GitHub Actions Workflow

```yaml
# .github/workflows/chromatic.yml
name: Chromatic

on:
  pull_request:
    branches: [main]

jobs:
  chromatic:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for Chromatic to detect changes

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - run: npm ci

      - name: Run Chromatic
        uses: chromaui/action@latest
        with:
          projectToken: ${{ secrets.CHROMATIC_PROJECT_TOKEN }}
          exitZeroOnChanges: true # Don't fail CI, just flag for review
          exitOnceUploaded: true  # Speed up: exit after upload
          onlyChanged: true       # Only test stories that changed
```

## PR Integration

Once configured, Chromatic adds a status check to every PR:

- **Build passed** — No visual changes detected
- **Changes detected** — Visual differences found, review required
- **Build failed** — Stories have errors

Click "Review changes" to see a side-by-side diff of every visual change.

## Approving Changes

1. Open the Chromatic build link from the PR
2. Review each visual change (before/after)
3. Accept intentional changes
4. Reject unintentional regressions

Accepted changes become the new baseline for future comparisons.

## Handling Dynamic Content

Some content changes between runs (dates, random data). Ignore these:

```tsx
// In your story, use fixed data
export const WithDate: Story = {
  args: {
    date: new Date("2026-01-15T00:00:00Z"), // Fixed date, not new Date()
  },
};
```

For elements you can't control, use Chromatic's `ignoreElements`:

```tsx
export const Dashboard: Story = {
  parameters: {
    chromatic: {
      diffThreshold: 0.3, // Allow 30% pixel difference
      delay: 300,         // Wait 300ms before screenshot (for animations)
    },
  },
};
```

## Ignoring Stories

Skip visual testing for interactive-only stories:

```tsx
export const Interactive: Story = {
  parameters: {
    chromatic: { disableSnapshot: true },
  },
};
```

## Responsive Snapshots

Test multiple viewport sizes:

```tsx
export const Responsive: Story = {
  parameters: {
    chromatic: {
      viewports: [375, 768, 1440], // Mobile, tablet, desktop
    },
  },
};
```

## Cost Awareness

Chromatic's free tier includes 5,000 snapshots per month. To reduce usage:

- Use `onlyChanged: true` in CI (only test changed stories)
- Skip decorative/static stories with `disableSnapshot: true`
- Limit responsive viewport testing to key components
- Use `--only-story-names` to test specific stories locally

## NPM Scripts

```json
{
  "scripts": {
    "chromatic": "chromatic --exit-zero-on-changes",
    "chromatic:ci": "chromatic --exit-zero-on-changes --only-changed"
  }
}
```
