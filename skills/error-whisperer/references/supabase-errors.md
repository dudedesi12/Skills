# Supabase Errors (30+)

Every common Supabase error with plain English explanation and exact fix.

## Auth Errors

### 1. Error: Invalid login credentials

**Why:** Wrong email or password. Supabase does not tell you which one for security.

**Fix:** Double-check the email exists and password is correct. Reset password if needed.
```typescript
const { error } = await supabase.auth.signInWithPassword({ email, password });
if (error) {
  // Show user-friendly message, not the raw error
  setError("Invalid email or password. Please try again.");
}
```

### 2. Error: Email not confirmed

**Why:** User signed up but has not clicked the confirmation link in their email.

**Fix:** Check spam folder. Or disable email confirmation in Supabase dashboard: Authentication > Providers > Email > Toggle off "Confirm email".

### 3. Error: User already registered

**Why:** Trying to sign up with an email that already has an account.

**Fix:** Redirect to login instead, or show "Account already exists, please log in."

### 4. Error: JWT expired

**Why:** The session token has expired and needs to be refreshed.

**Fix:**
```typescript
// Supabase client auto-refreshes, but make sure middleware handles it:
// lib/supabase/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
  const response = NextResponse.next({ request: { headers: request.headers } });
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() { return request.cookies.getAll(); },
        setAll(cookies) {
          cookies.forEach(({ name, value, options }) => {
            response.cookies.set(name, value, options);
          });
        },
      },
    }
  );
  await supabase.auth.getUser();
  return response;
}
```

### 5. Error: Invalid Refresh Token

**Why:** The refresh token was revoked or expired (usually after long inactivity).

**Fix:** Sign the user out and redirect to login:
```typescript
await supabase.auth.signOut();
window.location.href = "/login";
```

### 6. Error: OAuth callback error — provider not enabled

**Why:** The OAuth provider (Google, GitHub, etc.) is not configured in Supabase.

**Fix:** Go to Supabase Dashboard > Authentication > Providers > Enable and configure the provider with client ID and secret.

### 7. Error: Redirect URL mismatch

**Why:** The redirect URL after OAuth login is not in the allowed list.

**Fix:** Go to Supabase Dashboard > Authentication > URL Configuration > Add the redirect URL.

## RLS (Row Level Security) Errors

### 8. new row violates row-level security policy

**Why:** The current user does not have permission to INSERT this row based on RLS policies.

**Fix:** Create or fix the RLS policy:
```sql
-- Allow users to insert their own rows
CREATE POLICY "Users can insert own data" ON public.profiles
  FOR INSERT WITH CHECK (auth.uid() = user_id);
```

### 9. Query returns empty array (no error, just no data)

**Why:** RLS is silently blocking the query. This is the sneakiest Supabase issue.

**Fix:** Check RLS policies:
```sql
-- Check existing policies
SELECT * FROM pg_policies WHERE tablename = 'your_table';

-- Add a SELECT policy
CREATE POLICY "Users can read own data" ON public.profiles
  FOR SELECT USING (auth.uid() = user_id);
```

**Debug:** Temporarily use the service role key (server-side only) to confirm data exists:
```typescript
import { createClient } from "@supabase/supabase-js";
const admin = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY!);
const { data } = await admin.from("profiles").select("*");
```

### 10. Error: permission denied for table X

**Why:** RLS is enabled but no policies exist, or the user role does not have access.

**Fix:**
```sql
-- Option 1: Add proper RLS policies (recommended)
ALTER TABLE public.your_table ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Allow authenticated access" ON public.your_table
  FOR ALL USING (auth.role() = 'authenticated');

-- Option 2: For public read tables
CREATE POLICY "Public read access" ON public.your_table
  FOR SELECT USING (true);
```

### 11. UPDATE/DELETE affects 0 rows (silently)

**Why:** RLS policy blocks the UPDATE/DELETE but does not throw an error.

**Fix:** Check that RLS policies exist for UPDATE and DELETE operations:
```sql
CREATE POLICY "Users can update own data" ON public.profiles
  FOR UPDATE USING (auth.uid() = user_id);
```

## Database Errors

### 12. Error: relation "X" does not exist

**Why:** The table has not been created yet, or you are querying the wrong schema.

**Fix:** Run your migration or create the table:
```sql
CREATE TABLE public.profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### 13. Error: duplicate key value violates unique constraint

**Why:** You are trying to insert a row with a value that already exists in a unique column.

**Fix:** Use upsert instead:
```typescript
const { data, error } = await supabase
  .from("profiles")
  .upsert({ user_id: userId, display_name: name }, { onConflict: "user_id" });
