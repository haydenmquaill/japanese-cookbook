# 食事プランナー — Meal Planner

A Japanese-inspired meal prep planner built for batch cooking, freezer-friendly recipes, and triathlon training nutrition. Built with vanilla HTML/CSS/JS, hosted on GitHub Pages, with Supabase for authentication and data storage.

## Features

- **16 Japanese recipes** — authentic home-cooking staples from teriyaki chicken to oden, all freezer-friendly and batch-cooked in servings of 7
- **8-week rotation** — nutritionally balanced meal pairings that cycle through proteins, fats, and micronutrient variety
- **Cook mode** — unified step-by-step cook session for both weekly meals, with parallel prep phase and sequential wok cooking, built-in timers, and session persistence
- **Grocery lists** — weekly lists aggregated by store (Woolworths, Asian aisle, Genki Mart, Fishmonger) with serving adjusters and persistent check-off state
- **Grocery readiness check** — home screen warning card showing unchecked items before a cook session, with a cook button guard if shopping isn't done
- **Live macro calculation** — protein, fat, carbs, and kcal calculated dynamically from ingredient nutritional data, scaling with serving count
- **Smart ingredient scaling** — each ingredient has a `scale_factor` reflecting real cooking logic (proteins scale fully, sauces scale moderately, aromatics barely change)
- **Cook day flavour text** — contextual home screen messaging based on how far away your next cook day is
- **Multi-user** — Supabase Auth with per-user schedule, grocery state, and cook session persistence
- **Three themes** — Sumi (dark ink), Sakura (pink), Ai (indigo gold) with dark/light mode toggle

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML, CSS, JavaScript |
| Hosting | GitHub Pages |
| Auth | Supabase Auth (email + password) |
| Database | Supabase Postgres |
| Recipe data | `data.json` in Supabase Storage |

## Project Structure

```
/
├── index.html          # Single-page app
└── README.md
```

Recipe data, ingredient nutrition, store sourcing, and meal pairings all live in `data.json` stored in a private Supabase Storage bucket (`meal-planner`).

## Supabase Setup

### Storage

Create a private bucket called `meal-planner` and upload `data.json`. Add a SELECT policy for authenticated users.

### Database Tables

Run the following in the Supabase SQL Editor:

```sql
-- Schedule — tracks cook day preference and current week per user
create table schedule (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  cook_day int not null default 6,
  current_week int not null default 1,
  last_cooked_at timestamptz,
  updated_at timestamptz default now(),
  unique(user_id)
);
alter table schedule enable row level security;
create policy "schedule_select" on schedule for select to authenticated using (auth.uid() = user_id);
create policy "schedule_insert" on schedule for insert to authenticated with check (auth.uid() = user_id);
create policy "schedule_update" on schedule for update to authenticated using (auth.uid() = user_id);

-- Cook log — history of cook sessions
create table cook_log (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  week_number int not null,
  meal_a_id text not null,
  meal_b_id text not null,
  meal_a_servings int not null default 7,
  meal_b_servings int not null default 7,
  cooked_at timestamptz default now()
);
alter table cook_log enable row level security;
create policy "cook_log_select" on cook_log for select to authenticated using (auth.uid() = user_id);
create policy "cook_log_insert" on cook_log for insert to authenticated with check (auth.uid() = user_id);

-- Grocery state — persists checked items and serving counts
create table grocery_state (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  checked_items jsonb not null default '{}',
  servings jsonb not null default '{}',
  updated_at timestamptz default now(),
  unique(user_id)
);
alter table grocery_state enable row level security;
create policy "grocery_state_select" on grocery_state for select to authenticated using (auth.uid() = user_id);
create policy "grocery_state_insert" on grocery_state for insert to authenticated with check (auth.uid() = user_id);
create policy "grocery_state_update" on grocery_state for update to authenticated using (auth.uid() = user_id);

-- Cook session — persists active cook mode progress
create table cook_session (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  meal1_id text not null,
  meal2_id text not null,
  current_step int not null default 0,
  total_steps int not null default 0,
  week_number int not null default 1,
  completed boolean not null default false,
  updated_at timestamptz default now(),
  unique(user_id)
);
alter table cook_session enable row level security;
create policy "cook_session_select" on cook_session for select to authenticated using (auth.uid() = user_id);
create policy "cook_session_insert" on cook_session for insert to authenticated with check (auth.uid() = user_id);
create policy "cook_session_update" on cook_session for update to authenticated using (auth.uid() = user_id);
```

### Adding Users

Users are added via the Supabase Dashboard under Authentication → Users → Add user. Public signups are disabled by default.

## Recipe Data Format

`data.json` contains all recipes, pairings, and store metadata. Each recipe follows this structure:

```json
{
  "id": "teriyaki-chicken",
  "title": "Teriyaki Chicken",
  "title_jp": "照り焼きチキン",
  "protein_per_serving": 55.9,
  "fat_per_serving": 6.9,
  "carbs_per_serving": 93.5,
  "calories_per_serving": 660,
  "cook_time_mins": 40,
  "ingredients": [
    {
      "name": "Chicken breast (raw weight)",
      "amount": 1400,
      "unit": "g",
      "store": "woolworths",
      "scale_factor": 1.0,
      "protein_per_100g": 23.0,
      "fat_per_100g": 1.9,
      "carbs_per_100g": 0.0,
      "is_batch": true
    }
  ],
  "steps": [
    {
      "id": "s1",
      "title": "Make teriyaki sauce",
      "step_type": "prep",
      "content": "...",
      "timer_seconds": 240
    }
  ]
}
```

### Key fields

- **`amount`** — all ingredient amounts are batch totals for 7 servings
- **`scale_factor`** — controls how aggressively an ingredient scales when serving count changes. `1.0` = full linear scaling (proteins, rice), `0.75` = moderate (sauces, stock), `0.5` = minimal (aromatics, oil), `0.3` = barely changes (salt, spices)
- **`step_type`** — `prep` (parallel phase 1), `cook` (sequential wok phase), or `portion` (end of meal)
- **`is_batch`** — marks ingredients whose amounts are batch totals rather than per-serving

### Meal pairings

The rotation is defined as a `pairings` array in `data.json`. Each entry specifies `week`, `meal_a`, and `meal_b` by recipe ID. 16 weeks of unique pairings before anything repeats.

## Adding New Recipes

Add a new entry to the `recipes` array in `data.json` following the format above, then add it to the `pairings` array. Re-upload `data.json` to Supabase Storage — no code changes needed.

## Nutrition Data

Macro values are calculated dynamically at runtime from per-ingredient nutritional data (USDA FoodData Central). All amounts in `data.json` are raw/uncooked weights. Cooked weights and water loss are not accounted for — values are estimates suitable for meal planning, not clinical nutrition.

## Mobile

The app is designed mobile-first. To install as a home screen app on iOS: open in Safari → Share → Add to Home Screen. The app runs full-screen without browser UI.
