# Translation Patterns Reference

## JSON File Structure

Organize translations by feature namespace:

```
messages/
  en/
    common.json       ← shared across pages (buttons, navigation, errors)
    auth.json          ← login, signup, password reset
    dashboard.json     ← dashboard page
    assessment.json    ← visa assessment
    settings.json      ← user settings
  zh/
    common.json
    auth.json
    dashboard.json
    assessment.json
    settings.json
```

## Basic Key-Value

```json
// messages/en/common.json
{
  "buttons": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "confirm": "Confirm",
    "back": "Back",
    "next": "Next",
    "submit": "Submit"
  },
  "navigation": {
    "home": "Home",
    "dashboard": "Dashboard",
    "assessment": "Assessment",
    "settings": "Settings",
    "logout": "Log out"
  },
  "errors": {
    "generic": "Something went wrong. Please try again.",
    "notFound": "Page not found",
    "unauthorized": "Please log in to continue",
    "networkError": "Network error. Check your connection."
  }
}
```

## Interpolation (Variables)

```json
{
  "greeting": "Hello, {name}!",
  "resultsCount": "Showing {count} results",
  "lastUpdated": "Last updated {date}",
  "pointsScore": "Your points score is {score} out of {max}"
}
```

```tsx
const t = useTranslations("dashboard");
t("greeting", { name: user.fullName });
t("pointsScore", { score: 80, max: 100 });
```

## Pluralization

```json
{
  "results": {
    "zero": "No results found",
    "one": "1 result found",
    "other": "{count} results found"
  },
  "documents": {
    "zero": "No documents uploaded",
    "one": "1 document uploaded",
    "other": "{count} documents uploaded"
  }
}
```

```tsx
t("results", { count: results.length });
// count=0 → "No results found"
// count=1 → "1 result found"
// count=5 → "5 results found"
```

### Language-Specific Plural Rules

Different languages have different plural categories:

| Language | Categories | Example |
|----------|-----------|---------|
| English | one, other | 1 item, 2 items |
| Chinese | other (no plural) | 1 个项目, 2 个项目 |
| Arabic | zero, one, two, few, many, other | Complex rules |
| Hindi | one, other | 1 परिणाम, 2 परिणाम |
| Russian | one, few, many, other | 1 результат, 2 результата, 5 результатов |

For Arabic:
```json
{
  "results": {
    "zero": "لا توجد نتائج",
    "one": "نتيجة واحدة",
    "two": "نتيجتان",
    "few": "{count} نتائج",
    "many": "{count} نتيجة",
    "other": "{count} نتيجة"
  }
}
```

## Rich Text (Bold, Links, etc.)

```json
{
  "terms": "By signing up, you agree to our <terms>Terms of Service</terms> and <privacy>Privacy Policy</privacy>.",
  "important": "Your visa application is <bold>currently being processed</bold>."
}
```

```tsx
t.rich("terms", {
  terms: (chunks) => <a href="/terms" className="underline">{chunks}</a>,
  privacy: (chunks) => <a href="/privacy" className="underline">{chunks}</a>,
});

t.rich("important", {
  bold: (chunks) => <strong>{chunks}</strong>,
});
```

## Nested Namespaces

```json
{
  "visa": {
    "subclass189": {
      "title": "Skilled Independent Visa (subclass 189)",
      "description": "A points-tested visa for skilled workers who are not sponsored.",
      "requirements": {
        "age": "Under 45 years of age",
        "english": "Competent English (IELTS 6 or equivalent)",
        "skills": "Occupation on the Medium and Long-term Strategic Skills List"
      }
    },
    "subclass190": {
      "title": "Skilled Nominated Visa (subclass 190)",
      "description": "A points-tested visa for skilled workers nominated by a state or territory."
    }
  }
}
```

```tsx
const t = useTranslations("assessment.visa.subclass189");
t("title"); // "Skilled Independent Visa (subclass 189)"
t("requirements.age"); // "Under 45 years of age"
```

## Missing Translation Fallbacks

```typescript
// i18n/request.ts
import { getRequestConfig } from "next-intl/server";

export default getRequestConfig(async ({ requestLocale }) => {
  const locale = await requestLocale;

  // Load the requested locale's messages
  const userMessages = (await import(`../../messages/${locale}/common.json`)).default;

  // Load English as fallback
  const fallbackMessages = (await import("../../messages/en/common.json")).default;

  return {
    locale,
    messages: {
      // Spread fallback first, then override with user's locale
      ...fallbackMessages,
      ...userMessages,
    },
    // Show a warning in dev when a translation is missing
    onError(error) {
      if (process.env.NODE_ENV === "development") {
        console.warn(`Missing translation: ${error.message}`);
      }
    },
    getMessageFallback({ namespace, key }) {
      return `[${namespace}.${key}]`; // Shows [common.buttons.save] if missing
    },
  };
});
```

## Finding Missing Translations

Script to check for missing keys between locales:

```typescript
// scripts/check-translations.ts
import fs from "fs";
import path from "path";

const MESSAGES_DIR = path.join(process.cwd(), "messages");
const DEFAULT_LOCALE = "en";

function flattenKeys(obj: Record<string, unknown>, prefix = ""): string[] {
  return Object.entries(obj).flatMap(([key, value]) => {
    const fullKey = prefix ? `${prefix}.${key}` : key;
    if (typeof value === "object" && value !== null && !Array.isArray(value)) {
      return flattenKeys(value as Record<string, unknown>, fullKey);
    }
    return [fullKey];
  });
}

const locales = fs.readdirSync(MESSAGES_DIR).filter((f) =>
  fs.statSync(path.join(MESSAGES_DIR, f)).isDirectory()
);

const defaultDir = path.join(MESSAGES_DIR, DEFAULT_LOCALE);
const namespaces = fs.readdirSync(defaultDir).filter((f) => f.endsWith(".json"));

let missingCount = 0;

for (const ns of namespaces) {
  const defaultKeys = flattenKeys(
    JSON.parse(fs.readFileSync(path.join(defaultDir, ns), "utf-8"))
  );

  for (const locale of locales) {
    if (locale === DEFAULT_LOCALE) continue;
    const localePath = path.join(MESSAGES_DIR, locale, ns);

    if (!fs.existsSync(localePath)) {
      console.log(`MISSING FILE: ${locale}/${ns}`);
      missingCount += defaultKeys.length;
      continue;
    }

    const localeKeys = flattenKeys(
      JSON.parse(fs.readFileSync(localePath, "utf-8"))
    );

    for (const key of defaultKeys) {
      if (!localeKeys.includes(key)) {
        console.log(`MISSING KEY: ${locale}/${ns} → ${key}`);
        missingCount++;
      }
    }
  }
}

console.log(`\nTotal missing: ${missingCount}`);
process.exit(missingCount > 0 ? 1 : 0);
```

Run with: `npx tsx scripts/check-translations.ts`
