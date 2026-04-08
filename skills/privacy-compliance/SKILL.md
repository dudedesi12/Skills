---
name: privacy-compliance
description: "Use this skill whenever the user mentions privacy, GDPR, data protection, PII, personally identifiable information, Australian Privacy Act, APPs, consent, cookie consent, cookie banner, data retention, data deletion, right to be forgotten, right to erasure, data export, data portability, encryption at rest, audit log, audit trail, access log, data classification, sensitive data, personal data, immigration data, passport, visa status, privacy policy, terms of service, compliance, data breach, breach notification, anonymization, pseudonymization, data minimization, 'how do I handle user data', 'is my app compliant', 'before launch privacy', or ANY data privacy/protection task — even if they don't explicitly say 'privacy'. Immigration data is some of the most sensitive PII. This skill is not optional."
---

# Privacy Compliance

You are handling immigration data — visa status, passport details, skills assessments, personal circumstances. This is some of the most sensitive PII that exists. Australian Privacy Act compliance is mandatory. GDPR applies if you have any users from the EU. This skill covers what you need to implement in code.

## Stack Context

- Next.js App Router (TypeScript)
- Supabase (PostgreSQL, Auth, Storage)
- Tailwind CSS
- Vercel (deployment)
- Gemini API (for AI features — never OpenAI)

## 1. Australian Privacy Act Basics

The Privacy Act 1988 has 13 Australian Privacy Principles (APPs). The ones that directly affect your code:

| APP | Requirement | What You Build |
|-----|-------------|----------------|
| APP 1 | Open and transparent management of personal info | Privacy policy page, data practices disclosure |
| APP 3 | Collection of solicited personal info | Only collect what you need (data minimization) |
| APP 5 | Notification of collection | Tell users what you collect and why at point of collection |
| APP 6 | Use or disclosure | Only use data for stated purposes |
| APP 8 | Cross-border disclosure | Warn users if data goes offshore (Vercel, Supabase regions) |
| APP 11 | Security of personal info | Encrypt, access control, audit logging |
| APP 12 | Access to personal info | Users can request their data (export) |
| APP 13 | Correction of personal info | Users can correct their data |

See `references/australian-privacy-act.md` for all 13 APPs mapped to code actions.

## 2. GDPR Overlap

If any user is in the EU, GDPR applies regardless of where your business is based.

**Key GDPR rights that overlap with Australian law:**

| Right | GDPR Article | Implementation |
|-------|-------------|----------------|
| Right to access | Art. 15 | Data export endpoint |
| Right to rectification | Art. 16 | Edit profile/data pages |
| Right to erasure | Art. 17 | Account deletion flow |
| Right to data portability | Art. 20 | JSON/CSV export |
| Right to restrict processing | Art. 18 | Pause account feature |
| Right to be informed | Art. 13-14 | Privacy policy + consent UI |

**Lawful basis for processing** — You need at least one for each data processing activity:

```typescript
// types/privacy.types.ts
export type LawfulBasis =
  | "consent"           // User explicitly agreed (marketing emails)
  | "contract"          // Necessary for the service (visa assessment data)
  | "legal_obligation"  // Required by law (tax records)
  | "legitimate_interest"; // Business need that doesn't override user rights
```

See `references/gdpr-requirements.md` for the full compliance checklist.

## 3. Data Classification

Classify every piece of data you collect. Immigration platforms handle all four tiers:

