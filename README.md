# 食事プランナー — Meal Planner

A Japanese-inspired meal prep planner built for batch cooking, freezer-friendly recipes, and triathlon training nutrition. Built with vanilla HTML/CSS/JS, hosted on GitHub Pages, with Supabase for authentication and data storage.

## Features

- **30 Japanese recipes** — authentic home-cooking staples from teriyaki chicken to chicken nanban, covering a wide range of proteins, techniques, and regional dishes. All freezer-friendly except Maguro Don (raw tuna).
- **Smart meal scheduling** — pick your own meals each week from a searchable, filterable recipe list. Meal B is auto-suggested based on compatibility ranking (protein variety, fat balance, recent history).
- **15-week rotation** — enough unique pairings before anything repeats.
- **Cook mode** — unified step-by-step cook session for both weekly meals. Phase 1 runs all prep steps from both meals in parallel (rice, marinating, chopping). Phase 2 and 3 cook each meal sequentially on the wok. Built-in timers per step, session persistence across page reloads.
- **Grocery lists** — weekly lists aggregated by store (Woolworths, Asian aisle, Genki Mart, Fishmonger) with serving adjusters and persistent check-off state saved to Supabase.
- **Grocery readiness check** — home screen warning card showing unchecked items before a cook session, with a cook button guard if shopping isn't done.
- **Schedule locking** — current week locks once grocery shopping starts or a cook session begins. Future weeks always remain editable.
- **Live macro calculation** — protein, fat, carbs, and kcal calculated dynamically from a normalised ingredient database. All values scale with serving count using per-ingredient scale factors reflecting real cooking logic.
- **Cook day flavour text** — contextual home screen messaging based on how far away your next cook day is.
- **Multi-user** — Supabase Auth with per-user schedule, grocery state, cook session, and meal pairings.
- **Three themes** — Sumi (dark ink), Sakura (pink), Ai (indigo gold) with dark/light mode toggle.
- **Mobile-first** — designed for one-handed use in the kitchen. Installable as a home screen app on iOS and Android.

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

All recipe data, ingredient nutrition, store sourcing, and compatibility tags live in `data.json` stored in a private Supabase Storage bucket (`meal-planner`).

## Supabase Setup

### Storage

Create a private bucket called `meal-planner` and upload `data.json`. Add a SELECT policy for authenticated users:

```sql
create policy "storage_select" on storage.objects
  for select to authenticated
  using (bucket_id = 'meal-planner');
```

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

