---
name: tailwind-merge-config
description: Configuring tailwind-merge for a custom Tailwind v4 theme. Use when wiring extendTailwindMerge for project design tokens, when custom-prefix utility classes fail to merge or merge when they shouldn't, when adding a new token namespace to the merge config, or when shared-prefix utilities (text-/font-/border-/stroke-/divide-) collide silently. Pinned to tailwind-merge 3.6 / tailwindcss 4.3 — re-verify against source on upgrade.
---

# Configuring tailwind-merge for a custom Tailwind v4 theme

When a project ships its own design tokens as Tailwind v4 `@theme` CSS variables (e.g. `--color-brand-*`, `--radius-brand-*`), `twMerge` doesn't know about them by default. The fix is `extendTailwindMerge` — but the wiring is full of silent traps where a class _looks_ merged and isn't, or vice versa. The rules below let you wire any token namespace correctly the first time.

Throughout this skill, treat `brand-` as a stand-in for whatever prefix the project uses on its theme keys.

## Two facts everything follows from

1. **tailwind-merge: exact-trie beats validator.** A class part resolves by walking a trie of literal segments first; only if no exact path matches does it fall back to a group's validators. So enumerating a key as a string pins it to the right class group, overriding catch-alls — the color scale's `isAny`, the width groups' numeric matchers. This is why correct configs enumerate concrete keys instead of a `brand-*` validator, which would mis-route every prefix-shared utility.

2. **Tailwind v4: utilities resolve a namespace fallback chain, color before width.** `border-*`/`stroke-*` try the color namespaces (`--border-color`/`--color`, `--stroke`/`--color`) _before_ the width namespace (`--border-width`/`--stroke-width`). So a name that exists as both a width and a color renders as the color.

## Wire each namespace by family

| CSS namespace                                                    | Mechanism                                          | Why                                                                                                                                                |
| ---------------------------------------------------------------- | -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| color, background-color, text-color, border-color, fill, stroke  | **nothing** — leave the default `color: [isAny]`   | isAny already merges any `bg-/text-/border-/fill-/stroke-brand-*`; enumerating is redundant                                                        |
| text (font-size), font, font-weight, tracking, leading, radius, shadow | enumerate keys into the same-named theme scale | these scales are not isAny, so `rounded-brand-*` won't merge otherwise — and font-size vs text-color (both `text-`) need it to stay apart          |
| spacing, width, height, size, padding, margin, gap               | pool keys into the **`spacing`** scale             | `w-/h-/size-/p-/m-/gap-*` all read tailwind-merge's one `spacing` scale; over-inclusion is harmless because different class groups never cross-merge |
| border-width, stroke-width                                       | **custom theme scale + `fromTheme` wiring**        | no built-in scale; width groups match by number/length only, so a named width on the shared `border-/stroke-` prefix is swallowed by the color `isAny` unless pinned |

If a token export introduces a namespace not in the table, classify it into one of these families before wiring. Don't silently drop unknown namespaces — codegen should throw.

## override vs extend

Emit a scale under tailwind-merge's `override` (replace the default) when its CSS namespace is reset to project-only in your stylesheet (`--<ns>-*: initial`) — a closed vocabulary. Otherwise `extend` (keep Tailwind's defaults so `p-4`/`border-2`/`font-bold` still merge alongside named tokens). Keep the override list in sync with the `*: initial` list.

### A reset namespace is not a closed prefix — keep surviving static keywords

`override` replaces tailwind-merge's _entire_ scale for the group, including the keyword literals its default carried. But `--<ns>-*: initial` clears only the **theme namespace** — Tailwind still emits utilities on that prefix sourced elsewhere:

- **Static values** Tailwind hardcodes (`leading-none` → `line-height: 1`), independent of `--leading-*`.
- **Cross-namespace values** (`leading-<number>` resolves through `--spacing`, which is not reset).

So `leading-none` is still generated, yet a project-only override drops `none` from tailwind-merge's `leading` scale and orphans it (an unmatched class never merges). Re-add such keywords explicitly. Only a keyword whose tailwind-merge class group sources it **solely from the theme scale** needs this:

