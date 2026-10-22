[All parts](../README.md#the-parts)

# Shared · The bits that cost the most time

Four things in this recipe are not obvious from any card's documentation. If you only read one page in this repo, read this one.

| Section | Used by |
|---|---|
| [auto-entities `options` takes a whole card](#auto-entities-options-takes-a-whole-card) | Part 2 |
| [card-mod loses to bar-card on source order](#card-mod-loses-to-bar-card-on-source-order) | Part 3 |
| [The corners were square because of an unset variable](#the-corners-were-square-because-of-an-unset-variable) | Part 3 |
| [Severity order, and why descending](#severity-order-and-why-descending) | Part 3 |
| [Contrast: why the green is not `--success-color`](#contrast-why-the-green-is-not---success-color) | Part 3 |
| [Trimming the name without renaming anything](#trimming-the-name-without-renaming-anything) | Part 3 |

---

## auto-entities `options` takes a whole card

Most examples use `options` for small overrides:

```yaml
options:
  icon: mdi:battery
```

It accepts an **entire card configuration**. auto-entities merges `entity: <id>` into whatever you put there, per match, and hands the result to the wrapped card as a row:

```yaml
options:
  type: custom:bar-card     # every matched entity becomes one of these
  entity_row: true
  ...
```

This is the whole trick. Any custom card that can render as an entity row can be stamped out per entity this way.

Paired with an `area:` filter, the card maintains itself: add a battery device to that room in Home Assistant and a new row appears with no config change.

## card-mod loses to bar-card on source order

bar-card renders its own `<style>` block **after** its markup, inside its shadow root. card-mod injects **before** that. Both end up in the same shadow root, so for rules of equal specificity **bar-card wins on source order**.

The practical effect: a card-mod rule that looks correct silently does nothing.

```yaml
# does nothing - bar-card's own rule wins
bar-card-iconbar { color: var(--error-color); }

# works
bar-card-iconbar { color: var(--error-color) !important; }
```

This bit us on the icon colour specifically. bar-card hard-codes:

```css
bar-card-iconbar { color: var(--ha-icon-color); }
```

and `severity.icon` only changes the icon's **shape** (`mdi:battery-alert` vs `mdi:battery`), never its colour. So a "critical" row rendered with a red bar and a default grey icon until `!important` was added.

## The corners were square because of an unset variable

bar-card sets its radius like this:

```css
border-radius: var(--bar-card-border-radius, var(--ha-card-border-radius));
```

Many themes — including the default — never define `--ha-card-border-radius`. Both variables are then empty, the whole declaration is invalid, and it computes to **`0px`**. The bar renders with hard square corners inside a rounded card and looks broken.

Two ways to fix it. Set bar-card's own variable, which needs no `!important`:

```yaml
ha-card { --bar-card-border-radius: 6px; }
```

…or set the radius directly, which is what this recipe does, because the left and right ends want different values:

```css
bar-card-backgroundbar,
bar-card-currentbar,
bar-card-contentbar,
bar-card-targetbar,
bar-card-animationbar {
  border-radius: 0 4px 4px 0 !important;
}
```

**Square on the left is deliberate.** The fill and the track are separate stacked elements and share the radius. On a nearly-flat battery the fill is a sliver about one pixel wide, sitting exactly where a left-hand curve would eat it — round both ends and the most urgent row on the page loses its red marker entirely. Apply the radius to **all** of those elements, not just the track, or the fill pokes out of the corners once a battery is near 100%.

## Severity order, and why descending

bar-card's severity bounds are **inclusive at both ends**, and the matching loop has no early exit, so when two rules overlap the **last matching rule wins**.

Written descending, these give what you'd expect:

```yaml
severity:
  - { from: 25, to: 100, color: "#66bb6a" }
  - { from: 10, to: 25,  color: "var(--warning-color)" }
  - { from: 0,  to: 10,  color: "var(--error-color)" }
```

| Value | Rules matched | Wins |
|---|---|---|
| 10 | amber (10–25), red (0–10) | **red** |
| 25 | green (25–100), amber (10–25) | **amber** |
| 26 | green | green |

Write the same list ascending and a battery at exactly 10% comes out amber instead of red.

## Contrast: why the green is not `--success-color`

The percentage sits **inside** the bar, right-aligned. That means it only overlaps the fill at high percentages — and in this recipe anything above 25% is green. **Green is the only fill colour the text ever sits on.**

Measured against Home Assistant's default `--primary-text-color` (`#141414`), using the WCAG 2.1 relative-luminance formula:

| Background | Ratio vs `#141414` | Verdict |
|---|---|---|
| `--success-color` `#43a047` | **5.58:1** | AA pass, but hard to read |
| `#66bb6a` (used here) | **7.79:1** | comfortable |
| `--warning-color` `#ffa600` | 9.39:1 | fine |
| `--error-color` `#db4437` | **4.29:1** | fails AA |
| the pale track, any colour | 11–12:1 | fine |

Two things worth taking away:

1. **White text is not the fix.** On `#43a047`, white scores 3.30:1 — *worse* than the black it would replace. Dark text on a lighter fill was the only direction that worked.
2. **`--error-color` fails against both black (4.29:1) and white (4.29:1).** It gets away with it here only because red never appears above 10%, where the fill is a stub at the far left and the text is over the pale track. If you move the value outside the bar, or change the thresholds, re-check this.

The icon keeps the saturated `--success-color`, so the severity colour still reads strongly at a glance; only the bar fill is lightened.

## Trimming the name without renaming anything

Battery sensors are usually called "Window **Battery**", "Smoke Detector **Battery**". In a card already titled with the room, where every row is a battery, that word is noise on every line.

The obvious fix — renaming the entities — is the wrong one. Those names are doing useful work everywhere else: other dashboards, voice assistants, notifications. The word is only redundant *here*.

bar-card picks the name like this (`src/bar-card.ts`):

```js
const name = config.name ? config.name : state.attributes.friendly_name;
```

`name` is a plain string. bar-card has **no template support at all** — no `[[[ ]]]`, nothing — and auto-entities applies one `options` block to every matched entity, so a single static `name` would label every row identically. Neither can give you a per-row computed name.

So hide the real text and write a trimmed copy back with a pseudo-element:

```css
{% set fn = state_attr(config.entity,'friendly_name') %}
{% set nm = fn | replace(' Battery Level','') | replace(' Battery level','') | replace(' Battery','') | trim %}
bar-card-name { font-size: 0 !important; }
bar-card-name::after {
  content: "{{ nm if nm else fn }}";
  font-size: 15px !important;
}
```

Display-only. Nothing outside this view changes, and the rows stay self-maintaining.

**Two traps worth knowing**, both found by reading the rendered names back out of the DOM rather than looking at the page:

1. **Integrations disagree on capitalisation.** Zigbee sensors end in "Battery"; Home Assistant companion-app sensors end in "Battery level", lowercase *l*. A chain handling only `' Battery Level'` and `' Battery'` leaves "Darren Pixel 7 Pro **level**" behind. Order the replaces longest-first.
2. **Some entities are called nothing else.** A robot vacuum's battery sensor can have the friendly name "Battery Level" in full, so stripping leaves an empty string and the row renders blank. The `nm if nm else fn` fallback keeps the original in that case.

The trade-off: the name stops being real selectable text, and a tooltip still shows the untrimmed value.

---

[All parts](../README.md#the-parts)
