# Fix patterns by finding

Each fix goes in a new migration (`supabase/migrations/<UTC timestamp>_patchlabs_fix.sql`). Replace table, column
and policy names with the ones in the finding. `(select auth.uid())` is used instead of `auth.uid()` so Postgres
evaluates it once per query, not once per row. The full explanation of each finding, with a test query, is at
`https://patchlabs.dev/findings/<id>`.

Before writing any fix, check the app code for queries on the table (`.from('profiles')`, `select('*')`) and tell the
user what will change for logged-out visitors and other users.

## PL-001: anyone with the public key can read personal data

Intended when: the table is truly public content with no personal columns (blog posts, a product catalogue). Ask.

```sql
-- PL-001: profiles readable by anon, including email and phone
drop policy "Profiles are viewable by everyone" on public.profiles;
create policy "Users can view own profile"
  on public.profiles for select
  to authenticated
  using ((select auth.uid()) = id);
```

If logged-out visitors must still see some columns (a public profile page), grant only those columns:

```sql
revoke select on public.profiles from anon;
grant select (id, full_name) on public.profiles to anon;
-- The app must now select those columns by name; select('*') fails for anon.
```

## PL-002: any signed-in user can read every row

`using (true)` for `authenticated`, or `auth.uid() is not null`, is open to anyone who can sign up.

```sql
-- PL-002
drop policy "Authenticated users can view all profiles" on public.profiles;
create policy "Users can view own profile"
  on public.profiles for select
  to authenticated
  using ((select auth.uid()) = id);
```

Shared data (teams, workspaces): scope to membership instead.

```sql
create policy "Members read their team's projects"
  on public.projects for select
  to authenticated
  using (team_id in (
    select team_id from public.team_members where user_id = (select auth.uid())
  ));
```

## PL-003: RLS on, no policies

Not a leak: the app's own users get empty results. Never fix it by disabling RLS or adding `using (true)`.

```sql
-- PL-003
create policy "Owners read own notes" on public.notes
  for select to authenticated using ((select auth.uid()) = user_id);
create policy "Owners add own notes" on public.notes
  for insert to authenticated with check ((select auth.uid()) = user_id);
create policy "Owners edit own notes" on public.notes
  for update to authenticated
  using ((select auth.uid()) = user_id)
  with check ((select auth.uid()) = user_id);
create policy "Owners delete own notes" on public.notes
  for delete to authenticated using ((select auth.uid()) = user_id);
```

## PL-004: UPDATE policy without WITH CHECK

Without `with check`, a user can change their own row into anything, including `role`, `plan` or `user_id`.

```sql
-- PL-004
drop policy "Users can update own profile" on public.profiles;
create policy "Users can update own profile"
  on public.profiles for update
  to authenticated
  using ((select auth.uid()) = id)
  with check ((select auth.uid()) = id);
-- Optional: allow only the columns users should edit
revoke update on public.profiles from authenticated;
grant update (full_name) on public.profiles to authenticated;
```

## PL-005: policy is always true

Replace `using (true)` / `with check (true)` on writes with a real condition. Keep anon read only on tables meant to
be public. An insert-only `with check (true)` for a contact form can be intended; say so.

```sql
-- PL-005
drop policy "Enable update for all users" on public.orders;
create policy "Customers update own orders"
  on public.orders for update
  to authenticated
  using ((select auth.uid()) = user_id)
  with check ((select auth.uid()) = user_id);
```

## PL-006: SECURITY DEFINER function exposure

A definer function runs with its owner's rights and, in `public`, is callable through the API.

```sql
-- PL-006: keep definer only where needed, pin search_path, limit who can call it
create or replace function public.get_my_email()
returns text
language sql
security definer
set search_path = ''
as $$
  select email from auth.users where id = (select auth.uid());
$$;
revoke execute on function public.get_my_email() from public, anon;
grant execute on function public.get_my_email() to authenticated;
```

If the function does not need elevated rights, use `security invoker` instead. Server-only functions belong outside
the `public` schema.

## PL-007: public storage bucket

Public buckets serve any file to anyone with its URL. Intended for site images and avatars; not for IDs, invoices or
user uploads.

```sql
-- PL-007
update storage.buckets set public = false where id = 'documents';
create policy "Owners read own documents"
  on storage.objects for select
  to authenticated
  using (
    bucket_id = 'documents'
    and (storage.foldername(name))[1] = (select auth.uid())::text
  );
```

The app must then download with signed URLs (`createSignedUrl`) and upload under `<user id>/...`.

## PL-008: RLS not enabled

With RLS off, the anon key can read and write the whole table.

```sql
-- PL-008
alter table public.leads enable row level security;
-- Keep the public contact form working
create policy "Anyone can submit a lead"
  on public.leads for insert to anon, authenticated with check (true);
-- Only admins read
create policy "Admins read leads"
  on public.leads for select to authenticated
  using (exists (
    select 1 from public.user_roles r
    where r.user_id = (select auth.uid()) and r.role = 'admin'
  ));
```

Enabling RLS with no policies locks everyone out (PL-003). Always add the policies in the same migration.

## PL-014: view skips row level security

A view without `security_invoker` runs as its owner and returns rows the caller's RLS would hide. Needs Postgres 15
or newer (all current Supabase projects).

```sql
-- PL-014
alter view public.order_summaries set (security_invoker = true);
revoke select on public.order_summaries from anon;
```

Materialized views have no RLS at all: revoke `select` from `anon` and `authenticated` and serve them through a
function that checks the caller, or move them out of `public`. Intended when: a public aggregate with no personal
data (counts, totals). Ask.

## PL-009 and PL-010: secrets in code or Claude Code settings

Order matters:

1. Rotate the key in the provider dashboard (Supabase: Settings, API; Stripe: Developers, API keys). Until then it
   still works for anyone who copied it.
2. Remove it from the code; read it from a server-side env var with no public prefix (`VITE_`, `NEXT_PUBLIC_`).
3. For `.claude/settings.local.json`: delete the allow rule or `env` value, add the file to `.gitignore`, and
   `git rm --cached` it if git tracks it.
4. Tell the user it remains in git history and in any published package; rotation is what makes it safe.

The service role key must never reach browser code. Browser code uses the anon key plus RLS.

## Live site (PL-027 to PL-036)

These are header and cookie settings on the host (Vercel `vercel.json` headers, Netlify `_headers`, Cloudflare
rules), not code bugs. Suggest the header for the user's host and check it does not break embeds or scripts the site
loads (a strict Content-Security-Policy often does; start with `Content-Security-Policy-Report-Only`).
