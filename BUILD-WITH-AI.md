# Build it with AI

## Part 1: For you (the human)

### What happens

1. **Questions first.** Your AI asks what you've got — floors, areas, which devices charge themselves. Nothing changes on your system.
2. **You approve a plan.** It shows you what it intends to add and where.
3. **It builds and checks.** If you're missing something, that part gets left out rather than faked.

### What you need

| Project type | Tested route | What has to be set up first |
|---|---|---|
| Home Assistant dashboard | [Claude Code](https://claude.com/claude-code) + [HA-MCP](https://github.com/homeassistant-ai/ha-mcp) | HACS installed; HA-MCP connected so the AI can read and change Home Assistant. HACS downloads may need a click from you |
| Any chat AI without a connection | – | It can still produce the finished YAML and step-by-step instructions for you to paste |

Other AI assistants with a similar connection to Home Assistant (MCP) should work too. Claude Code is the route this was built and tested with.

**You also need your house set up in Home Assistant**: devices assigned to areas, and areas assigned to floors. The whole layout is built on the area registry — without it there is nothing to group by.

### The prompt

```text
I want to recreate the battery levels dashboard from
https://github.com/sygad/ha-battery-levels in my own Home Assistant.

Read BUILD-WITH-AI.md in that repo first and follow its "Part 2: For the AI"
section exactly. Start with the questions in Step 1, one at a time.
Don't install, create or change anything until I've approved your plan.
```

---

## Part 2: For the AI

### The files

| File | What it is |
|---|---|
| `battery-levels.yaml` | **The single source of truth.** A complete view: one floor section with three room cards. |
| `parts/01-floor-section/section.yaml` | The floor section on its own |
| `parts/02-room-card/room-card.yaml` | One room card, with the battery row inlined as `options:` |
| `parts/03-battery-row/battery-row.yaml` | The battery row, annotated. Not pasteable on its own |
| `shared/README.md` | Why the card-mod rules are the way they are. Read before changing any of them |

### Rules

- **Never change or delete the user's existing config.** Add a new view alongside it.
- **Ask before installing anything.**
- **Never guess an entity ID, area or floor.** Read them from the registry and confirm they exist.
- **Copy the config as it is.** Change only what's in the substitution table below. Do not "tidy" the CSS or the Jinja — in particular, do not remove `!important`, do not make the border radius symmetric, and do not strip each row's card border. Every one of those is deliberate and explained in `shared/README.md`.
- **If the user has no floors defined**, say so and offer either to group by area only (drop Part 1) or to help them set floors up. Don't invent a floor structure.
- **Never type passwords or tokens for the user.**

### Step 1: Ask the user, one question at a time

1. Which dashboard should this go on, and should it be a new view or an existing one?
2. Do you have **floors** set up (Settings → Areas, labels & zones), and which floors do you want included?
3. Which devices in your house **charge themselves** rather than taking replaceable cells — phones, tablets, laptops, robot vacuums, wearables? (These get excluded.)
4. Do you want batteries that report something other than a 0–100% value (volts, or `high`/`low`) included? Default: no, they can't be drawn on a bar.
5. Do you want the percentage to one decimal place (shows `0.5 %` accurately, but also `100.0 %`), or whole numbers?

### Step 2: Map their house

Read the area registry and build the real structure before writing anything:

| Role | How to find it | Required |
|---|---|---|
| Floors | Floor registry, ordered however the user thinks of the house | Required (else drop Part 1) |
| Areas per floor | `floor_areas(<floor_id>)` | Required |
| Battery sensors per area | `sensor` domain, `device_class: battery`, `unit_of_measurement: "%"`, in that area | Required |
| Rechargeable patterns | From the user's answer to question 3 — substrings of their entity IDs | Optional |
| Batteries with no area | Entities whose `area_id` is unset | Optional — offer an "Elsewhere" section |

Skip any area that has no matching batteries — don't create an empty card for it.

### Step 3: Show the plan and wait

List the floors, the rooms under each, and how many batteries each room will show. Give the total, and name anything being excluded and why. Wait for a clear yes.

### Step 4: Install

Four HACS cards. Ask first; the user may need to click to approve each download.

| Card | Repository | Why |
|---|---|---|
| auto-entities | `thomasloven/lovelace-auto-entities` | Finds and sorts the batteries |
| Bar Card | **`vogon1/bar-card`** | Draws the row |
| card-mod | `thomasloven/lovelace-card-mod` | Icon colour, radius, padding |
| Mushroom | `piitaya/lovelace-mushroom` | Floor titles only — skip it and use a native `heading` card instead |

> **Install `vogon1/bar-card`, not `custom-cards/bar-card`.** The original is unmaintained since 2023 and has been removed from HACS. If a search returns only `vogon1`, that is correct.

After installing, tell the user to hard-refresh or use **Settings → Reset frontend cache**. Until they do, the cards render as *"Custom element doesn't exist"* — that is a cache problem, not a config error.

### Step 5: Build

Start from `battery-levels.yaml`. For each floor, copy the section; for each room on that floor, copy the room card.

| What | Where | Change to |
|---|---|---|
| `title: Top Floor` | floor section, Mushroom title card | The floor's name |
| `title: Attic Dorma` | room card → `card.title` | The room's display name |
| `area: Attic Dorma` | room card → `filter.include[0].area` | The room name **exactly as in the area registry** |
| `exclude:` patterns | room card → `filter.exclude` | Substrings from the user's own rechargeable devices, or remove the block |
| `decimal: 1` | every battery row | `0` if the user chose whole numbers in question 5 |

Every room card needs an identical copy of the `options:` block. If you change the styling, change it in all of them.

### Step 6: Check it works

- Every room card shows the number of batteries you predicted in Step 3.
- Rows are ordered lowest percentage first within each room.
- A battery at or under 10% shows a **red** icon and a red bar; 11–25% amber; above 25% green.
- The icon is coloured, not grey. If it's grey, the `!important` on `bar-card-iconbar` has been lost.
- The bar has square left corners and rounded right ones. If all four are square, card-mod isn't loading.
- No card shows *"Custom element doesn't exist"*.
- Compare against `images/top-floor.jpg` if you can see the rendered page.

### Reference: role table

| Role | Units | Used by | Required | If missing |
|---|---|---|---|---|
| Floor | – | Part 1 | Optional | Drop Part 1, group by area only |
| Area | – | Parts 1, 2 | **Required** | Stop and ask the user to assign areas |
| Battery sensor | `%` | Part 3 | **Required** | Skip that room |
| Rechargeable device patterns | – | Part 2 | Optional | Omit the `exclude:` block |
