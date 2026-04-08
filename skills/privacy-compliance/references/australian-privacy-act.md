# Australian Privacy Principles — Developer Reference

All 13 APPs mapped to what you need to build.

## APP 1 — Open and Transparent Management

**Requirement:** Have a clearly expressed and up-to-date privacy policy.

**What to build:**
- `/privacy` page with plain-language privacy policy
- Must cover: what you collect, why, who you share with, how to access/correct/delete

```tsx
// app/privacy/page.tsx
export const metadata = {
  title: "Privacy Policy",
  description: "How we collect, use, and protect your personal information",
};

export default function PrivacyPage() {
  return (
    <article className="prose mx-auto max-w-3xl px-4 py-12">
      <h1>Privacy Policy</h1>
      <p className="text-sm text-gray-500">Last updated: {new Date().toLocaleDateString("en-AU")}</p>

      <h2>Information We Collect</h2>
      <p>We collect information you provide directly:</p>
      <ul>
        <li><strong>Account information:</strong> name, email address</li>
        <li><strong>Immigration profile:</strong> nationality, occupation, qualifications, English test scores</li>
        <li><strong>Sensitive documents:</strong> passport details, visa application references (encrypted)</li>
      </ul>

      <h2>How We Use Your Information</h2>
      <ul>
        <li>To provide visa eligibility assessments</li>
        <li>To calculate points test scores</li>
        <li>To send service-related communications</li>
        <li>To improve our service using anonymized analytics</li>
      </ul>

      <h2>Where Your Data Is Stored</h2>
      <p>Your data is processed by services in the United States:</p>
      <ul>
        <li>Supabase (database and authentication)</li>
        <li>Vercel (application hosting)</li>
        <li>Google Gemini API (AI-powered assessments)</li>
      </ul>

      <h2>Your Rights</h2>
      <ul>
        <li><strong>Access:</strong> Request a copy of your data (Settings → Export Data)</li>
        <li><strong>Correction:</strong> Update your information anytime in your profile</li>
        <li><strong>Deletion:</strong> Delete your account and all data (Settings → Delete Account)</li>
        <li><strong>Complaints:</strong> Contact the OAIC at oaic.gov.au</li>
      </ul>

      <h2>Contact</h2>
      <p>Privacy inquiries: privacy@yourdomain.com.au</p>
    </article>
  );
}
```

## APP 2 — Anonymity and Pseudonymity

**Requirement:** Allow users to interact anonymously where practical.

**What to build:**
- Allow browsing public content without an account
- Don't require real names for account registration (email-only signup)
- Visa assessment tools should work with a pseudonym until actual application data is needed

## APP 3 — Collection of Solicited Personal Information

**Requirement:** Only collect information that is reasonably necessary for your functions.

**Data minimization checklist:**
- [ ] Every form field has a clear purpose
- [ ] No "nice to have" fields — only collect what the feature needs
- [ ] Optional fields are clearly marked optional
- [ ] Don't collect nationality, DOB, or passport until needed for a specific assessment

```typescript
// GOOD: Only collect what this specific feature needs
const basicAssessmentSchema = z.object({
  occupation: z.string(),        // Needed for SOL check
  yearsExperience: z.number(),   // Needed for points calc
  englishLevel: z.enum(["superior", "proficient", "competent"]), // Needed for points
});

// BAD: Collecting everything upfront
const overCollectionSchema = z.object({
  occupation: z.string(),
  yearsExperience: z.number(),
  englishLevel: z.string(),
  passportNumber: z.string(),     // Not needed yet
  dateOfBirth: z.string(),        // Not needed yet
  maritalStatus: z.string(),      // Not needed yet
  numberOfChildren: z.number(),   // Not needed yet
});
```

## APP 4 — Dealing with Unsolicited Personal Information

**Requirement:** If you receive personal info you didn't ask for, determine if you could have collected it under APP 3. If not, destroy or de-identify it.

**What to build:**
- If users paste personal info into free-text fields (like a chat or notes), don't store it in plain text
- Sanitize AI prompts to strip obviously sensitive data before sending to Gemini

## APP 5 — Notification of Collection

**Requirement:** Tell users what you're collecting at the point of collection.

```tsx
// Include collection notices on forms
<form>
  <h2>Visa Eligibility Assessment</h2>
  <p className="text-sm text-gray-500">
    We'll use this information to assess your visa eligibility. Your data is
    encrypted and stored securely.{" "}
    <a href="/privacy" className="underline">Privacy Policy</a>
  </p>
  {/* form fields */}
</form>
```