```

### 14. Error: null value in column "X" violates not-null constraint

**Why:** You tried to insert/update without providing a required field.

**Fix:** Include all required fields, or add a default value:
```sql
ALTER TABLE public.profiles ALTER COLUMN display_name SET DEFAULT 'Anonymous';
```

### 15. Error: foreign key constraint violation

**Why:** You are referencing a row in another table that does not exist, or deleting a row that others depend on.

**Fix:** Make sure the referenced row exists first, or use CASCADE:
```sql
ALTER TABLE public.posts
  ADD CONSTRAINT fk_author FOREIGN KEY (author_id) REFERENCES public.profiles(id) ON DELETE CASCADE;
```

### 16. Error: column "X" of relation "Y" does not exist

**Why:** You are querying a column that does not exist in the table.

**Fix:** Check column name spelling and case. Supabase column names are lowercase by default.

### 17. Error: type "X" does not exist

**Why:** Using a custom type that has not been created, or wrong type name.

**Fix:** Create the type first:
```sql
CREATE TYPE public.status AS ENUM ('active', 'inactive', 'pending');
```

## Connection Errors

### 18. Error: Connection terminated unexpectedly

**Why:** Too many connections, or Supabase project is paused.

**Fix:** Check if your project is paused in the Supabase dashboard. Free tier projects pause after 7 days of inactivity.

### 19. Error: SASL: SCRAM-SERVER-FIRST-MESSAGE

**Why:** Wrong database password.

**Fix:** Reset the database password in Supabase Dashboard > Settings > Database.

### 20. Error: remaining connection slots are reserved

**Why:** Too many open database connections.

**Fix:** Use connection pooling. Supabase provides a pooler on port 6543:
```
DATABASE_URL=postgresql://user:pass@db.xxxxx.supabase.co:6543/postgres?pgbouncer=true
```

## Storage Errors

### 21. Error: Bucket not found

**Why:** The storage bucket has not been created.

**Fix:**
```sql
INSERT INTO storage.buckets (id, name, public) VALUES ('avatars', 'avatars', true);
```
Or create it in Dashboard > Storage > New Bucket.

### 22. Error: new row violates row-level security policy (storage)

**Why:** Storage also uses RLS. You need policies for the storage bucket.

**Fix:**
```sql
CREATE POLICY "Allow authenticated uploads" ON storage.objects
  FOR INSERT WITH CHECK (bucket_id = 'avatars' AND auth.role() = 'authenticated');

CREATE POLICY "Allow public reads" ON storage.objects
  FOR SELECT USING (bucket_id = 'avatars');
```

### 23. Error: File too large

**Why:** File exceeds the upload size limit.

**Fix:** Set the limit in Dashboard > Storage > Policies, or compress the file before upload.

## Realtime Errors

### 24. Realtime subscription not receiving updates

**Why:** Table does not have `REPLICA IDENTITY FULL` set, or Realtime is not enabled.

**Fix:**
```sql
-- Enable replica identity for realtime
ALTER TABLE public.messages REPLICA IDENTITY FULL;
```
Then enable Realtime in Dashboard > Database > Replication > Enable for the table.

### 25. Error: Realtime connection dropped

**Why:** Client lost connection. Common on mobile or unstable networks.

**Fix:**
```typescript
const channel = supabase.channel("messages")
  .on("postgres_changes", { event: "*", schema: "public", table: "messages" }, (payload) => {
    console.log("Change received:", payload);
  })
  .subscribe((status) => {
    if (status === "CHANNEL_ERROR") {
      // Retry after a delay
      setTimeout(() => channel.subscribe(), 5000);
    }
  });
```

## Edge Function Errors

### 26. Error: Edge Function returned a non-2xx status code

**Why:** Your Edge Function threw an error.

**Fix:** Check the function logs in Dashboard > Edge Functions > Logs.

### 27. Error: Edge Function timed out

**Why:** Function took longer than the allowed time (default 60s).

**Fix:** Optimize the function or increase the timeout. Consider breaking into smaller functions.

## Migration Errors

### 28. Error: Migration failed — already applied

**Why:** Trying to re-apply a migration that already ran.

**Fix:**
```bash
# Check migration status
supabase migration list
# If stuck, repair the migration record
supabase migration repair --status applied <version>
```

### 29. Error: supabase db push failed

**Why:** Schema conflict between local and remote.

**Fix:**
```bash
# Pull the latest remote schema first
supabase db pull
# Then push your changes
supabase db push
```

## Client Errors

### 30. Error: supabaseUrl is required

**Why:** Environment variable `NEXT_PUBLIC_SUPABASE_URL` is not set.

**Fix:**
```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 31. Error: Multiple GoTrueClient instances detected

**Why:** Creating multiple Supabase client instances.

**Fix:** Use a singleton pattern:
```typescript
// lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### 32. Error: Cannot read property of null (after auth.getUser)

**Why:** No user is logged in. `getUser()` returned null.

**Fix:**
```typescript
const { data: { user } } = await supabase.auth.getUser();
if (!user) {
  redirect("/login");
}
```
