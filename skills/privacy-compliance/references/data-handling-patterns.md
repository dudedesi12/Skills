# Data Handling Patterns

Code patterns for encryption, anonymization, pseudonymization, and secure data handling.

## Column-Level Encryption with pgcrypto

### Setup

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Store the encryption key in Supabase Vault
SELECT vault.create_secret('encryption_key', 'your-256-bit-key-here');
```

### Encrypt on Insert

```sql
-- Function to encrypt and store a value
CREATE OR REPLACE FUNCTION encrypt_and_store(
  p_table text,
  p_id uuid,
  p_column text,
  p_value text
)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  encryption_key text;
BEGIN
  SELECT decrypted_secret INTO encryption_key
  FROM vault.decrypted_secrets
  WHERE name = 'encryption_key';

  EXECUTE format(
    'UPDATE %I SET %I = pgp_sym_encrypt($1, $2) WHERE id = $3',
    p_table, p_column
  ) USING p_value, encryption_key, p_id;
END;
$$;
```

### Decrypt on Read

```sql
-- Function to decrypt a value
CREATE OR REPLACE FUNCTION decrypt_field(
  p_table text,
  p_id uuid,
  p_column text
)
RETURNS text
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  encryption_key text;
  encrypted_value bytea;
  result text;
BEGIN
  SELECT decrypted_secret INTO encryption_key
  FROM vault.decrypted_secrets
  WHERE name = 'encryption_key';

  EXECUTE format(
    'SELECT %I FROM %I WHERE id = $1',
    p_column, p_table
  ) INTO encrypted_value USING p_id;

  IF encrypted_value IS NULL THEN
    RETURN NULL;
  END IF;

  SELECT pgp_sym_decrypt(encrypted_value, encryption_key) INTO result;
  RETURN result;
END;
$$;
```

### TypeScript Wrapper

```typescript
// lib/privacy/encrypted-fields.ts
import { createClient } from "@/lib/supabase/server";

export async function getDecryptedField(
  table: string,
  id: string,
  column: string
): Promise<string | null> {
  const supabase = await createClient();
  const { data, error } = await supabase.rpc("decrypt_field", {
    p_table: table,
    p_id: id,
    p_column: column,
  });
  if (error) throw new Error(`Decryption failed: ${error.message}`);
  return data;
}