## APP 6 — Use or Disclosure of Personal Information

**Requirement:** Only use data for the purpose it was collected, or a directly related secondary purpose the user would reasonably expect.

**What to build:**
- Purpose tracking on data collection (what was the stated purpose)
- Don't use visa assessment data for marketing unless explicitly consented
- Don't share data with partners without explicit consent

## APP 7 — Direct Marketing

**Requirement:** Users must opt-in to marketing. Must be able to opt-out at any time.

```typescript
// Check consent before sending marketing
async function sendMarketingEmail(userId: string, template: string) {
  const hasMarketingConsent = await hasConsent(userId, "marketing_emails");
  if (!hasMarketingConsent) {
    console.log(`Skipping marketing email for ${userId} — no consent`);
    return;
  }
  // Send email with unsubscribe link
  await resend.emails.send({
    // ... email config
    headers: { "List-Unsubscribe": `<https://yourdomain.com/api/unsubscribe?uid=${userId}>` },
  });
}
```

## APP 8 — Cross-Border Disclosure

**Requirement:** Before disclosing personal info to an overseas recipient, take reasonable steps to ensure they comply with the APPs.

**Your cross-border transfers:**
- Supabase: AWS US-East (check your project settings)
- Vercel: Global CDN + US-East
- Google Gemini API: Google Cloud, USA
- Stripe: USA
- Resend: USA

**What to build:**
- Privacy policy must list these services and their locations
- Terms of service must include consent to cross-border transfer
- Consider using Supabase's Australian region if available

## APP 9 — Adoption, Use, or Disclosure of Government Identifiers

**Requirement:** Don't use government identifiers (passport number, TFN, etc.) as your own database identifiers.

**What to build:**
- Use UUIDs as primary keys, never passport numbers
- Store government IDs encrypted, never in plain text
- Never use government IDs in URLs or logs

## APP 10 — Quality of Personal Information

**Requirement:** Take reasonable steps to ensure data is accurate, complete, and up-to-date.

**What to build:**
- Allow users to edit their profile anytime
- Prompt users to review their data periodically
- Validate data at entry (Zod schemas)

## APP 11 — Security of Personal Information

**Requirement:** Protect personal information from misuse, interference, loss, and unauthorized access.

**Security checklist:**
- [ ] RLS enabled on all tables
- [ ] Sensitive columns encrypted with pgcrypto
- [ ] Auth via Supabase (bcrypt passwords)
- [ ] HTTPS everywhere (Vercel handles this)
- [ ] Security headers set (CSP, HSTS, etc.)
- [ ] API rate limiting
- [ ] File upload validation
- [ ] Service role key server-only

## APP 12 — Access to Personal Information

**Requirement:** Give users access to their personal information on request.

**What to build:**
- Data export endpoint (JSON download)
- Settings page with "Export My Data" button
- Must respond within 30 days (automated = instant)

## APP 13 — Correction of Personal Information

**Requirement:** Correct personal information on request.

**What to build:**
- Editable profile fields
- Ability to update all personal data
- Log corrections in audit trail

## Data Breach Response

If you discover a data breach involving personal information:

1. **Contain** — Immediately stop the breach (rotate keys, disable compromised endpoints)
2. **Assess** — Determine what data was affected and how many users
3. **Notify OAIC** — If serious harm is likely, notify within 30 days via oaic.gov.au
4. **Notify users** — Tell affected users what happened, what data was involved, and what steps to take
5. **Remediate** — Fix the vulnerability and review security measures

```typescript
// lib/privacy/breach-response.ts
export async function initiateBreachResponse(details: {
  discoveredAt: Date;
  description: string;
  affectedTables: string[];
  estimatedUsersAffected: number;
  severity: "low" | "medium" | "high" | "critical";
}) {
  // 1. Log the breach
  await logAuditEvent("system", "data_breach", "security", undefined, details);

  // 2. Alert administrators immediately
  await sendAdminAlert({
    subject: `DATA BREACH: ${details.severity.toUpperCase()}`,
    body: `Breach discovered at ${details.discoveredAt.toISOString()}\n\n${details.description}\n\nAffected: ~${details.estimatedUsersAffected} users`,
    channel: "critical",
  });

  // 3. If critical, rotate all API keys
  if (details.severity === "critical") {
    console.error("CRITICAL BREACH — Rotate all API keys immediately");
  }
}
```