```typescript
// lib/privacy/data-classification.ts
export type DataTier = "public" | "internal" | "confidential" | "restricted";

export const DATA_CLASSIFICATION: Record<string, { tier: DataTier; description: string }> = {
  // PUBLIC — No protection needed
  full_name: { tier: "public", description: "User's display name" },
  avatar_url: { tier: "public", description: "Profile photo URL" },

  // INTERNAL — Basic access control
  email: { tier: "internal", description: "Login email" },
  phone: { tier: "internal", description: "Contact number" },
  created_at: { tier: "internal", description: "Account creation date" },

  // CONFIDENTIAL — Encrypted, audit logged
  date_of_birth: { tier: "confidential", description: "DOB for age verification" },
  nationality: { tier: "confidential", description: "Country of citizenship" },
  occupation: { tier: "confidential", description: "Current job title" },
  skills_assessment: { tier: "confidential", description: "Skills assessment results" },
  english_test_score: { tier: "confidential", description: "IELTS/PTE scores" },
  points_calculation: { tier: "confidential", description: "Points test results" },

  // RESTRICTED — Encrypted, audit logged, access controlled, retention limited
  passport_number: { tier: "restricted", description: "Passport number" },
  visa_status: { tier: "restricted", description: "Current visa subclass and status" },
  visa_application_id: { tier: "restricted", description: "Immigration application reference" },
  health_declaration: { tier: "restricted", description: "Health examination results" },
  police_clearance: { tier: "restricted", description: "Criminal background check" },
  financial_evidence: { tier: "restricted", description: "Bank statements, income proof" },
};
```

**Rules by tier:**

| Tier | Encrypt at Rest | Audit Log Access | Retention Limit | Export Included |
|------|----------------|-----------------|-----------------|-----------------|
| Public | No | No | None | Yes |
| Internal | No | No | Account lifetime | Yes |
| Confidential | Yes | Yes | Account + 2 years | Yes |
| Restricted | Yes | Yes | Purpose + 1 year | On request only |

## 4. Consent Management

### Cookie Consent Banner

```tsx
// components/cookie-consent.tsx
"use client";
import { useState, useEffect } from "react";

interface ConsentPreferences {
  necessary: true; // Always true, can't be disabled
  analytics: boolean;
  marketing: boolean;
}

const CONSENT_KEY = "cookie-consent";

export function CookieConsent() {
  const [show, setShow] = useState(false);
  const [preferences, setPreferences] = useState<ConsentPreferences>({
    necessary: true,
    analytics: false,
    marketing: false,
  });

  useEffect(() => {
    const stored = localStorage.getItem(CONSENT_KEY);
    if (!stored) setShow(true);
    else setPreferences(JSON.parse(stored));
  }, []);

  function savePreferences(prefs: ConsentPreferences) {
    localStorage.setItem(CONSENT_KEY, JSON.stringify(prefs));
    setPreferences(prefs);
    setShow(false);
    // Fire event so analytics can initialize
    window.dispatchEvent(new CustomEvent("consent-updated", { detail: prefs }));
  }

  if (!show) return null;

  return (
    <div className="fixed bottom-0 left-0 right-0 z-50 border-t bg-white p-4 shadow-lg">
      <div className="mx-auto max-w-4xl">
        <p className="text-sm text-gray-700">
          We use cookies to improve your experience. You can choose which cookies to allow.
        </p>
        <div className="mt-3 flex flex-wrap gap-4">
          <label className="flex items-center gap-2 text-sm">
            <input type="checkbox" checked disabled className="rounded" />
            Necessary (required)
          </label>
          <label className="flex items-center gap-2 text-sm">
            <input
              type="checkbox"
              checked={preferences.analytics}
              onChange={(e) => setPreferences({ ...preferences, analytics: e.target.checked })}
              className="rounded"
            />
            Analytics
          </label>
          <label className="flex items-center gap-2 text-sm">
            <input
              type="checkbox"
              checked={preferences.marketing}
              onChange={(e) => setPreferences({ ...preferences, marketing: e.target.checked })}
              className="rounded"
            />
            Marketing
          </label>
        </div>
        <div className="mt-3 flex gap-3">
          <button
            onClick={() => savePreferences({ necessary: true, analytics: true, marketing: true })}
            className="rounded bg-blue-600 px-4 py-2 text-sm text-white hover:bg-blue-700"
          >
            Accept All
          </button>
          <button
            onClick={() => savePreferences(preferences)}
            className="rounded border px-4 py-2 text-sm hover:bg-gray-50"
          >
            Save Preferences
          </button>
          <button
            onClick={() => savePreferences({ necessary: true, analytics: false, marketing: false })}
            className="rounded border px-4 py-2 text-sm hover:bg-gray-50"
          >
            Reject Optional
          </button>
        </div>
      </div>
    </div>
  );
}
```

