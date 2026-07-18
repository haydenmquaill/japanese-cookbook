# 食事プランナー — Meal Planner

A Japanese-inspired meal prep planner built for batch cooking, freezer-friendly recipes, and triathlon training nutrition. Built with vanilla HTML/CSS/JS, hosted on GitHub Pages, with Supabase for authentication and data storage.

## Features

- **63 Japanese recipes** — authentic everyday home-cooking staples covering rice bowls, noodles, yōshoku, simmered dishes, stir-fries, and hot pots. Almost all freezer-friendly; a few (raw-fish or last-minute-fresh dishes like Maguro Don and Ochazuke) aren't, and are flagged accordingly.
- **Smart meal scheduling** — pick your own meals each week from a searchable recipe list, filterable by Protein source, Format (rice/noodles/pasta), and Cook style (fried/grilled/simmered/stir-fried/soup/hot pot/cold/raw) dropdowns, plus Favourites/Auto-Pairing/Vegetarian toggles and a Rating filter — same filter set in both the Meals tab and the Schedule tab's meal picker. Meal B is auto-suggested based on compatibility ranking (protein variety, fat balance, recent history). Recipes can be marked `auto_pairing_eligible: false` to keep them in the recipe list without the algorithm ever auto-suggesting them — used for dishes that don't batch-cook well (hot pots, Omurice) or aren't freezer-friendly (Maguro Don, Ochazuke). They're still fully searchable and selectable by hand in the meal picker.
- **Relative week rotation, not calendar dates** — weeks are tracked purely as "this week" / "next week" / "in N weeks" relative to your current position. You advance to the next week yourself (whenever you actually cook, not on a fixed schedule), so cooking early or late never desyncs the app from reality.
- **Cook mode** — unified step-by-step cook session for both weekly meals. Rice is cooked first in its own rice-cooker phase (sized from both meals' combined servings, water at 1.2× the rice weight), then prep for both meals runs in parallel (marinating, chopping), then each meal is cooked sequentially. Every step shows a "You'll need" block with the exact scaled quantity of each ingredient it introduces — no more leaving cook mode to check the Ingredients tab. Built-in timers per step, session persistence across page reloads, and a screen wake lock so the phone doesn't sleep mid-session.
- **Grocery lists** — weekly lists aggregated by store (Woolworths, Asian aisle, Genki Mart, Fishmonger) with serving adjusters and persistent check-off state saved to Supabase.
- **Grocery readiness check** — home screen warning card showing unchecked items before a cook session, with a cook button guard if shopping isn't done.
- **Schedule locking** — current week locks once grocery shopping starts or a cook session begins. Future weeks always remain editable.
- **Live macro calculation** — protein, fat, carbs, and kcal calculated dynamically from a normalised ingredient database. All values scale with serving count using per-ingredient scale factors reflecting real cooking logic.
- **Consistent units** — small seasoning amounts (soy sauce, mirin, sake, oils, vinegar, miso paste, sugar) are always recorded in tbsp/tsp, matching how you'd actually measure them at home. Bulk liquids (stock, water, frying oil) stay in mL/L regardless of quantity — see [Adding new recipes](#adding-new-recipes).
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
-- Schedule — tracks current week rotation per user (purely relative, no calendar dates)
create table schedule (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
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

-- Recipe preferences — per-user personal settings for a recipe (rating, favourite, auto-pairing override, notes)
create table recipe_preferences (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  recipe_id text not null,
  auto_pairing_eligible boolean,
  rating int,
  favourite boolean not null default false,
  notes text,
  updated_at timestamptz default now(),
  unique(user_id, recipe_id)
);
alter table recipe_preferences enable row level security;
create policy "recipe_preferences_select" on recipe_preferences for select to authenticated using (auth.uid() = user_id);
create policy "recipe_preferences_insert" on recipe_preferences for insert to authenticated with check (auth.uid() = user_id);
create policy "recipe_preferences_update" on recipe_preferences for update to authenticated using (auth.uid() = user_id);
create policy "recipe_preferences_delete" on recipe_preferences for delete to authenticated using (auth.uid() = user_id);
```

`recipe_preferences.auto_pairing_eligible` is `null` by default, meaning "inherit the recipe's own `auto_pairing_eligible` from `data.json`" — set it explicitly to override that default per-recipe from the app (e.g. deciding you actually do want Sukiyaki in your weekly rotation after all, without editing `data.json`). `rating` is a personal 1-5 score; `notes` is your own tweaks/substitutions, kept separate from the recipe's shared, cookbook-level `notes` field.

**Existing installs:** the `schedule` table used to have a `cook_day` column (a fixed weekday preference) that's no longer used now that weeks are tracked purely relatively. It's safe to leave in place, or drop it with `alter table schedule drop column cook_day;`.

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
        "cook_style": "grilled",
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
          "ingredient_ids": ["ing_soy_sauce", "ing_mirin", "ing_sake_or_dry_sherry"],
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
- `unit` — keep this consistent by ingredient type, not just per-recipe: small seasoning amounts (soy sauce, mirin, sake, rice vinegar, sesame/vegetable oil as a seasoning, oyster sauce, honey, miso paste, mayo, sugar) should always be `tbsp` or `tsp`, matching how they're actually measured at home — never `ml`/`g` for these, even if that means a slightly unusual-looking amount like `5.5`. Bulk liquids (dashi/chicken/beef stock, water, frying oil) stay in `ml`/`l` regardless of quantity. Weight-based solids (meat, rice, vegetables) stay in `g`. Countable items (eggs, garlic cloves, onions) use `null`.
- `scale_factor` — controls how aggressively the ingredient scales when serving count changes. `1.0` = full linear (proteins, rice, vegetables), `0.75` = moderate (sauces, stock), `0.6` = light (sugar, honey), `0.5` = minimal (aromatics, oil), `0.3` = barely changes (salt, spices).
- `is_batch` — marks ingredients whose stored amount is a batch total (most sauce/condiment amounts).
- `step_type` — `prep` (phase 1, runs in parallel across both meals), `cook` (phase 2/3, sequential per meal), `portion` (end of each meal).
- `ingredient_ids` — the ingredient IDs (from this recipe's own `ingredients` array) first introduced/combined in this step. Drives the "You'll need" quantity block shown above each step, both in the standalone Steps tab and in cook mode. Don't re-list an ingredient in a later step once it's already part of a sauce/mixture made earlier — only tag the step where a home cook actually needs to go grab it.
- A step titled exactly `"Cook the rice"` is treated specially in cook mode: it's pulled out of the per-recipe prep flow and replaced with a single combined rice-cooker step (sized from both meals' servings, water at 1.2× the rice weight, run before anything else). Its own `content` in `data.json` is just generic rice-cooker instructions — the actual amount is computed live in cook mode from `ingredient_ids`.

**Recipe compatibility:**
- `protein_source` — used to avoid pairing two meals with the same protein. Also powers the Protein filter dropdown in Meals/Schedule.
- `fat_level` — `low`, `medium`, or `high` — used to balance weekly fat intake
- `has_veg` — whether the meal contains substantial vegetables
- `carb_heavy` — whether the meal is potato or noodle based (avoids two carb-heavy meals in one week)
- `cook_style` — the recipe's primary cooking technique: `fried`, `grilled`, `simmered`, `stir-fried`, `soup`, `hot pot`, `cold`, or `raw`. Pick whichever technique most *defines* the dish, not every technique a step happens to use (e.g. Tonkatsu Don has a simmered onion-dashi sauce too, but it's tagged `fried` because the katsu is what makes it that dish). Powers the Cook Style filter dropdown.

**Other recipe-level fields:**
- `auto_pairing_eligible` — omit for normal recipes (defaults to eligible). Set to `false` for recipes that shouldn't ever be auto-suggested by the pairing algorithm — not freezer-friendly, or don't batch-cook into weekly containers well (hot pots, Omurice's per-portion omelette). These recipes stay fully visible and searchable in the meal picker, just heavily deprioritised in ranking and excluded from the fully-automated `autoFillWeek`/`autoFillSchedule` flow.

### Meal compatibility ranking

When a user picks Meal A, compatible Meal B options are ranked by score:
- **−40** same protein source
- **+20** fat level complement (low pairs with high)
- **−10** same fat level
- **−20** both meals are carb heavy
- **−30** meal cooked in the last 4 weeks

### Adding new recipes

1. Add any new ingredients to the `ingredients` array with their nutritional data
2. Add the recipe to the `recipes` array referencing ingredient IDs, using `tbsp`/`tsp` for seasoning amounts and `ml`/`l` for bulk liquids (see Key fields above)
3. Set `tags_compat.cook_style` to whichever technique most defines the dish
4. Tag each step's `ingredient_ids` (see above) so cook mode can show per-step quantities
5. If the recipe isn't freezer-friendly or doesn't batch-cook well, set `auto_pairing_eligible: false`
6. Re-upload `data.json` to Supabase Storage — no code changes needed

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
| Maguro Don 🌿⛔ | 53.9g | 6.5g | 85.5g | 616 |
| Whiting Fry | 44.4g | 17.5g | 110.5g | 777 |
| Ebi Fry | 44.4g | 16.0g | 110.5g | 764 |
| Salmon Nanbanzuke | 39.3g | 24.6g | 99.9g | 778 |
| Agedashi Tofu Don | 23.0g | 14.6g | 101.3g | 629 |
| Kinoko Gohan | 18.1g | 9.2g | 91.4g | 521 |
| Yaki Tofu Don | 22.8g | 14.1g | 96.3g | 603 |
| Edamame and Egg Donburi | 30.7g | 16.4g | 95.0g | 650 |
| Vegetable Tempura Don | 21.1g | 9.7g | 127.0g | 680 |
| Tonkatsu Don | 49.2g | 25.0g | 116.9g | 889 |
| Kakuni | 38.4g | 54.0g | 97.4g | 1029 |
| Chashu Don | 37.3g | 53.8g | 92.8g | 1005 |
| Buta Kimchi | 40.1g | 21.0g | 87.0g | 697 |
| Buta Miso Itame | 40.6g | 21.0g | 95.9g | 735 |
| Tamago Don | 29.0g | 17.8g | 95.0g | 656 |
| Karei Nitsuke | 43.5g | 5.9g | 92.5g | 597 |
| Ochazuke 🌿⛔ | 38.8g | 22.3g | 83.1g | 688 |
| Miso Ramen | 42.5g | 24.8g | 88.4g | 764 |
| Curry Udon | 33.8g | 14.2g | 92.6g | 636 |
| Yaki Udon | 36.4g | 18.6g | 89.0g | 700 |
| Tsukimi Udon | 28.4g | 12.6g | 84.2g | 573 |
| Zaru Soba | 26.8g | 8.4g | 92.5g | 548 |
| Yakimeshi | 38.6g | 20.4g | 94.8g | 762 |
| Omurice ⛔ | 32.4g | 22.6g | 86.0g | 706 |
| Chicken Katsu Curry | 50.4g | 26.8g | 98.2g | 900 |
| Hambāgu | 44.2g | 28.4g | 90.5g | 828 |
| Korokke | 33.6g | 24.5g | 92.4g | 758 |
| Buri Daikon | 40.8g | 15.6g | 90.2g | 674 |
| Chikuzenni | 37.4g | 13.8g | 89.6g | 636 |
| Sukiyaki ⛔ | 41.6g | 30.2g | 88.4g | 838 |
| Chinjao Rosu | 39.2g | 16.4g | 88.8g | 668 |
| Yasai Itame Don | 30.5g | 16.8g | 92.4g | 654 |
| Gyoza to Gohan | 36.8g | 22.4g | 90.6g | 730 |
| Naporitan | 29.8g | 21.6g | 95.4g | 716 |
| Ebi to Kinoko Wafu Pasta | 38.2g | 17.4g | 90.8g | 692 |
| Hayashi Rice | 37.6g | 18.2g | 93.4g | 704 |
| Mizutaki ⛔ | 44.8g | 14.6g | 82.4g | 654 |

🌿 Fresh only — do not freeze.
⛔ Not included in auto-pairing suggestions by default (`auto_pairing_eligible: false`) — still fully browsable and manually selectable. Maguro Don and Ochazuke are also excluded for this reason, in addition to being fresh-only.
