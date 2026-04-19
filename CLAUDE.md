# State Assets Visualization — Victoria 3 Mod

## What This Mod Does

Adds five tables across two Budget panel tabs:

### Assets Tab (existing)

#### Table 1: State-Owned Industry
- Total government-owned building **levels** (domestic only)
- **Privatization value** — what the Investment Pool would pay to buy them (per the game's own formula)
- Historical snapshots: current, start of year, previous year, two years ago, five years ago
- Year-over-year deltas for both levels and value

#### Table 2: Government Net Worth
- **Treasury** — the country's gold reserves minus debt principal
- **Net Worth** — treasury + state asset privatization value from Table 1
- Same historical snapshot rows and deltas as Table 1

### Capital Tab (v1.3)

#### Table 3: National Industry Levels
- 3 columns: **Domestic** | **Foreign** | **Total**
- Counts ALL nationally-owned building **levels** (gov + private + self combined) using `levels_owned_by_country`
- Domestic: buildings in ROOT's states; Foreign: buildings in other countries (pre-filtered by `gdp_ownership_ratio(ROOT) > 0`)
- Uses **staircase approach**: 200 independent if-blocks checking `levels_owned_by_country >= N` for N=1..200
- Historical snapshots with absolute deltas on levels

#### Table 4: National Industry Value
- 3 columns: **Domestic Value** | **Foreign Value** | **Total Asset Value**
- Values derived from Table 3 levels × cost tier × 250
- Total Asset Value = domestic + foreign (derived as `nat_sum_value`)
- Historical snapshots with percentage deltas

#### Table 5: National Capital
- 4 columns: **Total Asset Value** | **Treasury** | **Inv. Pool** | **Sum**
- Total Asset Value from Table 4; Treasury = gold reserves minus debt principal; Inv. Pool = investment pool balance
- Sum = total asset value + treasury + investment pool
- Compact "Jan YYYY" row labels to fit 4 columns
- Historical snapshots with percentage deltas

## Architecture

Eight files, each with a clear responsibility:

| File | Purpose |
|---|---|
| `.metadata/metadata.json` | Mod identity, version, tags. `replace_paths` includes `gui/budget_panel.gui` (full override). |
| `common/script_values/state_assets_values.txt` | Two script values evaluated in **building scope**: `inv_gov_owned_levels_value` (level × ownership fraction, rounded) and `inv_gov_owned_value_per_building` (cost tier × 250 × rounded levels). Used by Assets tab (Tables 1-2) only. |
| `common/scripted_effects/state_assets_effects.txt` | Country-scope effects: domestic gov counting (Assets tab), national asset counting with staircase (Capital tab), treasury/ipool capture, net worth and national capital computation, delta computation, yearly snapshot rolling (5-deep chain), first-tick seed with v1.1/v1.3b/v1.3c upgrade support. |
| `common/scripted_effects/state_assets_staircase.txt` | Building-scope effects for Capital tab: `inv_staircase_nat_levels_effect` (200 if-blocks counting `levels_owned_by_country >= N`) and `inv_compute_nat_value_for_building_effect` (cost tier × 250 × staircase levels). |
| `common/on_actions/state_assets_on_actions.txt` | Hooks into `on_monthly_pulse_country`. Snapshot rolling triggered by `month = 0` check inside monthly pulse (not yearly pulse, which is staggered). |
| `gui/budget_panel.gui` | Full copy of vanilla budget panel. Assets tab has Tables 1-2. Capital tab (4th tab) has Tables 3-5. |
| `localization/english/state_assets_l_english.yml` | Localization keys for headers, column names, row labels. **Must have UTF-8 BOM** (`EF BB BF`). |
| `CLAUDE.md` | This file. |

## Ownership Trigger Semantics (empirically verified)

Building-level ownership has **two independent dimensions**:

### Dimension 1: Type of Owner

| Trigger | Measures | Range |
|---|---|---|
| `country_ownership_fraction` | **Government** ownership — any government (domestic or foreign) | 0–1 |
| `private_ownership_fraction` | **Private** ownership — any private investor (domestic or foreign), excluding self | 0–1 |
| `self_ownership_fraction` | **Self** ownership — workforce/unclaimed (always local) | 0–1 |

These three sum to 1.0 for any building.

### Dimension 2: Location of Owner

| Trigger | Measures |
|---|---|
| `levels_owned_by_country = { target = c:TAG value >= N }` | Total **levels** owned by target country's nationals (gov + private + self combined) |
| `fraction_of_levels_owned_by_country = { target = c:TAG value >= N }` | **Share** of levels owned by target country — equivalent to `levels_owned_by_country(TAG) / total_levels` |

These two dimensions are **fully independent**. Type tells you gov vs private vs self. Location tells you which country's nationals. They do NOT cross-filter — there is no trigger for "domestic government-owned fraction" directly.

## Key Engine Concepts

- **`gdp_ownership_ratio(country)`** — C++ getter returning fraction of this country's GDP owned by specified country. Used ONLY as a pre-filter (`> 0`) to skip countries where ROOT has no investment. NOT used for actual value calculation.
- **`investment_pool`** — country-scope value, returns the investment pool balance. Usable in `set_variable` value fields.
- **`gold_reserves`** — country-scope value, returns the treasury amount. Can be negative (debt). Usable in `set_variable` value fields.
- **`principal`** — country-scope value, returns the debt principal amount.
- **`every_scope_building`** — iterates all buildings in the country's states. Inside this loop, `ROOT` is the country, `PREV` is the building.
- **`every_country`** — iterates all alive countries. Used with `NOT = { this = scope:inv_root }` to skip ROOT.
- **`is_building_group`** — matches parent groups (e.g., `bg_manufacturing` catches `bg_light_industry`, `bg_heavy_industry`, `bg_military_industry`).
- **Port vs Railway** — both belong to `bg_private_infrastructure` but have different construction costs. Railway is checked by `is_building_type` before the medium-cost group check.
- **`round = yes`** — available in script values, rounds to nearest integer (0.5 rounds up).
- **`|K` format** — GUI format suffix for currency abbreviation (150000 -> "150K"). Used with `@money!` icon prefix.
- **`|+=K` format** — signed currency delta format (+150K / -50K).
- **`text` vs `raw_text`** — `text` resolves localization keys; `raw_text` renders literally (use for dynamic `[expressions]`).
- **`month` trigger** — 0-indexed (0 = January, 11 = December). Used to detect January inside monthly pulse for snapshot rolling.
- **Variables** — stored per-country via `set_variable`/`change_variable`. Accessed in GUI as `GetPlayer.MakeScope.Var('name').GetValue`. In localization, use `GetPlayer.MakeScope.Var('name').GetValue` (NOT `Root.MakeScope`).
- **Tab system** — vanilla `tab_buttons` template has 5 slots. Budget uses 4 (Overview, States, Assets, Capital). `InformationPanel.SelectTab('name')` is string-based.

## Privatization Value Formula

`required_construction x PRIVATIZATION_PER_LEVEL_COST (250) x government_owned_levels`

Construction cost tiers (from `game/common/script_values/building_values.txt`):
- **very_low (100)**: trade centers (`bg_trade`)
- **low (200)**: agriculture, ranching, plantations, logging, rubber, fishing, whaling
- **medium (400)**: mines, oil, ports, arts academies, power plants
- **high (600)**: light industry (`bg_light_industry`)
- **very_high (800)**: heavy industry, military industry, railways

## Staircase Level Counting (Capital Tab)

`levels_owned_by_country` and `fraction_of_levels_owned_by_country` are **comparator triggers only** — they cannot be used as scalar multipliers in script values (empirically verified: always returns 0 when used as a value). The staircase approach works around this:

- 200 independent `if` blocks (NOT `else_if`), each checking `levels_owned_by_country = { target = ROOT value >= N }` for N = 1..200
- Each matching `if` adds 1 to `temp_bldg_nat_levels` on ROOT
- Result: exact level count for ROOT's nationals in that building (capped at 200)
- Always runs all 200 checks per building (no short-circuit), but only runs while Capital tab is visible

## Foreign Building Iteration

National asset counting uses two passes:
1. **Domestic**: `every_scope_building` in ROOT's states
2. **Foreign**: `every_country { NOT this=scope:inv_root, gdp_ownership_ratio(scope:inv_root)>0 } { every_scope_building { ... } }`

The `gdp_ownership_ratio` filter is O(countries) — efficient pre-filter that skips countries with zero ROOT investment. Inside each building, `levels_owned_by_country = { target = ROOT value >= 1 }` confirms ROOT has ownership, then the staircase counts exact levels. ROOT scope persists through nested `every_country`/`every_scope_building` chains. Foreign accumulation uses `scope:inv_root` to write back to the player country's variables.

## Known Engine Limitations

- **`fraction_of_levels_owned_by_country` cannot be used as a scalar multiplier.** Empirically verified — always returns 0 when used in `set_variable` value fields or script value expressions. Only works as a comparator trigger (`value >= N`). This forced the staircase approach for level counting.
- **No cross-dimensional trigger.** The two ownership dimensions (type × location) are independent. There is no single trigger for "domestic government-owned fraction" or "foreign private-owned fraction".
- **`required_construction` is not a building-scope trigger.** Must use an if/else_if chain on `is_building_group`/`is_building_type` to determine cost tier.
- **Urban centers (`bg_service`) are always workforce-owned** — never government-owned, excluded from counting.

## Variables Used (country scope)

### State-Owned Industry (Table 1, Assets tab)

| Variable | Updated | Purpose |
|---|---|---|
| `gov_levels_current` | Monthly | Current total government-owned levels |
| `gov_value_current` | Monthly | Current total privatization value |
| `gov_levels_year_0/1/2/3/4/5` | Yearly | Snapshots (year_3/4 are intermediate, only year_5 displayed) |
| `gov_value_year_0/1/2/3/4/5` | Yearly | Value snapshots (same structure) |
| `gov_levels_delta_y1/2/5` | Monthly | year_0 minus year_1/2/5 |
| `gov_value_delta_y1/2/5` | Monthly | percentage: (year_0/year_X - 1) * 100 |

### Treasury & Net Worth (Table 2, Assets tab)

| Variable | Updated | Purpose |
|---|---|---|
| `treasury_current` | Monthly | Current gold reserves minus principal |
| `treasury_year_0/1/2/3/4/5` | Yearly | Treasury snapshots |
| `treasury_delta_y1/2/5` | Monthly | percentage deltas |
| `total_current` | Monthly | Net worth = treasury + gov_value |
| `total_year_0/1/2/5` | Yearly | Net worth snapshots (derived) |
| `total_delta_y1/2/5` | Monthly | percentage deltas |

### National Industry Levels (Table 3, Capital tab)

| Variable | Updated | Purpose |
|---|---|---|
| `nat_dom_levels_current` | Monthly | Domestic nationally-owned levels |
| `nat_for_levels_current` | Monthly | Foreign nationally-owned levels |
| `nat_sum_levels_current` | Monthly | Sum = domestic + foreign (derived) |
| `nat_dom_levels_year_0/1/2/3/4/5` | Yearly | Domestic level snapshots |
| `nat_for_levels_year_0/1/2/3/4/5` | Yearly | Foreign level snapshots |
| `nat_sum_levels_year_0/1/2/5` | Yearly | Sum level snapshots (derived, no year_3/4) |
| `nat_dom_levels_delta_y1/2/5` | Monthly | Domestic: year_0 minus year_1/2/5 |
| `nat_for_levels_delta_y1/2/5` | Monthly | Foreign: year_0 minus year_1/2/5 |
| `nat_sum_levels_delta_y1/2/5` | Monthly | Sum: year_0 minus year_1/2/5 |

### National Industry Value (Table 4, Capital tab)

| Variable | Updated | Purpose |
|---|---|---|
| `nat_dom_value_current` | Monthly | Domestic nationally-owned asset value |
| `nat_for_value_current` | Monthly | Foreign nationally-owned asset value |
| `nat_sum_value_current` | Monthly | Sum = domestic + foreign (derived) |
| `nat_dom_value_year_0/1/2/3/4/5` | Yearly | Domestic value snapshots |
| `nat_for_value_year_0/1/2/3/4/5` | Yearly | Foreign value snapshots |
| `nat_sum_value_year_0/1/2/5` | Yearly | Sum value snapshots (derived, no year_3/4) |
| `nat_dom_value_delta_y1/2/5` | Monthly | percentage deltas |
| `nat_for_value_delta_y1/2/5` | Monthly | percentage deltas |
| `nat_sum_value_delta_y1/2/5` | Monthly | percentage deltas |

### National Capital (Table 5, Capital tab)

| Variable | Updated | Purpose |
|---|---|---|
| `ipool_current` | Monthly | Current investment pool balance |
| `ipool_year_0/1/2/3/4/5` | Yearly | Investment pool snapshots |
| `ipool_delta_y1/2/5` | Monthly | percentage deltas |
| `natcap_current` | Monthly | National capital = nat_sum_value + treasury + ipool |
| `natcap_year_0/1/2/5` | Yearly | National capital snapshots (derived) |
| `natcap_delta_y1/2/5` | Monthly | percentage deltas |
| `temp_bldg_nat_levels` | Per-building | Staircase result: ROOT's levels in current building (transient) |
| `temp_bldg_nat_value` | Per-building | Value for current building: cost tier × 250 × levels (transient) |

### Snapshot Infrastructure

| Variable | Purpose |
|---|---|
| `snap_year_0/1/2/3/4/5` | Year labels for each snapshot slot (shared across all tables) |
| `snap_age_1/2/5` | Years since snapshot was taken (for GUI dash/data visibility) |

### Flags

| Variable | Purpose |
|---|---|
| `gov_initialized` | First-tick seed flag (new game: populates all snapshots from current) |
| `gov_v11_initialized` | v1.1 upgrade seed flag (existing v1.0 save: populates treasury/net worth vars) |
| `gov_v13_initialized` | v1.3 upgrade seed flag (set by v1.3b seed for compat) |
| `gov_v13b_initialized` | v1.3b upgrade seed flag (existing v1.0-1.2 or old v1.3 save: populates split dom/for national asset/capital vars) |
| `gov_v13c_initialized` | v1.3c upgrade seed flag (existing v1.3b save: populates `nat_sum_value` snapshots) |

## Snapshot Rolling Chain

Each January: `year_5 <- year_4 <- year_3 <- year_2 <- year_1 <- year_0 <- current`

Intermediate slots (year_3, year_4) are stored but not displayed in the GUI. They exist solely to make the 5-year rolling chain work correctly.

Derived snapshots (total, natcap) only need year_0/1/2/5 — computed from their component variables after rolling.

## Building Filters (what gets counted)

### Assets Tab (government-owned, Tables 1-2)
```
is_government_funded = no          # not govt-funded (barracks etc.)
is_subsistence_building = no       # not subsistence
NOT bg_service                     # not urban centers
NOT bg_owner_buildings             # not manors/financial districts
NOT bg_gold_fields                 # not gold mines
NOT bg_monuments_hidden            # not monuments
country_ownership_fraction > 0     # some government owns at least a share
```

### Capital Tab (nationally-owned, Tables 3-5)
```
is_government_funded = no
is_subsistence_building = no
NOT bg_service
NOT bg_owner_buildings
NOT bg_gold_fields
NOT bg_monuments_hidden
levels_owned_by_country = { target = ROOT value >= 1 }  # ROOT's nationals own at least 1 level
```

Note: Capital tab uses `levels_owned_by_country` (counts all of ROOT's nationals: gov + private + self) instead of `country_ownership_fraction` (which only measures government ownership by any country).

## Development Conventions

- **Back up** before major version changes. Keep backups outside the Paradox mod folder.
- **Localization files** must be UTF-8 with BOM (`EF BB BF`). Single space indent before keys.
- **Game reference files** are in the Victoria 3 installation under `game/` (e.g., `game/common/script_values/`, `game/common/defines/`).
- **Save compatibility**: when adding new variables, use a version-specific initialization flag (e.g., `gov_v13_initialized`) so existing saves get new variables seeded on first tick.
- **Testing timestamps**: during testing, use v1.3.X [HH:MM] naming convention.
