# Anthony Orciuoli — go-live setup

The site is a single static file (`index.html`). Before it's fully live you need to
fill in three public config values and deploy. None of these are secrets that grant
backend control — security is enforced server-side (Formspree validates its own form;
Supabase enforces Row-Level Security). **Never put a Supabase `service_role` key in the site.**

---

## 1. Contact form — Formspree

1. Create a free account at https://formspree.io.
2. Create a new form pointed at Anthony's email inbox.
3. Copy the form ID (the part after `/f/`, e.g. `xrgkabcd`).
4. In `index.html`, set:
   ```js
   const FORMSPREE_ID = 'xrgkabcd';   // ← your form ID
   ```
5. Formspree's free tier requires confirming the first submission via email.

A hidden honeypot field (`_gotcha`) is already wired for spam protection.

---

## 2. Events backend — Supabase

### a. Create the project
1. Create a free project at https://supabase.com.
2. Open **SQL Editor** and run:

   ```sql
   create table public.events (
     id uuid primary key default gen_random_uuid(),
     event_date date not null,
     title text not null,
     venue text,
     created_at timestamptz default now()
   );

   alter table public.events enable row level security;

   create policy "public read" on public.events
     for select using (true);
   create policy "auth insert" on public.events
     for insert to authenticated with check (true);
   create policy "auth delete" on public.events
     for delete to authenticated using (true);
   ```

   (Optional seed data:)
   ```sql
   insert into public.events (event_date, title, venue) values
     ('2026-06-14', 'The Vanishing Hour', 'The Rivoli Theatre · 8:00 PM'),
     ('2026-07-02', 'Close-Up Cabaret', 'Smoke & Mirrors Lounge · 7:30 PM'),
     ('2026-08-19', 'Summer Illusions Gala', 'Grand Hall · 6:00 PM');
   ```

### b. Lock down auth (CRITICAL)
3. **Authentication → Sign In / Providers → Email**: turn **OFF** "Allow new users to sign up."
   This is what makes shipping the anon key safe — only accounts you create can log in,
   so "authenticated" effectively means "Anthony."
4. **Authentication → Users → Add user**: create Anthony's login (email + password).

### c. Wire the keys
5. **Project Settings → API**: copy **Project URL** and the **anon / public** key
   (NOT the `service_role` key).
6. In `index.html`, set:
   ```js
   const SUPABASE_URL = 'https://xxxxxxxx.supabase.co';
   const SUPABASE_ANON_KEY = 'eyJhbGci...';   // anon/public key only
   ```

Anthony logs in via the **✦ Admin** link in the Events section, then adds/deletes shows.
Changes persist in the database and are visible to every visitor.

### d. Gallery image manager (Storage + table)
The gallery is also admin-managed (upload / drag-to-reorder / delete), reusing the same login.

1. **Storage → New bucket**: name it **`gallery`** and mark it **Public**.
2. **SQL Editor**, run:
   ```sql
   create table public.gallery_images (
     id uuid primary key default gen_random_uuid(),
     path text not null,
     alt text default '',
     sort_order int not null default 0,
     created_at timestamptz default now()
   );
   alter table public.gallery_images enable row level security;
   create policy "public read images" on public.gallery_images for select using (true);
   create policy "auth insert images" on public.gallery_images for insert to authenticated with check (true);
   create policy "auth update images" on public.gallery_images for update to authenticated using (true) with check (true);
   create policy "auth delete images" on public.gallery_images for delete to authenticated using (true);

   -- Storage object policies for the 'gallery' bucket
   create policy "public read gallery" on storage.objects for select using (bucket_id = 'gallery');
   create policy "auth upload gallery" on storage.objects for insert to authenticated with check (bucket_id = 'gallery');
   create policy "auth delete gallery" on storage.objects for delete to authenticated using (bucket_id = 'gallery');
   ```

Once logged in, scroll to **Gallery** → **✦ Upload images** (auto-resized in the browser),
drag tiles to reorder, hover an image and click **×** to delete. All changes are live for visitors.

### e. Testimonials (table)
The Reviews section is also admin-managed (add / edit / delete), reusing the same login.

1. **SQL Editor**, run:
   ```sql
   create table public.reviews (
     id uuid primary key default gen_random_uuid(),
     quote text not null,
     reviewer text not null,
     detail text default '',
     rating int not null default 5,
     sort_order int not null default 0,
     created_at timestamptz default now()
   );
   alter table public.reviews enable row level security;
   create policy "public read reviews" on public.reviews for select using (true);
   create policy "auth insert reviews" on public.reviews for insert to authenticated with check (true);
   create policy "auth update reviews" on public.reviews for update to authenticated using (true) with check (true);
   create policy "auth delete reviews" on public.reviews for delete to authenticated using (true);

   -- seed the three starter testimonials
   insert into public.reviews (quote, reviewer, detail, rating, sort_order) values
     ('Anthony had our entire dinner party speechless. The card work inches from our faces was simply impossible — we''re still arguing about how he did it.', 'Marisa Holloway', 'Private party · Portland, OR', 5, 0),
     ('We booked Anthony for our company gala and he completely stole the night. Polished, funny, and genuinely jaw-dropping. Worth every penny.', 'Devon Park', 'Corporate event · Seattle, WA', 5, 1),
     ('The stage show was pure wonder from start to finish. My kids were mesmerized and so were the adults. A magician who actually makes you believe again.', 'Elena Vásquez', 'Theater show · Vancouver, WA', 5, 2);
   ```

Logged in, scroll to **Reviews** → **+ Add testimonial**, or hover a card and click **✎** to edit or **×** to delete.

---

## 3. Deploy — Vercel

1. Go to https://vercel.com → **Add New → Project** → import `MightyJake/Anthony-Website`.
2. Framework preset: **Other** (no build command, no output dir — it's a static file).
3. Deploy. Add a custom domain under **Settings → Domains** if desired.

Pushing to the repo's default branch will auto-redeploy.

---

## Still to do (content)
- Replace the placeholder hero bio (marked `EDIT: Anthony's bio` in `index.html`).
- Upload real performance photos via the in-site **Gallery** admin (once section 2d is done).