### Purpose-Specific Consent for Data Collection

```typescript
// lib/privacy/consent.ts
import { createClient } from "@/lib/supabase/server";

export type ConsentPurpose =
  | "visa_assessment"
  | "marketing_emails"
  | "data_sharing_partners"
  | "ai_processing"
  | "analytics";

export async function recordConsent(
  userId: string,
  purpose: ConsentPurpose,
  granted: boolean
) {
  const supabase = await createClient();
  await supabase.from("user_consents").upsert({
    user_id: userId,
    purpose,
    granted,
    granted_at: granted ? new Date().toISOString() : null,
    revoked_at: granted ? null : new Date().toISOString(),
    ip_address: null, // Set from request headers in the API route
  }, { onConflict: "user_id,purpose" });
}

export async function hasConsent(userId: string, purpose: ConsentPurpose): Promise<boolean> {
  const supabase = await createClient();
  const { data } = await supabase
    .from("user_consents")
    .select("granted")
    .eq("user_id", userId)
    .eq("purpose", purpose)
    .single();
  return data?.granted === true;
}
```

```sql
-- supabase/migrations/xxx_user_consents.sql
CREATE TABLE user_consents (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id uuid REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  purpose text NOT NULL,
  granted boolean NOT NULL DEFAULT false,
  granted_at timestamptz,
  revoked_at timestamptz,
  ip_address inet,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  UNIQUE(user_id, purpose)
);

ALTER TABLE user_consents ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own consents"
  ON user_consents FOR SELECT
  USING (auth.uid() = user_id);

CREATE POLICY "Users can update own consents"
  ON user_consents FOR ALL
  USING (auth.uid() = user_id);
```

## 5. Right to Deletion

Users must be able to delete their account and all associated data. This is required by both the Australian Privacy Act (APP 13) and GDPR (Art. 17).

```typescript
// app/api/user/delete-account/route.ts
import { NextRequest, NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: NextRequest) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const body = await request.json();
  if (body.confirmation !== "DELETE MY ACCOUNT") {
    return NextResponse.json({ error: "Confirmation text required" }, { status: 400 });
  }

  // 1. Export user data first (for audit trail)
  const exportData = await exportUserData(user.id);

  // 2. Store anonymized deletion record
  const adminSupabase = createAdminClient();
  await adminSupabase.from("deletion_records").insert({
    anonymized_id: crypto.randomUUID(),
    deleted_at: new Date().toISOString(),
    data_categories: Object.keys(exportData),
    reason: body.reason ?? "user_requested",
  });

  // 3. Delete user data from all tables (cascade handles most)
  await adminSupabase.from("profiles").delete().eq("id", user.id);
  await adminSupabase.from("user_consents").delete().eq("user_id", user.id);
  await adminSupabase.from("audit_logs").delete().eq("user_id", user.id);

  // 4. Delete files from storage
  const { data: files } = await adminSupabase.storage
    .from("user-documents")
    .list(user.id);
  if (files?.length) {
    await adminSupabase.storage
      .from("user-documents")
      .remove(files.map((f) => `${user.id}/${f.name}`));
  }

  // 5. Delete the auth user (this invalidates all sessions)
  await adminSupabase.auth.admin.deleteUser(user.id);

  return NextResponse.json({ success: true });
}
```

### Data Export Endpoint

```typescript
// app/api/user/export-data/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const exportData = await exportUserData(user.id);

  return new NextResponse(JSON.stringify(exportData, null, 2), {
    headers: {
      "Content-Type": "application/json",
      "Content-Disposition": `attachment; filename="my-data-${new Date().toISOString().split("T")[0]}.json"`,
    },
  });
}

async function exportUserData(userId: string) {
  const supabase = await createClient();

  const [profile, consents, assessments, documents] = await Promise.all([
    supabase.from("profiles").select("*").eq("id", userId).single(),
    supabase.from("user_consents").select("*").eq("user_id", userId),
    supabase.from("visa_assessments").select("*").eq("user_id", userId),
    supabase.from("user_documents").select("name, type, created_at").eq("user_id", userId),
  ]);

  return {
    exported_at: new Date().toISOString(),
    profile: profile.data,
    consents: consents.data,
    assessments: assessments.data,
    documents: documents.data,
  };
}
```

