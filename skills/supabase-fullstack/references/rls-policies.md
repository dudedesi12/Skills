# RLS Policy Patterns

Row Level Security (RLS) is how Supabase protects your data. When RLS is enabled on a table, NO rows are accessible unless a policy explicitly allows it.

## How RLS Works

- `USING (condition)` — controls which existing rows can be read/updated/deleted
- `WITH CHECK (condition)` — controls which new rows can be inserted/updated
- `auth.uid()` — returns the currently logged-in user's UUID
- `auth.jwt()` — returns the full JWT token (access custom claims)

## Pattern 1: Owner-Only Access

Users can only see and modify their own data.

```sql
ALTER TABLE notes ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own notes"
ON notes FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "Users can create own notes"
ON notes FOR INSERT
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own notes"
ON notes FOR UPDATE
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete own notes"
ON notes FOR DELETE
USING (auth.uid() = user_id);
```

## Pattern 2: Public Read, Owner Write

Anyone can read, only the owner can modify.

```sql
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read posts"
ON posts FOR SELECT
USING (true);

CREATE POLICY "Authors can create posts"
ON posts FOR INSERT
WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Authors can update own posts"
ON posts FOR UPDATE
USING (auth.uid() = author_id)
WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Authors can delete own posts"
ON posts FOR DELETE
USING (auth.uid() = author_id);
```

## Pattern 3: Public Read, Authenticated Write

Anyone can read, any logged-in user can create.

```sql
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read comments"
ON comments FOR SELECT
USING (true);

CREATE POLICY "Authenticated users can create comments"
ON comments FOR INSERT
WITH CHECK (auth.uid() IS NOT NULL);

CREATE POLICY "Users can update own comments"
ON comments FOR UPDATE
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete own comments"
ON comments FOR DELETE
USING (auth.uid() = user_id);
```

## Pattern 4: Published/Draft Visibility

Only show published items to the public. Authors can see their own drafts.

```sql
ALTER TABLE articles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read published articles"
ON articles FOR SELECT
USING (
  published = true
  OR auth.uid() = author_id
);

CREATE POLICY "Authors can create articles"
ON articles FOR INSERT
WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Authors can update own articles"
ON articles FOR UPDATE
USING (auth.uid() = author_id)
WITH CHECK (auth.uid() = author_id);
```

## Pattern 5: Role-Based Access (using profiles table)

```sql
-- Assume profiles table has a 'role' column: 'user', 'moderator', 'admin'

ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read posts"
ON posts FOR SELECT
USING (true);

CREATE POLICY "Users can create posts"
ON posts FOR INSERT
WITH CHECK (auth.uid() IS NOT NULL);

CREATE POLICY "Authors and moderators can update posts"
ON posts FOR UPDATE
USING (
  auth.uid() = author_id
  OR EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid()
    AND role IN ('moderator', 'admin')
  )
);

CREATE POLICY "Authors and admins can delete posts"
ON posts FOR DELETE
USING (
  auth.uid() = author_id
  OR EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid()
    AND role = 'admin'
  )
);
```

## Pattern 6: Team-Based Access

Users can access data belonging to their team.

```sql
-- team_members table: user_id, team_id, role

ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Team members can view projects"
ON projects FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM team_members
    WHERE team_members.team_id = projects.team_id
    AND team_members.user_id = auth.uid()
  )
);

CREATE POLICY "Team members can create projects"
ON projects FOR INSERT
WITH CHECK (
  EXISTS (
    SELECT 1 FROM team_members
    WHERE team_members.team_id = projects.team_id
    AND team_members.user_id = auth.uid()
  )
);

CREATE POLICY "Team admins can update projects"
ON projects FOR UPDATE
USING (
  EXISTS (
    SELECT 1 FROM team_members
    WHERE team_members.team_id = projects.team_id
    AND team_members.user_id = auth.uid()
    AND team_members.role IN ('admin', 'owner')
  )
);

CREATE POLICY "Team owners can delete projects"
ON projects FOR DELETE
USING (
  EXISTS (
    SELECT 1 FROM team_members
    WHERE team_members.team_id = projects.team_id
    AND team_members.user_id = auth.uid()
    AND team_members.role = 'owner'
  )
);
```

## Pattern 7: Admin Override

Admins can do everything, regardless of other policies.

```sql
-- Using JWT custom claims (set via Supabase dashboard or auth hook)
CREATE POLICY "Admins can do everything"
ON posts FOR ALL
USING (
  (auth.jwt() -> 'app_metadata' ->> 'role') = 'admin'
);

-- Or using a profiles table
CREATE POLICY "Admins can do everything"
ON posts FOR ALL
USING (
  EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid()
    AND role = 'admin'
  )
);
```

