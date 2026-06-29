# Component Type Reference

Field-by-field specification for every component type in Discord's component system, as exposed (fully or partially) through Discord.Net's `ComponentBuilderV2`. This mirrors Discord's own [Component Reference](https://discord.com/developers/docs/components/reference), cross-checked against Discord.Net's [Components V2 guide](https://docs.discordnet.dev/guides/components_v2/advanced.html).

Every component shares two base fields:

| Field | Type | Notes |
|---|---|---|
| `type` | integer | The component type ID (table below). |
| `id` | integer (optional) | 32-bit identifier, unique within the message. Auto-assigned sequentially if omitted. Sending `id: 0` is allowed but treated as "unset" — Discord replaces it. Set your own `id` if you want to reliably re-find a component later with `FindComponentById`. |

Interactive components additionally use `custom_id` (string, 1–100 chars, developer-defined, unique per message) instead of/alongside `id` — this is the value that comes back in the interaction payload when a user clicks/selects/submits.

## Full type table

| Type # | Name | Category | Usable in | V2-only? | Discord.Net builder support (as of v3.20.1 docs) |
|---|---|---|---|---|---|
| 1 | Action Row | Layout | Message | No | Yes |
| 2 | Button | Interactive | Message | No | Yes |
| 3 | String Select | Interactive | Message, Modal | No | Yes |
| 4 | Text Input | Interactive | Modal | No | Yes |
| 5 | User Select | Interactive | Message, Modal | No | Yes |
| 6 | Role Select | Interactive | Message, Modal | No | Yes |
| 7 | Mentionable Select | Interactive | Message, Modal | No | Yes |
| 8 | Channel Select | Interactive | Message, Modal | No | Yes |
| 9 | Section | Layout | Message | Yes | Yes |
| 10 | Text Display | Content | Message, Modal | Yes | Yes |
| 11 | Thumbnail | Content | Message (accessory) | Yes | Yes |
| 12 | Media Gallery | Content | Message | Yes | Yes |
| 13 | File | Content | Message | Yes | Yes |
| 14 | Separator | Layout | Message | Yes | Yes |
| 17 | Container | Layout | Message | Yes | Yes |
| 18 | Label | Layout | Modal | — | **Not in Discord.Net's documented type table** — verify before use |
| 19 | File Upload | Interactive | Modal | — | **Not in Discord.Net's documented type table** — verify before use |
| 21 | Radio Group | Interactive | Modal | — | **Not in Discord.Net's documented type table** — verify before use |
| 22 | Checkbox Group | Interactive | Modal | — | **Not in Discord.Net's documented type table** — verify before use |
| 23 | Checkbox | Interactive | Modal | — | **Not in Discord.Net's documented type table** — verify before use |

Types 18/19/21/22/23 are real, current entries in Discord's own API reference (they're the modern replacement for raw Action-Row text inputs in modals), but Discord.Net's published Components V2 guide only documents types 1–14 and 17. Treat them as "API-level, library support unconfirmed" — check the changelog of whatever Discord.Net version is installed before depending on them.

---

## Action Row (1)

A row that groups interactive components horizontally. Top-level only (or inside a `Container`).

- Holds **either** up to 5 Buttons **or** exactly 1 select component (any of the five select types) **or** (deprecated for modals) 1 Text Input.
- You can't mix buttons and selects in the same row.

## Button (2)

Must live inside an Action Row or a Section's `accessory` slot.

| Field | Type | Notes |
|---|---|---|
| `style` | integer | See styles table below. |
| `label` | string (optional) | Max 80 chars. |
| `emoji` | partial emoji (optional) | `name`/`id`/`animated`. |
| `custom_id` | string | Required for Primary/Secondary/Success/Danger styles. |
| `url` | string | Required for Link style; max 512 chars. |
| `sku_id` | snowflake | Required for Premium style. |
| `disabled` | boolean (optional) | Default `false`. |

**Styles:**

| Style | Value | Use for | Required field |
|---|---|---|---|
| Primary | 1 | The one recommended action in a group | `custom_id` |
| Secondary | 2 | Supporting/alternative actions | `custom_id` |
| Success | 3 | Positive confirmation | `custom_id` |
| Danger | 4 | Irreversible/destructive action | `custom_id` |
| Link | 5 | Navigates to a URL, no interaction sent | `url` |
| Premium | 6 | Purchase flow, no interaction sent | `sku_id` |

Rules: a button has exactly one of `custom_id`, `url`, or `sku_id` — never a mix. Link and Premium buttons never trigger an `InteractionCreated` event in your bot.

Design guidance from Discord: keep labels to ~34 chars with an icon/emoji or ~38 without; use only one Primary button per group; if several buttons are equally important, make them all Secondary instead of all Primary.

## Select menus (3, 5, 6, 7, 8)

String Select, User Select, Role Select, Mentionable Select, and Channel Select all share the same shape and constraints, differing only in what populates `options`/auto-population.