## 6. PII Encryption at Rest

Supabase uses PostgreSQL's `pgcrypto` extension for column-level encryption. Use this for restricted-tier data.

```sql
-- supabase/migrations/xxx_enable_encryption.sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt sensitive columns
ALTER TABLE profiles
  ADD COLUMN passport_number_encrypted bytea,
  ADD COLUMN date_of_birth_encrypted bytea;

-- Encryption function
CREATE OR REPLACE FUNCTION encrypt_pii(plaintext text)
RETURNS bytea
LANGUAGE sql
IMMUTABLE
AS $$
  SELECT pgp_sym_encrypt(plaintext, current_setting('app.encryption_key'))
$$;

-- Decryption function
CREATE OR REPLACE FUNCTION decrypt_pii(ciphertext bytea)
RETURNS text
LANGUAGE sql
STABLE
AS $$
  SELECT pgp_sym_decrypt(ciphertext, current_setting('app.encryption_key'))
$$;
```

```typescript
// lib/privacy/encryption.ts
import { createClient } from "@/lib/supabase/server";

export async function storeEncryptedField(
  table: string,
  id: string,
  column: string,
  value: string
) {
  const supabase = await createClient();
  const { error } = await supabase.rpc("encrypt_and_store", {
    p_table: table,
    p_id: id,
    p_column: column,
    p_value: value,
  });
  if (error) throw new Error(`Encryption failed: ${error.message}`);
}
```

**Important:** Store the encryption key in Supabase Vault or as a Vercel environment variable. Never hardcode it.

## 7. Audit Logging

Track who accessed sensitive data and when. Required by APP 11 and GDPR Art. 30.

```sql
-- supabase/migrations/xxx_audit_logs.sql
CREATE TABLE audit_logs (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id uuid REFERENCES auth.users(id) ON DELETE SET NULL,
  action text NOT NULL,
  resource_type text NOT NULL,
  resource_id text,
  details jsonb DEFAULT '{}',
  ip_address inet,
  user_agent text,
  created_at timestamptz DEFAULT now()
);

-- Index for querying by user and time
CREATE INDEX idx_audit_logs_user_created ON audit_logs(user_id, created_at DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);

-- RLS: Only admins can read audit logs
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Admins can read audit logs"
  ON audit_logs FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE profiles.id = auth.uid()
      AND profiles.role = 'admin'
    )
  );

-- Anyone can insert (logging their own actions)
CREATE POLICY "Users can create audit entries"
  ON audit_logs FOR INSERT
  WITH CHECK (auth.uid() = user_id);
```

```typescript
// lib/privacy/audit-log.ts
import { createClient } from "@/lib/supabase/server";
import { headers } from "next/headers";

type AuditAction =
  | "view"
  | "create"
  | "update"
  | "delete"
  | "export"
  | "login"
  | "logout"
  | "consent_granted"
  | "consent_revoked";

export async function logAuditEvent(
  userId: string,
  action: AuditAction,
  resourceType: string,
  resourceId?: string,
  details?: Record<string, unknown>
) {
  const supabase = await createClient();
  const headersList = await headers();

  await supabase.from("audit_logs").insert({
    user_id: userId,
    action,
    resource_type: resourceType,
    resource_id: resourceId,
    details: details ?? {},
    ip_address: headersList.get("x-forwarded-for")?.split(",")[0] ?? null,
    user_agent: headersList.get("user-agent") ?? null,
  });
}

// Usage in any API route or server action:
// await logAuditEvent(user.id, "view", "visa_assessment", assessmentId);
// await logAuditEvent(user.id, "export", "user_data");
// await logAuditEvent(user.id, "delete", "account");
```

## 8. Data Retention Policies

Don't keep data longer than necessary. Set up automatic cleanup.