## Pattern 8: Self-Registration (profiles)

Users can create their own profile (only one, matching their auth ID).

```sql
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view all profiles"
ON profiles FOR SELECT
USING (true);

CREATE POLICY "Users can create own profile"
ON profiles FOR INSERT
WITH CHECK (auth.uid() = id);

CREATE POLICY "Users can update own profile"
ON profiles FOR UPDATE
USING (auth.uid() = id)
WITH CHECK (auth.uid() = id);
```

## Pattern 9: Invite-Only Access

Only users who have been explicitly invited can access a resource.

```sql
-- invitations table: resource_id, invited_user_id, status

ALTER TABLE resources ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Invited users can view resources"
ON resources FOR SELECT
USING (
  auth.uid() = owner_id
  OR EXISTS (
    SELECT 1 FROM invitations
    WHERE invitations.resource_id = resources.id
    AND invitations.invited_user_id = auth.uid()
    AND invitations.status = 'accepted'
  )
);
```

## Pattern 10: Time-Based Access

Content is only visible after a certain date.

```sql
ALTER TABLE scheduled_posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Published posts visible after publish date"
ON scheduled_posts FOR SELECT
USING (
  publish_at <= now()
  OR auth.uid() = author_id
);
```

## Pattern 11: Rate Limiting via RLS

Limit how many rows a user can insert per day.

```sql
CREATE POLICY "Users limited to 10 posts per day"
ON posts FOR INSERT
WITH CHECK (
  auth.uid() = author_id
  AND (
    SELECT COUNT(*) FROM posts
    WHERE author_id = auth.uid()
    AND created_at > now() - interval '1 day'
  ) < 10
);
```

## Pattern 12: Soft Delete

Users can "delete" rows but they are just marked as deleted. Only admins see deleted rows.

```sql
ALTER TABLE items ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users see non-deleted items"
ON items FOR SELECT
USING (
  deleted_at IS NULL
  OR EXISTS (
    SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin'
  )
);

CREATE POLICY "Users can soft-delete own items"
ON items FOR UPDATE
USING (auth.uid() = user_id)
WITH CHECK (auth.uid() = user_id);
```

## Pattern 13: Hierarchical Access (Parent-Child)

Access to child records requires access to the parent.

```sql
ALTER TABLE comments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Comments visible if parent post is visible"
ON comments FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM posts
    WHERE posts.id = comments.post_id
    AND (posts.published = true OR posts.author_id = auth.uid())
  )
);
```

## Pattern 14: Friend/Follower Access

Only show content to users who follow the author.

```sql
ALTER TABLE private_posts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Followers and author can view"
ON private_posts FOR SELECT
USING (
  auth.uid() = author_id
  OR EXISTS (
    SELECT 1 FROM follows
    WHERE follows.following_id = private_posts.author_id
    AND follows.follower_id = auth.uid()
  )
);
```

## Pattern 15: Service Role Bypass

The service role key bypasses RLS entirely. Use it in:
- Webhooks
- Background jobs
- Admin operations
- Edge Functions that need full access

```ts
// ONLY use in server-side code, NEVER in browser
import { createClient } from "@supabase/supabase-js";

const adminClient = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY! // Bypasses ALL RLS
);
```

## Common Mistakes

### Mistake 1: Forgetting to enable RLS
```sql
-- ALWAYS enable RLS on every table
ALTER TABLE my_table ENABLE ROW LEVEL SECURITY;
-- Without policies after enabling, NO ONE can access the data
```

### Mistake 2: Using `auth.uid()` without checking for NULL
```sql
-- BAD — if user is not logged in, auth.uid() is NULL
-- NULL = NULL evaluates to NULL (not true), so this is actually safe
-- But be explicit:
CREATE POLICY "Authenticated users only"
ON secret_data FOR SELECT
USING (auth.uid() IS NOT NULL);
```

### Mistake 3: Forgetting WITH CHECK on UPDATE policies
```sql
-- BAD — user could update author_id to someone else's ID
CREATE POLICY "Update own"
ON posts FOR UPDATE
USING (auth.uid() = author_id);

-- GOOD — prevents changing ownership
CREATE POLICY "Update own"
ON posts FOR UPDATE
USING (auth.uid() = author_id)
WITH CHECK (auth.uid() = author_id);
```

### Mistake 4: Performance — use EXISTS instead of IN
```sql
-- BAD — slow with large tables
USING (
  team_id IN (SELECT team_id FROM team_members WHERE user_id = auth.uid())
);

-- GOOD — faster with indexes
USING (
  EXISTS (
    SELECT 1 FROM team_members
    WHERE team_members.team_id = projects.team_id
    AND team_members.user_id = auth.uid()
  )
);
```
