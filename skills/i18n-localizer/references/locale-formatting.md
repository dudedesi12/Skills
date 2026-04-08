# Locale Formatting Reference

Date, number, currency, phone, and address formats for the immigration platform's key markets.

## Locale Code Reference

| Market | Locale Code | Language | Script Direction |
|--------|------------|----------|-----------------|
| Australia | en-AU | English | LTR |
| United Kingdom | en-GB | English | LTR |
| China | zh-CN | Simplified Chinese | LTR |
| India | hi-IN / en-IN | Hindi / English | LTR |
| Philippines | fil-PH / en-PH | Filipino / English | LTR |
| Pakistan | ur-PK | Urdu | RTL |
| United States | en-US | English | LTR |
| Saudi Arabia | ar-SA | Arabic | RTL |

## Date Formats

```typescript
// lib/i18n/format-date.ts
export function formatDateByLocale(date: Date, locale: string): string {
  return new Intl.DateTimeFormat(locale, {
    day: "numeric",
    month: "long",
    year: "numeric",
  }).format(date);
}
```

| Locale | Format | Example |
|--------|--------|---------|
| en-AU | DD Month YYYY | 8 April 2026 |
| en-GB | DD Month YYYY | 8 April 2026 |
| en-US | Month DD, YYYY | April 8, 2026 |
| zh-CN | YYYY年M月D日 | 2026年4月8日 |
| hi-IN | DD Month YYYY | 8 अप्रैल 2026 |
| ar-SA | DD Month YYYY (Hijri) | ١٠ شوال ١٤٤٧ هـ |
| fil-PH | Month DD, YYYY | Abril 8, 2026 |

### Short Date

```typescript
export function formatShortDate(date: Date, locale: string): string {
  return new Intl.DateTimeFormat(locale, {
    day: "2-digit",
    month: "2-digit",
    year: "numeric",
  }).format(date);
}
```

| Locale | Format | Example |
|--------|--------|---------|
| en-AU | DD/MM/YYYY | 08/04/2026 |
| en-US | MM/DD/YYYY | 04/08/2026 |
| zh-CN | YYYY/MM/DD | 2026/04/08 |

### Relative Time

```typescript
export function formatRelativeTime(date: Date, locale: string): string {
  const rtf = new Intl.RelativeTimeFormat(locale, { numeric: "auto" });
  const diffMs = date.getTime() - Date.now();
  const diffDays = Math.round(diffMs / (1000 * 60 * 60 * 24));

  if (Math.abs(diffDays) < 1) {
    const diffHours = Math.round(diffMs / (1000 * 60 * 60));
    return rtf.format(diffHours, "hour");
  }
  if (Math.abs(diffDays) < 30) return rtf.format(diffDays, "day");
  const diffMonths = Math.round(diffDays / 30);
  return rtf.format(diffMonths, "month");
}
```

| Locale | "2 days ago" | "in 3 months" |
|--------|-------------|--------------|
| en-AU | 2 days ago | in 3 months |
| zh-CN | 2天前 | 3个月后 |
| hi-IN | 2 दिन पहले | 3 महीने में |
| ar-SA | قبل يومين | خلال 3 أشهر |

## Number Formats

```typescript
export function formatNumber(value: number, locale: string): string {
  return new Intl.NumberFormat(locale).format(value);
}
```

| Locale | Decimal | Grouping | Example (1234567.89) |
|--------|---------|----------|---------------------|
| en-AU | . | , | 1,234,567.89 |
| en-US | . | , | 1,234,567.89 |
| zh-CN | . | , | 1,234,567.89 |
| hi-IN | . | ,  (lakhs) | 12,34,567.89 |
| ar-SA | ٫ | ٬ | ١٬٢٣٤٬٥٦٧٫٨٩ |
| ur-PK | . | , | 1,234,567.89 |

## Currency Formats

```typescript
export function formatCurrency(amount: number, currency: string, locale: string): string {
  return new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
  }).format(amount);
}
```

| Locale | Currency | Code | Example ($1,250.00) |
|--------|----------|------|-------------------|
| en-AU | Australian Dollar | AUD | A$1,250.00 or $1,250.00 |
| en-GB | British Pound | GBP | £1,250.00 |
| en-US | US Dollar | USD | $1,250.00 |
| zh-CN | Chinese Yuan | CNY | ¥1,250.00 |
| hi-IN | Indian Rupee | INR | ₹1,250.00 |
| fil-PH | Philippine Peso | PHP | ₱1,250.00 |
| ur-PK | Pakistani Rupee | PKR | Rs 1,250.00 |
| ar-SA | Saudi Riyal | SAR | ١٬٢٥٠٫٠٠ ر.س |

### Visa Application Fees

Common immigration fees to format:

```typescript
const VISA_FEES: Record<string, { amount: number; currency: string }> = {
  "subclass-189": { amount: 4640, currency: "AUD" },
  "subclass-190": { amount: 4640, currency: "AUD" },
  "subclass-491": { amount: 4640, currency: "AUD" },
  "skills-assessment": { amount: 500, currency: "AUD" },
  "english-test-ielts": { amount: 395, currency: "AUD" },
  "english-test-pte": { amount: 410, currency: "AUD" },
};

// Display in user's preferred currency (approximate)
export function formatVisaFee(feeKey: string, locale: string, displayCurrency?: string): string {
  const fee = VISA_FEES[feeKey];
  if (!fee) return "N/A";

  if (!displayCurrency || displayCurrency === fee.currency) {
    return formatCurrency(fee.amount, fee.currency, locale);
  }

  // Show both: original + converted
  const original = formatCurrency(fee.amount, fee.currency, "en-AU");
  return `${original} (approx.)`;
}
```

## Phone Number Formats

| Country | Format | Example |
|---------|--------|---------|
| Australia | +61 X XXXX XXXX | +61 4 1234 5678 |
| UK | +44 XXXX XXXXXX | +44 7911 123456 |
| China | +86 XXX XXXX XXXX | +86 138 1234 5678 |
| India | +91 XXXXX XXXXX | +91 98765 43210 |
| Philippines | +63 XXX XXX XXXX | +63 917 123 4567 |
| Pakistan | +92 XXX XXXXXXX | +92 300 1234567 |

## Address Formats

```
Australia:
  {name}
  {street}
  {suburb} {state} {postcode}
  AUSTRALIA

UK:
  {name}
  {street}
  {city}
  {county}
  {postcode}
  UNITED KINGDOM

China:
  {postcode}
  {province}{city}{district}
  {street}
  {name}
  中国

India:
  {name}
  {street}
  {city} - {pincode}
  {state}
  INDIA
```

## Using Formatters in next-intl

```tsx
"use client";
import { useFormatter, useLocale } from "next-intl";

export function PriceDisplay({ amount }: { amount: number }) {
  const format = useFormatter();
  const locale = useLocale();

  return (
    <div>
      <p className="text-2xl font-bold">
        {format.number(amount, { style: "currency", currency: "AUD" })}
      </p>
      <p className="text-sm text-gray-500">
        {format.dateTime(new Date(), { dateStyle: "long" })}
      </p>
      <p className="text-sm text-gray-500">
        {format.relativeTime(new Date(Date.now() - 86400000))}
      </p>
    </div>
  );
}
```
