# Common SaaS Database Schema Templates

## Core: Users and Profiles

Every app needs this. The `profiles` table extends Supabase's built-in `auth.users` with your app-specific data.

```sql
-- Profiles (auto-created via trigger when user signs up)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  full_name TEXT NOT NULL DEFAULT '',
  username TEXT UNIQUE,
  avatar_url TEXT,
  bio TEXT DEFAULT '',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can view profiles" ON profiles FOR SELECT USING (true);
CREATE POLICY "Users can update own profile" ON profiles FOR UPDATE USING (auth.uid() = id) WITH CHECK (auth.uid() = id);

-- Auto-create profile on signup
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id, email, full_name, avatar_url)
  VALUES (
    NEW.id,
    NEW.email,
    COALESCE(NEW.raw_user_meta_data->>'full_name', ''),
    COALESCE(NEW.raw_user_meta_data->>'avatar_url', '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
AFTER INSERT ON auth.users
FOR EACH ROW EXECUTE FUNCTION handle_new_user();

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at BEFORE UPDATE ON profiles
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

## Teams / Organizations

Multi-tenant SaaS with team-based access.

```sql
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  logo_url TEXT,
  created_by UUID NOT NULL REFERENCES profiles(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TYPE team_role AS ENUM ('owner', 'admin', 'member', 'viewer');

CREATE TABLE team_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  role team_role NOT NULL DEFAULT 'member',
  joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(team_id, user_id)
);

CREATE INDEX team_members_team_id_idx ON team_members(team_id);
CREATE INDEX team_members_user_id_idx ON team_members(user_id);

ALTER TABLE teams ENABLE ROW LEVEL SECURITY;
ALTER TABLE team_members ENABLE ROW LEVEL SECURITY;

-- Team members can view their teams
CREATE POLICY "Team members can view team" ON teams FOR SELECT
USING (EXISTS (
  SELECT 1 FROM team_members WHERE team_members.team_id = teams.id AND team_members.user_id = auth.uid()
));

-- Only authenticated users can create teams
CREATE POLICY "Authenticated users can create teams" ON teams FOR INSERT
WITH CHECK (auth.uid() = created_by);

-- Only owners/admins can update teams
CREATE POLICY "Admins can update team" ON teams FOR UPDATE
USING (EXISTS (
  SELECT 1 FROM team_members WHERE team_members.team_id = teams.id AND team_members.user_id = auth.uid() AND team_members.role IN ('owner', 'admin')
));

-- Team members can view other members
CREATE POLICY "Members can view team members" ON team_members FOR SELECT
USING (EXISTS (
  SELECT 1 FROM team_members AS tm WHERE tm.team_id = team_members.team_id AND tm.user_id = auth.uid()
));

-- Auto-add creator as owner
CREATE OR REPLACE FUNCTION add_team_owner()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO team_members (team_id, user_id, role) VALUES (NEW.id, NEW.created_by, 'owner');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_team_created
AFTER INSERT ON teams
FOR EACH ROW EXECUTE FUNCTION add_team_owner();
```

## Team Invitations

```sql
CREATE TABLE team_invitations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  role team_role NOT NULL DEFAULT 'member',
  invited_by UUID NOT NULL REFERENCES profiles(id),
  token TEXT UNIQUE NOT NULL DEFAULT encode(gen_random_bytes(32), 'hex'),
  expires_at TIMESTAMPTZ NOT NULL DEFAULT now() + interval '7 days',
  accepted_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(team_id, email)
);

CREATE INDEX team_invitations_token_idx ON team_invitations(token);

ALTER TABLE team_invitations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Team admins can create invitations" ON team_invitations FOR INSERT
WITH CHECK (EXISTS (
  SELECT 1 FROM team_members WHERE team_members.team_id = team_invitations.team_id AND team_members.user_id = auth.uid() AND team_members.role IN ('owner', 'admin')
));

CREATE POLICY "Invitees can view their invitations" ON team_invitations FOR SELECT
USING (
  email = (SELECT email FROM auth.users WHERE id = auth.uid())
  OR EXISTS (
    SELECT 1 FROM team_members WHERE team_members.team_id = team_invitations.team_id AND team_members.user_id = auth.uid() AND team_members.role IN ('owner', 'admin')
  )
);
```

## Content / Blog / CMS

```sql
CREATE TABLE posts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  body TEXT DEFAULT '',
  excerpt TEXT DEFAULT '',
  cover_image_url TEXT,
  published BOOLEAN NOT NULL DEFAULT false,
  published_at TIMESTAMPTZ,
  author_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Full-text search
  fts TSVECTOR GENERATED ALWAYS AS (
    setweight(to_tsvector('english', COALESCE(title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(body, '')), 'B')
  ) STORED
);

CREATE INDEX posts_slug_idx ON posts(slug);
CREATE INDEX posts_author_id_idx ON posts(author_id);
CREATE INDEX posts_published_idx ON posts(published, published_at DESC);
CREATE INDEX posts_fts_idx ON posts USING GIN (fts);

CREATE TABLE tags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT UNIQUE NOT NULL,
  slug TEXT UNIQUE NOT NULL
);