```sql
-- supabase/migrations/xxx_data_retention.sql

-- Function to clean up expired data
CREATE OR REPLACE FUNCTION cleanup_expired_data()
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  -- Delete unverified accounts older than 30 days
  DELETE FROM auth.users
  WHERE email_confirmed_at IS NULL
  AND created_at < now() - interval '30 days';

  -- Anonymize audit logs older than 2 years
  UPDATE audit_logs
  SET user_id = NULL, ip_address = NULL, user_agent = NULL,
      details = jsonb_set(details, '{anonymized}', 'true')
  WHERE created_at < now() - interval '2 years'
  AND user_id IS NOT NULL;

  -- Delete expired visa assessment drafts
  DELETE FROM visa_assessments
  WHERE status = 'draft'
  AND updated_at < now() - interval '90 days';

  -- Delete old session data
  DELETE FROM user_sessions
  WHERE last_active_at < now() - interval '90 days';
END;
$$;

-- Run cleanup daily at 3 AM AEST via pg_cron
SELECT cron.schedule(
  'daily-data-cleanup',
  '0 17 * * *',  -- 3 AM AEST = 5 PM UTC
  $$SELECT cleanup_expired_data()$$
);
```

## 9. Privacy Impact Assessment Template

Before collecting any new category of personal data, complete a Privacy Impact Assessment (PIA):

```markdown
## Privacy Impact Assessment

### 1. Data Being Collected
- What: [describe the data]
- Why: [purpose for collection]
- Lawful basis: [consent / contract / legal obligation / legitimate interest]

### 2. Data Flow
- Collected from: [form / API / third party]
- Stored in: [Supabase table name]
- Processed by: [which services — Gemini, Vercel functions, etc.]
- Shared with: [any third parties]
- Crosses borders: [yes/no — Supabase region, Vercel edge]

### 3. Risk Assessment
- What if this data leaked? [impact: low / medium / high / critical]
- What's the minimum data needed? [data minimization check]
- How long do we need it? [retention period]
- Can we anonymize/pseudonymize it? [yes/no]

### 4. Safeguards
- [ ] Data encrypted at rest
- [ ] Access controlled by RLS
- [ ] Audit logging enabled
- [ ] Retention policy set
- [ ] Consent collected (if consent is the lawful basis)
- [ ] Privacy policy updated
- [ ] Data export includes this data
- [ ] Deletion flow removes this data
```

## 10. Cross-Border Data Disclosure

Australian Privacy Act APP 8 requires disclosure when data goes offshore.

```typescript
// Your data crosses borders to:
const DATA_LOCATIONS = {
  supabase: "AWS us-east-1 (Virginia, USA)", // Check your Supabase project region
  vercel: "Global edge network + iad1 (Virginia, USA)",
  geminiApi: "Google Cloud (USA)",
  stripe: "USA",
  resend: "USA",
  upstash: "Global edge",
};

// Add to your privacy policy:
// "Your data is processed by services located in the United States.
//  By using our service, you consent to this cross-border transfer.
//  All transfers are protected by industry-standard encryption."
```

## Rules

1. **Never store PII you don't need** — If a form field isn't essential, don't collect it.
2. **Encrypt restricted-tier data** — Passport numbers, visa details, health records must be encrypted at column level.
3. **Log every access to sensitive data** — If someone views, edits, or exports confidential/restricted data, log it.
4. **Implement deletion completely** — Account deletion must remove data from ALL tables, storage, and auth.
5. **Set retention limits** — Every data category should have a defined retention period.
6. **Get explicit consent** — For anything beyond "necessary for the service" (marketing, AI processing, analytics).
7. **Allow data export** — Users must be able to download all their data in a standard format (JSON).
8. **Update privacy policy** — Every time you add a new data collection point or third-party service.
9. **Disclose cross-border transfers** — Tell users their data goes to US-based services.
10. **Review quarterly** — Run through the privacy checklist every 3 months.

See `references/` for complete guides:
- `australian-privacy-act.md` — All 13 APPs mapped to developer actions
- `gdpr-requirements.md` — GDPR compliance checklist
- `data-handling-patterns.md` — Encryption, anonymization, and pseudonymization code