- `leading` — class group is `[themeLeading, ...spacing]`, no literal `none` → **re-add `none`**.
- `radius`, `shadow` — class groups hardcode `none`/`full`/`''`, so those survive any theme override → nothing to do.
- `leading-<number>` — already merges via the un-overridden `spacing` scale → nothing to do.

When adding a group to the override list, check its tailwind-merge default class group: any keyword listed as a bare string (not via a `fromTheme` getter) that Tailwind still emits must be re-added.

## Preserve generated class groups when spreading

If your codegen also emits `extend.classGroups` entries (e.g. prefix-less typography shorthands), any later `extendTailwindMerge` call that adds `fromTheme` width groups must **spread** the existing classGroups, not replace the key:

```ts
classGroups: { ...baseConfig.extend.classGroups, "border-w": [...], /* … */ }
```

Custom shorthands often carry **no standard prefix**, so nothing else recovers them — not `isAny`, not any default group. Overwrite the key instead of spreading and `brand-typography-a brand-typography-b` silently keeps both — the spread is load-bearing.

## Shared-prefix gotchas

A prefix shared by two CSS properties is where merges break _silently_: a value of one kind gets eaten by the other. Pin the non-color side so they coexist.

- `text-` = font-size + text-color → enumerate font-size into `theme.text`; text-color stays isAny.
- `font-` = family + weight → enumerate both `theme.font` and `theme['font-weight']`.
- `border-`/`stroke-` = width + color → wire width via `fromTheme`; color stays isAny.

### Color-shadowing on border/stroke width

Because Tailwind resolves `border-*`/`stroke-*` color-first (fact 2), a width key that **also exists as a color** (e.g. a token name appears in both `--border-width-*` and `--border-color-*`) renders as a color and is unreachable as a width on that prefix. Drop such shadowed keys from the width scale so tailwind-merge agrees with Tailwind. But `divide-x`/`divide-y` are width-only — no color competes on their prefix — so give them a separate **`divide-width`** scale carrying the full, unfiltered border-width set.

**The wiring, not the scale, is where divide breaks.** Point `divide-x`/`divide-y` at the `divide-width` scale via `fromTheme` — not at `border-width` (and not at tailwind-merge's numeric default). A divide-only key like `divide-x-brand-dialog` is absent from the filtered `border-width` set, so it matches no validator at the `divide-x` trie node and falls **up** to the `divide` node, where the default `divide-color`'s `isAny` claims it: the width silently becomes a color. Symptoms — two divide widths don't collapse, and a divide width is dropped when paired with a divide color. Binding to `divide-width` makes the key match at its own node first (fact 1), so it stays a width. Reproducible against tailwind-merge 3.6 by deliberately wiring divide to `border-width` instead.

## Verify behavior, not just shape

These failures are silent, so write runtime tests against `twMerge`. For each prefix-shared family assert **all three** directions:

- **merge** — two of the same kind collapse to the last: `rounded-brand-a rounded-brand-b` → `rounded-brand-b`.
- **coexist** — a width and a color keep both: `border-brand-<width> border-brand-<color>` → both. This is the direction that breaks when wiring is missing.
- **survive** — for an `override` scale, a Tailwind static keyword on that prefix still merges with a project key: `leading-brand-a leading-none` → `leading-none`. Breaks when the override drops the keyword.

After regenerating the config, run codegen twice and confirm the output is byte-identical (idempotent).

### Lint fights the fixtures

Tailwind-aware lint plugins (e.g. `eslint-plugin-better-tailwindcss`) match `twMerge(...)` by callee and parse its string args as class lists. On lint they will reorder (consistent class order), simplify (canonical classes), or reject (no-conflicting-classes, no-unknown-classes) the intentional conflicts in your test fixtures — silently rewriting what the test asserts (e.g. swapping the last class so a "last wins" expectation no longer holds). Disable the relevant rules at the top of the test file:

```ts
/* eslint-disable better-tailwindcss/enforce-consistent-class-order, better-tailwindcss/no-conflicting-classes, better-tailwindcss/no-duplicate-classes, better-tailwindcss/no-unknown-classes, better-tailwindcss/enforce-canonical-classes -- intentional fixtures; do not reorder/simplify/reject */
```
