# Ethio Outage Reporter

A lightweight, zero-cost Telegram Mini App for crowdsourcing and visualizing local **power**, **water**, and **Wi-Fi** outages on an interactive heatmap.

**[Launch Telegram Mini App](https://t.me/ethio_outage_report_bot/app)** · `@ethio_outage_report_bot`

![Ethio Outage Reporter Screenshot](https://i.ibb.co/TDwNwpyj/Screenshot-2026-11-22-150613.png)

---

## Features

- **Telegram Mini App Integration:** Opens directly inside Telegram (`@ethio_outage_report_bot`) using the Telegram WebApp SDK to read the user's Telegram ID without forcing separate sign-up forms.
- **Interactive Outage Heatmap:** Displays recent reports (last 24 hours) over **OpenStreetMap** using **Leaflet.js** and **Leaflet.heat**, with category filtering (**All Outages**, **Electricity**, **Water**, **Wi-Fi**) and a **Street / Satellite** layer switcher.
- **Per-Category Cooldown Enforcement:** Enforces a **1 report per category per Telegram user every 12 hours** at the database level via a PostgreSQL `BEFORE INSERT` trigger, disabling already-reported categories in the UI.
- **$0 Hosting Stack:** Built with static frontend markup, free map tiles, and a free-tier Supabase PostgreSQL database with Row Level Security (RLS).

---

## Tech Stack

- **Frontend:** HTML, CSS, JavaScript, [Telegram WebApp SDK](https://core.telegram.org/bots/webapps)
- **Mapping:** [Leaflet.js](https://leafletjs.com/), [Leaflet.heat](https://github.com/Leaflet/Leaflet.heat), [OpenStreetMap](https://www.openstreetmap.org/), [Esri World Imagery](https://www.esri.com/)
- **Database & Backend:** [Supabase](https://supabase.com/) (PostgreSQL, Row Level Security, SQL trigger)
- **Deployment:** Static hosting (e.g., GitHub Pages) linked via `@BotFather`

---

## How It Works

1. **User Context & GPS:** When opened inside Telegram, the app initializes the Telegram WebApp context, retrieves `telegram_user_id`, and requests browser geolocation (`navigator.geolocation`) to center the map and pin the report.
2. **Database Cooldown Check:** Submitting a report inserts `(telegram_user_id, type, lat, lng)` into Supabase. A PostgreSQL trigger (`enforce_outage_cooldown`) checks if the same user already submitted that category within the past 12 hours and rejects duplicates.
3. **Heatmap Rendering:** Queries recent outage rows from Supabase and renders intensity clusters on the map.

---

## Setup & Database Schema

1. Create a free project on **Supabase** and run the table, index, trigger, and RLS policies in the **SQL Editor**:

   ```sql
   create table public.outages (
     id bigint generated always as identity primary key,
     telegram_user_id bigint not null,
     type text not null check (type in ('electricity', 'water', 'wifi')),
     lat double precision not null,
     lng double precision not null,
     created_at timestamptz default now()
   );

   create index idx_outages_user_type_time
     on public.outages (telegram_user_id, type, created_at desc);

   create index idx_outages_created_at
     on public.outages (created_at desc);

   create or replace function public.enforce_outage_cooldown()
   returns trigger as $$ begin if exists ( select 1 from public.outages where telegram_user_id = new.telegram_user_id and type = new.type and created_at > now() - interval '12 hours' ) then raise exception 'COOLDOWN_ACTIVE: You already reported a \% outage in the last 12 hours.', new.type; end if; return new; end; $$ language plpgsql;

   create trigger trg_enforce_outage_cooldown
   before insert on public.outages
   for each row execute function public.enforce_outage_cooldown();

   alter table public.outages enable row level security;

   create policy "Public read" on public.outages for select using (true);
   create policy "Public insert" on public.outages for insert with check (telegram_user_id > 0);

   grant usage on schema public to anon;
   grant select, insert on table public.outages to anon;
   grant usage, select on all sequences in schema public to anon;
