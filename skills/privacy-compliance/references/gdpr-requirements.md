# GDPR Compliance Checklist for Developers

If any user is located in the EU, GDPR applies. This checklist covers the technical requirements.

## Lawful Basis Mapping

Map every data processing activity to a lawful basis:

| Data Processing | Lawful Basis | Notes |
|----------------|-------------|-------|
| Account creation (email, name) | Contract | Necessary to provide the service |
| Visa eligibility assessment | Contract | Core service functionality |
| Login session management | Contract | Necessary for authenticated access |
| Marketing emails | Consent | Must be opt-in, easy opt-out |
| Analytics tracking | Legitimate interest / Consent | Use cookie consent for analytics |
| AI-powered assessments | Consent | Get explicit consent before sending data to Gemini |
| Payment processing | Contract + Legal obligation | Stripe for payments, tax records legally required |
| Error logging (Sentry) | Legitimate interest | Minimal PII, necessary for service quality |

## Data Subject Rights Implementation

### Right to Access (Art. 15)

Users can request all data you hold about them.

```typescript
// Already implemented in the data export endpoint
// Must respond within 30 days
// Must be in a machine-readable format (JSON, CSV)
// Must be free of charge for the first request
```

### Right to Rectification (Art. 16)

Users can correct inaccurate data.

```typescript
// Implementation: editable profile page
// Log all corrections in audit trail
// Update all derived data when source data changes
```

### Right to Erasure (Art. 17)

Users can request deletion of their data.

**Exceptions — you can refuse deletion for:**
- Legal obligations (tax records — keep for 7 years)
- Legal claims (ongoing disputes)
- Public interest

```typescript
// lib/privacy/deletion.ts
export const DELETION_EXCEPTIONS: Record<string, { reason: string; retainUntil: string }> = {
  stripe_invoices: {
    reason: "Legal obligation — tax records must be retained for 7 years",
    retainUntil: "7 years from invoice date",
  },
  audit_logs: {
    reason: "Legitimate interest — security audit trail",
    retainUntil: "2 years, then anonymized",
  },
};
```

### Right to Data Portability (Art. 20)

Export data in a structured, commonly used, machine-readable format.

```typescript
// Supported export formats
export async function exportUserDataFormatted(userId: string, format: "json" | "csv") {
  const data = await exportUserData(userId);

  if (format === "json") {
    return JSON.stringify(data, null, 2);
  }

  // CSV export for spreadsheet users
  const rows = [
    ["Category", "Field", "Value"],
    ...Object.entries(data.profile ?? {}).map(([key, value]) =>
      ["Profile", key, String(value)]
    ),
    ...Object.entries(data.consents ?? {}).map(([key, value]) =>
      ["Consents", key, String(value)]
    ),
  ];

  return rows.map((row) => row.map((cell) => `"${cell}"`).join(",")).join("\n");
}
```

### Right to Restrict Processing (Art. 18)

Users can request you stop processing their data while a dispute is resolved.

```typescript
// Add a processing_restricted flag to profiles
// When true: don't send to AI, don't include in analytics, don't send emails
// Still allow login and data access

export async function restrictProcessing(userId: string) {
  const supabase = await createClient();
  await supabase
    .from("profiles")
    .update({ processing_restricted: true })
    .eq("id", userId);

  await logAuditEvent(userId, "update", "processing_restriction", userId, {
    restricted: true,
  });
}
```

### Right to Object (Art. 21)

Users can object to processing based on legitimate interest.

```typescript
// For direct marketing: must stop immediately
// For other processing: assess if legitimate interest overrides

async function handleObjection(userId: string, processingActivity: string) {
  if (processingActivity === "marketing") {
    await recordConsent(userId, "marketing_emails", false);
    return { accepted: true };
  }

  // For other activities, log and review
  await logAuditEvent(userId, "update", "processing_objection", userId, {
    activity: processingActivity,
    status: "pending_review",
  });
  return { accepted: false, message: "Your objection has been logged and will be reviewed within 30 days." };
}
```

## Consent Requirements

GDPR consent must be:

1. **Freely given** — No penalty for refusing (can still use the service)
2. **Specific** — Separate consent for each purpose
3. **Informed** — Clear explanation of what they're consenting to
4. **Unambiguous** — Affirmative action (no pre-ticked boxes)

```tsx
// GOOD: Separate, unchecked consent checkboxes
<label className="flex items-start gap-2">
  <input type="checkbox" name="consent_ai" className="mt-1" />
  <span className="text-sm">
    I agree to my profile data being processed by AI to generate visa
    recommendations.{" "}
    <a href="/privacy#ai-processing" className="underline">Learn more</a>
  </span>
</label>

<label className="flex items-start gap-2">
  <input type="checkbox" name="consent_marketing" className="mt-1" />
  <span className="text-sm">
    I'd like to receive tips and updates about visa pathways via email.
  </span>
</label>

// BAD: Bundled consent
<label>
  <input type="checkbox" defaultChecked />
  I agree to the terms of service, privacy policy, and marketing communications.
</label>
```

## Records of Processing Activities (Art. 30)

Maintain a register of all data processing activities.

```typescript
// lib/privacy/processing-register.ts
export const PROCESSING_REGISTER = [
  {
    activity: "User Registration",
    dataCategories: ["email", "password_hash", "full_name"],
    purpose: "Account creation and authentication",
    lawfulBasis: "contract" as const,
    recipients: ["Supabase (database)", "Resend (welcome email)"],
    retention: "Until account deletion",
    crossBorder: ["USA (Supabase AWS)", "USA (Resend)"],
  },
  {
    activity: "Visa Eligibility Assessment",
    dataCategories: ["nationality", "occupation", "qualifications", "english_score", "work_experience"],
    purpose: "Calculate visa eligibility and points",
    lawfulBasis: "contract" as const,
    recipients: ["Supabase (storage)", "Google Gemini API (AI analysis)"],
    retention: "Until account deletion or 2 years after last assessment",
    crossBorder: ["USA (Supabase AWS)", "USA (Google Cloud)"],
  },
  {
    activity: "Analytics Tracking",
    dataCategories: ["page_views", "anonymized_user_id", "device_type"],
    purpose: "Improve service quality and user experience",
    lawfulBasis: "consent" as const,
    recipients: ["Vercel Analytics / PostHog"],
    retention: "12 months",
    crossBorder: ["USA (Vercel)"],
  },
  {
    activity: "Payment Processing",
    dataCategories: ["email", "payment_method", "billing_address"],
    purpose: "Process subscription payments",
    lawfulBasis: "contract" as const,
    recipients: ["Stripe"],
    retention: "7 years (tax obligation)",
    crossBorder: ["USA (Stripe)"],
  },
];
```

## Data Protection Impact Assessment (DPIA)

Required when processing is likely to result in a high risk to individuals. Immigration data qualifies.

A DPIA should cover:
1. Description of the processing and its purposes
2. Assessment of necessity and proportionality
3. Assessment of risks to individuals
4. Measures to address those risks

See the Privacy Impact Assessment template in the main SKILL.md.

## Breach Notification (Art. 33-34)

- Notify supervisory authority within **72 hours**
- Notify affected individuals "without undue delay" if high risk
- Document all breaches (even ones you don't report)

## Children's Data (Art. 8)

If your platform could be used by anyone under 16:
- Need parental consent
- Privacy notice must be written in clear, plain language
- Consider age verification

For an immigration platform: unlikely to have users under 16, but if you add dependent visa tracking, be aware.