-- User pairings — planned meal pairings per week per user
create table user_pairings (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  week_number int not null,
  meal_a_id text not null,
  meal_b_id text not null,
  created_at timestamptz default now(),
  unique(user_id, week_number)
);
alter table user_pairings enable row level security;
create policy "user_pairings_select" on user_pairings for select to authenticated using (auth.uid() = user_id);
create policy "user_pairings_insert" on user_pairings for insert to authenticated with check (auth.uid() = user_id);
create policy "user_pairings_update" on user_pairings for update to authenticated using (auth.uid() = user_id);
create policy "user_pairings_delete" on user_pairings for delete to authenticated using (auth.uid() = user_id);
```

### Adding Users

Users are added via the Supabase Dashboard under Authentication → Users → Add user. Public signups are disabled by default.

## data.json Structure

`data.json` version 2.0 uses a normalised structure — ingredients are defined once in a master list and referenced by ID in each recipe.

```json
{
  "version": "2.0",
  "ingredients": [
    {
      "id": "ing_chicken_breast_raw_weight",
      "name": "Chicken breast (raw weight)",
      "protein_per_100g": 23.0,
      "fat_per_100g": 1.9,
      "carbs_per_100g": 0.0,
      "store": "woolworths",
      "unit_g": null
    }
  ],
  "recipes": [
    {
      "id": "teriyaki-chicken",
      "title": "Teriyaki Chicken",
      "title_jp": "照り焼きチキン",
      "description": "...",
      "protein_per_serving": 55.9,
      "fat_per_serving": 6.9,
      "carbs_per_serving": 93.5,
      "calories_per_serving": 660,
      "cook_time_mins": 40,
      "difficulty": "easy",
      "freezer_friendly": true,
      "tags": ["chicken", "rice"],
      "tags_compat": {
        "protein_source": "chicken",
        "fat_level": "low",
        "has_veg": true,
        "carb_heavy": false
      },
      "ingredients": [
        {
          "ingredient_id": "ing_chicken_breast_raw_weight",
          "amount": 1400,
          "unit": "g",
          "scale_factor": 1.0,
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
      ],
      "notes": "...",
      "store_notes": { "woolworths": [...], "genki": [...] }
    }
  ],
  "stores": {
    "woolworths": { "label": "Woolworths / Coles", "emoji": "🛒" },
    "woolworths-asian": { "label": "Woolworths / Coles (Asian aisle)", "emoji": "🛒" },
    "genki": { "label": "Genki Mart / Asian Grocer", "emoji": "🏮" },
    "fishmonger": { "label": "Fish Market / Fishmonger", "emoji": "🐟" }
  }
}
```

### Key fields

**Ingredients (master list):**
- `unit_g` — grams per countable unit (e.g. 55g per egg, 5g per garlic clove). Omit for items measured by weight or volume.

**Recipe ingredients:**
- `amount` — batch total for 7 servings. All amounts are pre-scaled to a base of 7.
- `scale_factor` — controls how aggressively the ingredient scales when serving count changes. `1.0` = full linear (proteins, rice, vegetables), `0.75` = moderate (sauces, stock), `0.6` = light (sugar, honey), `0.5` = minimal (aromatics, oil), `0.3` = barely changes (salt, spices).
- `is_batch` — marks ingredients whose stored amount is a batch total (most sauce/condiment amounts).
- `step_type` — `prep` (phase 1, runs in parallel across both meals), `cook` (phase 2/3, sequential per meal), `portion` (end of each meal).

**Recipe compatibility:**
- `protein_source` — used to avoid pairing two meals with the same protein
- `fat_level` — `low`, `medium`, or `high` — used to balance weekly fat intake
- `has_veg` — whether the meal contains substantial vegetables
- `carb_heavy` — whether the meal is potato or noodle based (avoids two carb-heavy meals in one week)

### Meal compatibility ranking

When a user picks Meal A, compatible Meal B options are ranked by score:
- **−40** same protein source
- **+20** fat level complement (low pairs with high)
- **−10** same fat level
- **−20** both meals are carb heavy
- **−30** meal cooked in the last 4 weeks

### Adding new recipes

1. Add any new ingredients to the `ingredients` array with their nutritional data
2. Add the recipe to the `recipes` array referencing ingredient IDs
3. Re-upload `data.json` to Supabase Storage — no code changes needed

## Nutrition data

Macro values are sourced from USDA FoodData Central. All amounts are raw/uncooked weights. Cooked weights and water loss are not accounted for — values are estimates suitable for meal planning. Frying oil absorption is estimated at 30ml per batch rather than the full volume used.

## Mobile installation

Open in Safari → Share → Add to Home Screen. The app runs full-screen without browser UI when launched from the home screen.

## Recipes

| Recipe | Protein | Fat | Carbs | kcal |
|---|---|---|---|---|
| Teriyaki Chicken | 55.9g | 6.9g | 93.5g | 660 |
| Japanese Chicken Curry | 55.2g | 7.2g | 93.9g | 661 |
| Salmon Miso Bowl | 53.8g | 33.3g | 88.1g | 867 |
| Tonjiru | 27.1g | 11.0g | 91.3g | 573 |
| Gyudon | 37.3g | 17.0g | 92.7g | 673 |
| Simmered Tofu & Vegetables | 17.4g | 9.1g | 90.8g | 515 |
| Oyakodon | 44.2g | 15.0g | 92.2g | 681 |
| Butadon | 29.0g | 45.7g | 94.8g | 906 |
| Nikujaga | 35.2g | 12.1g | 109.9g | 689 |
| Shake Shioyaki | 42.5g | 24.4g | 85.0g | 730 |
| Saba Miso | 38.0g | 22.1g | 91.9g | 718 |
| Ebi Chili | 38.4g | 8.8g | 86.5g | 579 |
| Tori Soboro | 44.5g | 22.4g | 92.2g | 748 |
| Oden | 25.8g | 11.0g | 93.3g | 575 |
| Hijiki Gohan | 28.7g | 11.6g | 86.9g | 567 |
| Yakitori Donburi | 38.6g | 13.3g | 93.6g | 648 |
| Jingisukan | 52.6g | 27.3g | 94.4g | 834 |
| Yakisoba | 41.9g | 19.4g | 67.1g | 611 |
| Chicken Karaage | 50.9g | 25.1g | 100.0g | 830 |
| Buta Shogayaki | 38.7g | 16.3g | 89.2g | 658 |
| Sanma Shioyaki | 39.3g | 26.8g | 85.3g | 740 |
| Mapo Tofu | 35.2g | 22.6g | 86.1g | 689 |
| Niku Udon | 32.0g | 10.0g | 54.8g | 437 |
| Chicken Nanban | 55.3g | 35.1g | 97.9g | 929 |
| Kani Tamago Don | 27.7g | 11.5g | 97.7g | 605 |
| Buri Teriyaki | 42.1g | 17.2g | 92.2g | 692 |
| Maguro Don 🌿 | 53.9g | 6.5g | 85.5g | 616 |
| Whiting Fry | 44.4g | 17.5g | 110.5g | 777 |
| Ebi Fry | 44.4g | 16.0g | 110.5g | 764 |
| Salmon Nanbanzuke | 39.3g | 24.6g | 99.9g | 778 |

🌿 Fresh only — do not freeze.