export async function setEncryptedField(
  table: string,
  id: string,
  column: string,
  value: string
): Promise<void> {
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

## Anonymization

Anonymization permanently removes the ability to identify an individual. Anonymized data is no longer personal data under GDPR/APPs.

```typescript
// lib/privacy/anonymize.ts

export function anonymizeEmail(email: string): string {
  // "john.doe@gmail.com" → "j***e@g***l.com"
  const [local, domain] = email.split("@");
  const anonLocal = local[0] + "***" + local[local.length - 1];
  const [domainName, tld] = domain.split(".");
  const anonDomain = domainName[0] + "***" + domainName[domainName.length - 1];
  return `${anonLocal}@${anonDomain}.${tld}`;
}

export function anonymizeIp(ip: string): string {
  // "192.168.1.100" → "192.168.0.0"
  const parts = ip.split(".");
  return `${parts[0]}.${parts[1]}.0.0`;
}

export function anonymizeProfile(profile: Record<string, unknown>): Record<string, unknown> {
  return {
    ...profile,
    id: crypto.randomUUID(), // Replace real ID
    full_name: null,
    email: null,
    phone: null,
    date_of_birth: null,
    passport_number_encrypted: null,
    // Keep non-identifying data for analytics
    nationality: profile.nationality,
    occupation_category: profile.occupation_category,
    visa_subclass: profile.visa_subclass,
    points_score: profile.points_score,
    created_at: profile.created_at,
    anonymized: true,
    anonymized_at: new Date().toISOString(),
  };
}
```

### Batch Anonymization for Analytics

```sql
-- Create anonymized analytics table
CREATE TABLE analytics_assessments (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  occupation_category text,
  visa_subclass text,
  points_score integer,
  english_level text,
  assessment_result text,
  created_month date, -- Only month, not exact date
  source_region text  -- Country only, not city
);

-- Function to anonymize and archive completed assessments
CREATE OR REPLACE FUNCTION anonymize_old_assessments()
RETURNS integer
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  count integer;
BEGIN
  WITH moved AS (
    INSERT INTO analytics_assessments (
      occupation_category, visa_subclass, points_score,
      english_level, assessment_result, created_month, source_region
    )
    SELECT
      occupation_category, visa_subclass, points_score,
      english_level, result,
      date_trunc('month', created_at)::date,
      nationality
    FROM visa_assessments
    WHERE status = 'completed'
    AND created_at < now() - interval '6 months'
    RETURNING id
  )
  SELECT count(*) INTO count FROM moved;

  -- Delete the original identified records
  DELETE FROM visa_assessments
  WHERE status = 'completed'
  AND created_at < now() - interval '6 months';

  RETURN count;
END;
$$;
```

## Pseudonymization

Pseudonymization replaces identifiers with artificial ones. Unlike anonymization, it's reversible with the key. GDPR still considers pseudonymized data as personal data.

```typescript
// lib/privacy/pseudonymize.ts
import crypto from "crypto";

const PSEUDO_SECRET = process.env.PSEUDONYMIZATION_KEY!;

export function pseudonymizeId(userId: string): string {
  return crypto
    .createHmac("sha256", PSEUDO_SECRET)
    .update(userId)
    .digest("hex")
    .substring(0, 16);
}

export function pseudonymizeForAnalytics(data: {
  userId: string;
  action: string;
  metadata: Record<string, unknown>;
}) {
  return {
    pseudoId: pseudonymizeId(data.userId),
    action: data.action,
    metadata: data.metadata,
    // No real user ID, email, or name
  };
}
```

## Secure File Handling

For documents like passport scans and police clearances:

```typescript
// lib/privacy/secure-upload.ts
import { createClient } from "@/lib/supabase/server";

const ALLOWED_TYPES = ["application/pdf", "image/jpeg", "image/png"];
const MAX_SIZE = 10 * 1024 * 1024; // 10MB

export async function secureUpload(
  userId: string,
  file: File,
  category: "passport" | "police_clearance" | "qualification"
) {
  // 1. Validate file type
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new Error(`File type ${file.type} not allowed`);
  }

  // 2. Validate file size
  if (file.size > MAX_SIZE) {
    throw new Error("File too large (max 10MB)");
  }

  // 3. Generate non-guessable filename
  const ext = file.name.split(".").pop();
  const filename = `${userId}/${category}/${crypto.randomUUID()}.${ext}`;

  // 4. Upload to private bucket
  const supabase = await createClient();
  const { error } = await supabase.storage
    .from("user-documents") // Must be a PRIVATE bucket
    .upload(filename, file, {
      contentType: file.type,
      upsert: false,
    });

  if (error) throw new Error(`Upload failed: ${error.message}`);

  // 5. Log the upload
  const { logAuditEvent } = await import("./audit-log");
  await logAuditEvent(userId, "create", "document", filename, { category, size: file.size });

  return filename;
}

// Generate time-limited signed URLs for viewing
export async function getSecureDocumentUrl(userId: string, path: string): Promise<string> {
  const supabase = await createClient();

  // Verify the user owns this document
  if (!path.startsWith(`${userId}/`)) {
    throw new Error("Unauthorized access to document");
  }

  const { data, error } = await supabase.storage
    .from("user-documents")
    .createSignedUrl(path, 300); // 5 minute expiry

  if (error) throw new Error(`Failed to generate URL: ${error.message}`);

  // Log the access
  const { logAuditEvent } = await import("./audit-log");
  await logAuditEvent(userId, "view", "document", path);

  return data.signedUrl;
}
```

## Data Minimization in AI Prompts

When sending data to Gemini API, strip unnecessary PII:

```typescript
// lib/privacy/sanitize-for-ai.ts
export function sanitizeForAiProcessing(profile: UserProfile): Record<string, unknown> {
  // Only send what the AI needs for assessment — no identifying info
  return {
    occupation: profile.occupation,
    occupation_code: profile.occupation_code,
    years_experience: profile.years_experience,
    qualification_level: profile.qualification_level,
    english_test_type: profile.english_test_type,
    english_scores: profile.english_scores,
    age_range: getAgeRange(profile.date_of_birth), // "25-32" not exact date
    // DO NOT SEND: name, email, passport, DOB, phone, address
  };
}

function getAgeRange(dob: string): string {
  const age = Math.floor((Date.now() - new Date(dob).getTime()) / 31557600000);
  if (age < 25) return "under-25";
  if (age <= 32) return "25-32";
  if (age <= 39) return "33-39";
  if (age <= 44) return "40-44";
  return "45-plus";
}
```

## Secure Deletion

When deleting data, ensure it's actually gone:

```typescript
// lib/privacy/secure-delete.ts
export async function secureDeleteUser(userId: string) {
  const adminSupabase = createAdminClient();

  // 1. Delete from all tables (order matters for foreign keys)
  const tables = [
    "visa_assessments",
    "user_documents",
    "user_consents",
    "notifications",
    "audit_logs",
    "profiles",
  ];

  for (const table of tables) {
    const { error } = await adminSupabase.from(table).delete().eq("user_id", userId);
    if (error) console.error(`Failed to delete from ${table}:`, error.message);
  }

  // 2. Delete files from storage
  const { data: folders } = await adminSupabase.storage
    .from("user-documents")
    .list(userId);

  if (folders?.length) {
    for (const folder of folders) {
      const { data: files } = await adminSupabase.storage
        .from("user-documents")
        .list(`${userId}/${folder.name}`);

      if (files?.length) {
        await adminSupabase.storage
          .from("user-documents")
          .remove(files.map((f) => `${userId}/${folder.name}/${f.name}`));
      }
    }
  }

  // 3. Delete auth user (invalidates all sessions)
  await adminSupabase.auth.admin.deleteUser(userId);

  // 4. Create anonymized deletion record (for legal compliance)
  await adminSupabase.from("deletion_records").insert({
    anonymized_id: crypto.randomUUID(),
    deleted_at: new Date().toISOString(),
    reason: "user_requested",
  });
}
```