CREATE TABLE post_tags (
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
ALTER TABLE tags ENABLE ROW LEVEL SECURITY;
ALTER TABLE post_tags ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read published posts" ON posts FOR SELECT
USING (published = true OR auth.uid() = author_id);

CREATE POLICY "Authors can create posts" ON posts FOR INSERT
WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Authors can update own posts" ON posts FOR UPDATE
USING (auth.uid() = author_id) WITH CHECK (auth.uid() = author_id);

CREATE POLICY "Authors can delete own posts" ON posts FOR DELETE
USING (auth.uid() = author_id);

CREATE POLICY "Anyone can read tags" ON tags FOR SELECT USING (true);
CREATE POLICY "Anyone can read post_tags" ON post_tags FOR SELECT USING (true);

-- Auto-set published_at when publishing
CREATE TRIGGER set_updated_at BEFORE UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE OR REPLACE FUNCTION set_published_at()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.published = true AND OLD.published = false THEN
    NEW.published_at = now();
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_publish BEFORE UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION set_published_at();
```

## Comments (Threaded)

```sql
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  body TEXT NOT NULL,
  post_id UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  parent_id UUID REFERENCES comments(id) ON DELETE CASCADE, -- For threading
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX comments_post_id_idx ON comments(post_id);
CREATE INDEX comments_user_id_idx ON comments(user_id);
CREATE INDEX comments_parent_id_idx ON comments(parent_id);

ALTER TABLE comments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can read comments on published posts" ON comments FOR SELECT
USING (EXISTS (
  SELECT 1 FROM posts WHERE posts.id = comments.post_id AND (posts.published = true OR posts.author_id = auth.uid())
));

CREATE POLICY "Authenticated users can create comments" ON comments FOR INSERT
WITH CHECK (auth.uid() = user_id AND auth.uid() IS NOT NULL);

CREATE POLICY "Users can update own comments" ON comments FOR UPDATE
USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can delete own comments" ON comments FOR DELETE
USING (auth.uid() = user_id);
```

## Subscriptions / Billing

```sql
CREATE TYPE subscription_status AS ENUM ('active', 'past_due', 'canceled', 'trialing', 'paused');
CREATE TYPE plan_interval AS ENUM ('monthly', 'yearly');

CREATE TABLE plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  description TEXT DEFAULT '',
  price_monthly INTEGER NOT NULL, -- in cents
  price_yearly INTEGER NOT NULL,  -- in cents
  features JSONB NOT NULL DEFAULT '[]',
  max_team_members INTEGER,
  max_storage_mb INTEGER,
  is_active BOOLEAN NOT NULL DEFAULT true,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
  plan_id UUID NOT NULL REFERENCES plans(id),
  status subscription_status NOT NULL DEFAULT 'trialing',
  interval plan_interval NOT NULL DEFAULT 'monthly',
  stripe_subscription_id TEXT UNIQUE,
  stripe_customer_id TEXT,
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  cancel_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX subscriptions_team_id_idx ON subscriptions(team_id);
CREATE INDEX subscriptions_stripe_sub_idx ON subscriptions(stripe_subscription_id);

ALTER TABLE plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Anyone can view active plans" ON plans FOR SELECT USING (is_active = true);

CREATE POLICY "Team members can view subscription" ON subscriptions FOR SELECT
USING (EXISTS (
  SELECT 1 FROM team_members WHERE team_members.team_id = subscriptions.team_id AND team_members.user_id = auth.uid()
));
```

## Notifications

```sql
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  type TEXT NOT NULL, -- 'comment', 'mention', 'invite', 'system'
  title TEXT NOT NULL,
  body TEXT DEFAULT '',
  link TEXT, -- URL to navigate to when clicked
  read_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX notifications_user_id_idx ON notifications(user_id, read_at, created_at DESC);

ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own notifications" ON notifications FOR SELECT
USING (auth.uid() = user_id);

CREATE POLICY "Users can update own notifications" ON notifications FOR UPDATE
USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);

-- Enable realtime for live notifications
ALTER PUBLICATION supabase_realtime ADD TABLE notifications;
```

## Activity Log / Audit Trail

```sql
CREATE TABLE activity_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES profiles(id) ON DELETE SET NULL,
  team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
  action TEXT NOT NULL, -- 'created', 'updated', 'deleted', 'invited', etc.
  resource_type TEXT NOT NULL, -- 'post', 'comment', 'team', etc.
  resource_id UUID,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX activity_log_team_idx ON activity_log(team_id, created_at DESC);
CREATE INDEX activity_log_user_idx ON activity_log(user_id, created_at DESC);

ALTER TABLE activity_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Team members can view activity" ON activity_log FOR SELECT
USING (EXISTS (
  SELECT 1 FROM team_members WHERE team_members.team_id = activity_log.team_id AND team_members.user_id = auth.uid()
));
```

## File Attachments

```sql
CREATE TABLE attachments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  file_name TEXT NOT NULL,
  file_size INTEGER NOT NULL, -- in bytes
  mime_type TEXT NOT NULL,
  storage_path TEXT NOT NULL, -- path in Supabase Storage
  uploaded_by UUID NOT NULL REFERENCES profiles(id),
  post_id UUID REFERENCES posts(id) ON DELETE CASCADE,
  team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX attachments_post_id_idx ON attachments(post_id);

ALTER TABLE attachments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Team members can view attachments" ON attachments FOR SELECT
USING (EXISTS (
  SELECT 1 FROM team_members WHERE team_members.team_id = attachments.team_id AND team_members.user_id = auth.uid()
));

CREATE POLICY "Team members can upload" ON attachments FOR INSERT
WITH CHECK (
  auth.uid() = uploaded_by AND
  EXISTS (
    SELECT 1 FROM team_members WHERE team_members.team_id = attachments.team_id AND team_members.user_id = auth.uid()
  )
);
```

## Waitlist (Pre-Launch)

```sql
CREATE TABLE waitlist (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  referral_code TEXT UNIQUE NOT NULL DEFAULT encode(gen_random_bytes(6), 'hex'),
  referred_by TEXT REFERENCES waitlist(referral_code),
  position INTEGER,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- No RLS needed if you only access via service role in a route handler
ALTER TABLE waitlist ENABLE ROW LEVEL SECURITY;
-- No policies = no public access (use service role key in server-side code)
```