| Field | Type | Notes |
|---|---|---|
| `custom_id` | string | 1–100 chars. |
| `options` | array (String Select only) | 1–25 entries; each option's `label`/`value`/`description` ≤ 100 chars. |
| `channel_types` | array (Channel Select only) | Restricts which channel types are selectable. |
| `placeholder` | string (optional) | Max 150 chars. |
| `min_values` / `max_values` | integer (optional) | Both 0–25 (min) / 1–25 (max); default 1/1. Set both to enable multi-select. |
| `default_values` | array (optional, User/Role/Mentionable/Channel only) | Pre-selected entities; count must fit within `min_values`/`max_values`. |
| `required` | boolean (optional) | Modal-only; ignored in messages. |
| `disabled` | boolean (optional) | Message-only; **causes an error if set inside a modal**. |

String Selects must sit inside an Action Row (messages) — newer Discord modal layouts prefer wrapping the select in a `Label` rather than an Action Row, but Discord.Net's documented examples still use the Action Row form.

## Text Input (4)

Free-form text entry, modal-only.

| Field | Type | Notes |
|---|---|---|
| `custom_id` | string | 1–100 chars. |
| `style` | integer | `1` = Short (single line), `2` = Paragraph (multi-line). |
| `min_length` / `max_length` | integer (optional) | 0–4000 / 1–4000. |
| `required` | boolean (optional) | Default `true`. |
| `value` | string (optional) | Pre-filled value, max 4000 chars. |
| `placeholder` | string (optional) | Max 100 chars. |

## Section (9)

Top-level (or inside a `Container`) layout component that pairs text with a small accessory — the typical use is a changelog blurb next to a thumbnail, or a description next to a button.

| Field | Notes |
|---|---|
| `components` | 1–3 `Text Display` children. |
| `accessory` | Exactly one `Button` **or** one `Thumbnail`. |

## Text Display (10)

Plain Markdown text — mentions, emoji, formatting all work as in a normal message `content`. Up to 4000 chars. Usable top-level, inside a `Container`, inside a `Section`, or directly inside a modal. Pingable mentions inside a Text Display still respect `allowed_mentions` on the message.

## Thumbnail (11)

Small image, only valid as a `Section`'s `accessory`. Supports static and animated images (GIF/WEBP); video is not supported.

| Field | Notes |
|---|---|
| `media` | An "unfurled media item" — either an external `url` or `attachment://<filename>`. |
| `description` | Optional alt text, max 1024 chars. |
| `spoiler` | Optional, default `false`; blurs the thumbnail. |

## Media Gallery (12)

A gallery of 1–10 images/media items, top-level or inside a `Container`.

| Field | Notes |
|---|---|
| `items` | Array of 1–10 items, each with `media` (unfurled media item), optional `description` (≤1024 chars), optional `spoiler`. |

## File (13)

Displays exactly one previously-uploaded attachment. Only supports the `attachment://<filename>` URI form (not arbitrary external URLs) — you must upload the file alongside the message and reference its filename.

| Field | Notes |
|---|---|
| `file` | Unfurled media item, `attachment://` only. |
| `spoiler` | Optional, default `false`. |
| `name` / `size` | Read-only, filled in by the API in responses — don't set these yourself. |

## Separator (14)

Pure spacing/divider, top-level or inside a `Container`.

| Field | Notes |
|---|---|
| `divider` | Optional boolean, default `true` — whether a visible line is drawn. |
| `spacing` | Optional integer, `1` = small padding, `2` = large padding. Default `1`. |

## Container (17)

A visually-grouped block with an optional colored accent bar down the side (similar look to an embed).

| Field | Notes |
|---|---|
| `components` | Any mix of Action Row, Text Display, Section, Media Gallery, Separator, File. |
| `accent_color` | Optional RGB integer (`0x000000`–`0xFFFFFF`). |
| `spoiler` | Optional, default `false`; blurs the whole container. |

Containers cannot nest another Container.

## Unfurled media item (shared structure)

Used by Thumbnail, Media Gallery items, and File. Two forms:

- `{ "url": "https://..." }` — any externally hosted, publicly fetchable URL.
- `{ "url": "attachment://filename.ext" }` — references a file uploaded in the same request/message; required for `File`, optional-but-common for Thumbnail/Media Gallery.

## Modal-only newcomers (18, 19, 21–23) — API reference, confirm library support

These appear in Discord's official reference but are **not** in Discord.Net's documented Components V2 type table as of the v3.20.1 docs snapshot this skill is based on. Summarized for awareness; verify against your installed package before using:

- **Label (18)** — wraps a single modal component with a `label` (≤45 chars) and optional `description` (≤100 chars). Discord now recommends Label over a raw Action Row for modal inputs.
- **File Upload (19)** — lets users attach files inside a modal; `min_values`/`max_values` 0–10, `required` flag.
- **Radio Group (21)** — exactly-one-of-N choice, 2–10 options, must be inside a Label.
- **Checkbox Group (22)** / **Checkbox (23)** — multi-select and single yes/no checkbox variants, also Label-wrapped.

## Limits worth memorizing

- Max **40** total components per message.
- Action Row: max **5** buttons, or exactly **1** select/text-input.
- Section: **1–3** Text Displays + exactly 1 accessory.
- Media Gallery: **1–10** items.
- Button label: **80** chars. Select placeholder: **150** chars. Text Display: **4000** chars. Description fields (Thumbnail/Media item): **1024** chars. `custom_id`: **1–100** chars.
- Once `ComponentsV2` is set on a message, it's permanent, and `content`/`embeds`/`poll`/`stickers` no longer work on that message.
