# Publish30 LinkedIn Certificate — setup notes

What's built so far (in the `publish30-idea-generator` Supabase project — reused rather
than spinning up a separate free project, since it's already active and on the same
account):

- `public.credentials` table — one row per finisher: `credential_id` (random 8-char hex,
  auto-generated, not guessable, not sequential), `full_name`, `email`,
  `completion_month`, `completion_year`, `created_at`.
- Row Level Security is **on** with **zero policies**, so the table is completely
  unreachable from the public API — no listing, no reading emails, nothing. Verified
  directly against the `anon` role: `select count(*) from credentials` returns `0` rows
  for that role, and a wrong credential ID returns an empty result, never an error (so
  there's no way to probe which IDs exist).
- `public.verify_credential(credential_id)` — the one way in from the outside. Exact-match
  lookup, returns just `full_name` and a formatted `completion_date` (e.g. "October
  2026"). Nothing else. Granted to `anon`/`authenticated` on purpose — that's what the
  verify page below calls.
- `public.app_config` — two settings (`linkedin_organization_id`, `verify_base_url`), same
  RLS lockdown as `credentials`. Lets you update the org ID once you have it without
  touching code.
- `public.linkedin_add_to_profile_url(credential_id, month, year)` — builds one finisher's
  "Add to profile" LinkedIn link from a row + the two config values. Not exposed to the
  public API — you run it yourself (see "Generate the links" below).

One thing to clean up: there's a `Test Finisher` / `test@example.com` test row in
`credentials` (credential ID `2840d6cf`) that I used to verify everything end-to-end. The
delete needs your confirmation (it timed out waiting for an answer when I tried it), so
either tell me to delete it or run this yourself in the Supabase SQL editor:

```sql
delete from public.credentials where email = 'test@example.com';
```

## 1. Set the real LinkedIn organization ID

Once you've created the Publish30 LinkedIn Page and have its numeric org ID:

```sql
update public.app_config set value = '123456789' where key = 'linkedin_organization_id';
```

And once you know where the verify page will actually live (see deployment below):

```sql
update public.app_config
set value = 'https://verify.publish30.com/verify'  -- no trailing slash
where key = 'verify_base_url';
```

## 2. Import the finisher list

Send the list as a CSV with these columns (any order): `full_name`, `email`,
`completion_month` (1-12), `completion_year` (e.g. 2026). Don't include `credential_id` —
it's generated automatically on insert. Import via the Supabase dashboard's Table Editor
→ `credentials` → Insert → Import data from CSV, or send me the file and I'll load it.

## 3. Generate the LinkedIn links

Run this (SQL editor, or ask me to run it) any time after import to get every finisher's
name, email, and personalized "Add to profile" link in one result you can export as CSV
for the mail merge:

```sql
select full_name, email,
       to_char(make_date(completion_year, completion_month, 1), 'FMMonth YYYY') as completion_date,
       linkedin_add_to_profile_url(credential_id, completion_month, completion_year) as link
from credentials
order by full_name;
```

(There's deliberately no standing database view for this — a view can silently inherit
its owner's privileges and bypass the table's RLS lockdown if it's ever granted too
broadly later on, so this stays a query you run on demand instead of a permanent object.)

## 4. Deploy the verification page

Everything in this folder is the whole site — `index.html` (a small landing page),
`verify/index.html` (the actual verification page), and `_redirects` (a Cloudflare Pages
rule, ignored by other hosts). It's self-contained static HTML/CSS/JS, no build step.

The page reads the credential ID two ways, so it works on either host without extra
config:
- Path style: `https://yourdomain/verify/abc12345` (needs the `_redirects` rewrite — works
  out of the box on **Cloudflare Pages**)
- Query string: `https://yourdomain/verify/?id=abc12345` (works everywhere, including
  **GitHub Pages**, with zero extra config)

**Cloudflare Pages** (recommended per the plan — free tier is fine for commercial use):
1. pages.cloudflare.com → Create a project → Direct Upload.
2. Drag this whole folder in, or zip it and upload the zip.
3. Deploy. Use the path-style links (`_redirects` makes `/verify/abc12345` work).
4. Optional: attach a custom domain in the project's Custom Domains tab (~$12/yr for the
   domain itself, Cloudflare Pages hosting stays free).

**GitHub Pages** (also free for commercial use):
1. Push this folder to a repo, enable Pages on the `main` branch in Settings → Pages.
2. Use query-string links (`/verify/?id=abc12345`) — GitHub Pages won't serve an arbitrary
   `/verify/abc12345` path to `verify/index.html` without extra routing tricks, so skip
   that and just use the query string form; it's indistinguishable to the finisher.

The anon Supabase key is hardcoded in `verify/index.html`. That's expected and safe — it's
a public, publishable key, and RLS plus the function grants are what actually protect the
data, not keeping that key secret.

## 5. Supabase free-tier pause — keep-alive

A daily scheduled ping to `verify_credential` keeps the free Supabase project from
auto-pausing after a week of inactivity, which would otherwise turn every finisher's
LinkedIn certificate link into a dead page. This is already set up as a recurring
scheduled task (not something that needs to be redone).

## Still pending (not part of this handoff)

- Badge design in Canva.
- Finisher email + LinkedIn post drafts (also already drafted separately).
- Future cohorts: repeat the name-matching / import / link-generation process above.
