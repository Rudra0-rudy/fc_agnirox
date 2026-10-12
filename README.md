# Agnirox FC

Website for **Agnirox FC**, a football club from Asansol, West Bengal (est. 2025).
The public site shows the squad, match report, season stats and photo gallery.
A password-protected admin page lets the club update everything without touching code.

**Live site:** https://fc-agnirox.vercel.app

---

## How it works

```
 admin.html ──(password + JSON)──▶  Supabase  ◀──(read only)── index.html
  (coach edits)                  (content table)              (public visitors)
```

- **Static site.** Plain HTML, CSS and JavaScript. No build step, no framework.
- **Supabase** stores all editable content as one JSON row.
- **index.html** reads that row and renders the page. It re-checks every 30 seconds.
- **admin.html** is the editor. It verifies a password through a Supabase function before it unlocks, and saves through another password-checked function.
- If Supabase is empty or unreachable, `index.html` falls back to built-in default data, so the page never shows up blank.

## Files

| File | Purpose |
|---|---|
| `index.html` | Public website |
| `admin.html` | Admin panel (not indexed by search engines) |
| `README.md` | This file |

## Setup

### 1. Create a Supabase project

Create a project at [supabase.com](https://supabase.com). From **Project Settings → API**, copy:

- **Project URL**, e.g. `https://xxxxxxxxxxxxxxxxxxxx.supabase.co`
- **anon public key**

The URL and key must belong to the **same project**. The key is a JWT, and its `ref` field has to match the subdomain of the URL. A mismatch is the most common reason the site can't connect.

### 2. Run the SQL

Open **SQL Editor** in Supabase, paste the following, change the password, and run it. It is safe to run again.

```sql
create extension if not exists pgcrypto with schema extensions;

-- content (single row)
create table if not exists public.content (
  id         int primary key default 1,
  data       jsonb not null default '{}'::jsonb,
  updated_at timestamptz default now(),
  constraint single_row check (id = 1)
);
insert into public.content (id, data) values (1, '{}'::jsonb)
on conflict (id) do nothing;

-- admin password (stored hashed)
create table if not exists public.admin_secrets (
  id            int primary key default 1,
  password_hash text not null,
  constraint single_secret check (id = 1)
);

-- CHANGE 'your-password-here' before running
insert into public.admin_secrets (id, password_hash)
values (1, extensions.crypt('your-password-here', extensions.gen_salt('bf')))
on conflict (id) do update set password_hash = excluded.password_hash;

-- security: public can read content, nothing else
alter table public.content       enable row level security;
alter table public.admin_secrets enable row level security;

drop policy if exists "public read" on public.content;
create policy "public read" on public.content
  for select to anon, authenticated using (true);

revoke all on public.admin_secrets from anon, authenticated;
revoke insert, update, delete on public.content from anon, authenticated;
grant  select on public.content to anon, authenticated;

-- login check
create or replace function public.verify_admin_password(p_password text)
returns boolean language plpgsql security definer
set search_path = public, extensions as $$
declare stored_hash text;
begin
  select password_hash into stored_hash from public.admin_secrets where id = 1;
  return stored_hash is not null
     and extensions.crypt(p_password, stored_hash) = stored_hash;
end; $$;

-- save content
create or replace function public.update_content_with_password(
  p_password text, p_data jsonb)
returns boolean language plpgsql security definer
set search_path = public, extensions as $$
declare stored_hash text;
begin
  select password_hash into stored_hash from public.admin_secrets where id = 1;
  if stored_hash is null
     or extensions.crypt(p_password, stored_hash) <> stored_hash then
    raise exception 'Invalid password';
  end if;
  if p_data is null or jsonb_typeof(p_data) <> 'object' then
    raise exception 'Invalid data';
  end if;
  update public.content set data = p_data, updated_at = now() where id = 1;
  return true;
end; $$;

revoke all on function public.verify_admin_password(text)              from public;
revoke all on function public.update_content_with_password(text,jsonb) from public;
grant execute on function public.verify_admin_password(text)              to anon;
grant execute on function public.update_content_with_password(text,jsonb) to anon;

notify pgrst, 'reload schema';
```

### 3. Add your keys to the site

In both `index.html` and `admin.html`, find the config block and set your own values:

```js
const SUPABASE_URL  = 'https://YOUR-PROJECT-REF.supabase.co';
const SUPABASE_ANON = 'YOUR-ANON-PUBLIC-KEY';
```

The anon key is meant to be public. It is safe to commit. Your data is protected by the row-level security rules and the password-checked functions above.

### 4. Deploy

Push to GitHub and import the repo in [Vercel](https://vercel.com). No build settings are needed. Every push to `main` redeploys the site.

## Using the admin panel

1. Open `/admin.html` on your site.
2. Enter the admin password.
3. Edit the hero scorecard, match report, season numbers, squad and gallery.
4. Press **Save to Supabase** (or `Ctrl/Cmd + S`).
5. The public site picks up the change within 30 seconds, or on the next page load.

On the first login the panel shows "Using defaults · not yet saved". Press Save once to write the starting data to Supabase.

**Image hosting:** photos are linked by URL (currently hosted on ibb.co). Upload a photo to any image host and paste the direct link into the admin panel.

## Data format

Everything lives in one JSON object:

```json
{
  "season": "2024–25",
  "hero":   { "status": "Full Time", "matchday": 24, "homeTeam": "Agnirox",
              "homeShort": "A", "awayTeam": "Rivals", "awayShort": "R",
              "homeScore": 3, "awayScore": 2, "result": "Won",
              "minute": "90+4'", "ground": "Chitra Ground" },
  "stats":  { "played": 24, "won": 18, "drawn": 3, "lost": 3,
              "goals": 35, "form": ["W", "W", "D", "W", "L"] },
  "reportTitle": "Last Sunday, 3–2.",
  "heroSub": "Short intro text under the headline",
  "report": { "matchday": 24, "tag": "Full Time · MD 24", "headline": "...",
              "lead": "...", "motm": { "name": "...", "tag": "GK" },
              "chips":   [{ "label": "POSSESSION · 47%" }],
              "scorers": [{ "name": "Player Name", "minute": "23'" }] },
  "players": [{ "name": "Player Name", "role": "Fwd", "img": "https://..." }],
  "gallery": ["https://.../1.jpg", "https://.../2.jpg"]
}
```

`result` can be `Won`, `Lost` or `Draw`.

## Run locally

Open `index.html` in a browser, or use VS Code's **Live Server** extension. Chrome or Edge works best. The page needs an internet connection to reach Supabase, Google Fonts and the image host.

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Red "Cloud unavailable" banner | The site can't reach Supabase. Check that `SUPABASE_URL` matches the project your key belongs to. Open the browser console (F12) and look for `Supabase fetch failed: HTTP ...`. |
| `HTTP 401` or `permission denied` | The read policy or grants are missing. Re-run the SQL in step 2. |
| `HTTP 404` | The `content` table doesn't exist, or the URL is wrong. |
| Site shows the default data, not your edits | The `content` row is empty. Log in to the admin panel and press Save. Then clear the cache: `localStorage.removeItem('agnirox_cloud_cache_v2')` in the console, and hard-refresh (`Ctrl+Shift+R`). |
| "Wrong password" at admin login | The `verify_admin_password` function is missing, or the password in the SQL differs from the one you typed. Re-run the SQL. |
| Save fails with "Invalid password" | Same as above. |
| Photos don't load | The image link is dead or not a direct image URL. Test it in a new tab. |

## Security notes

- The admin password is stored as a bcrypt hash in `admin_secrets`. That table is not readable from the browser.
- There is no rate limiting on password attempts. Use a long password (12+ characters).
- Anyone who knows the `/admin.html` URL can see the login screen. Only the password gets them in.
- Don't put your plain-text password in the repo. Keep it out of any SQL file you commit.

## Contact

- Instagram: [@agnirox_fc](https://www.instagram.com/agnirox_fc)
- Email: agniroxfc@gmail.com
- place: Asansol. Sundays.
