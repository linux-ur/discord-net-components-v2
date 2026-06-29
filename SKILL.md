---
name: discord-net-components-v2
description: Reference and implementation guide for building Discord bot UIs with Discord.Net's Message Components V2 system (ComponentBuilderV2) in C#. Covers every V2-only component (Container, Section, Text Display, Thumbnail, Media Gallery, File, Separator), the shared V1/V2 interactive components (Action Row, Button, String/User/Role/Mentionable/Channel Select, Text Input), component nesting rules, the Discord.Net fluent builder API, custom IDs vs numeric component IDs, the IS_COMPONENTS_V2 / MessageFlags.ComponentsV2 flag, interaction handling (buttons, selects, modals, FindComponentById), and known v3.18+ breaking changes. Use this skill whenever the user is writing or debugging a Discord bot in C# with Discord.Net and mentions "Components V2", "ComponentBuilderV2", containers, sections, action rows, buttons, select menus, modals, message components, or interaction handling for Discord — even if they just paste an error about `MessageFlags.ComponentsV2`, component nesting, or "Invalid Form Body".
---

# Discord.Net — Message Components V2

## What this skill covers

Discord's **Message Components V2** is a layout system that replaces (optionally) the old `content` + `embeds` model with a tree of composable components: containers, sections, text blocks, media galleries, separators, files, and the familiar action rows of buttons/selects. Discord.Net exposes this through `ComponentBuilderV2`.

This skill is the single source of truth for:
1. Which component goes inside which (nesting rules) — get this wrong and Discord.Net throws before it even calls the API.
2. The exact field limits (char counts, min/max items) so generated payloads don't get rejected by Discord.
3. The Discord.Net-specific builder syntax (`WithX` fluent methods) and the C# patterns for sending, updating, and reacting to these components.
4. The flag (`MessageFlags.ComponentsV2` / `IS_COMPONENTS_V2`) that gates all of this, and the breaking changes introduced around Discord.Net v3.18.

Read the body below for the everyday cheat sheet. Open a file in `references/` when you need exhaustive field tables, full worked examples, or troubleshooting steps.

| File | Open it when you need... |
|---|---|
| `references/component-types.md` | The full field-by-field spec for every component type (limits, required fields, parent/child rules, message-vs-modal availability). |
| `references/builder-guide.md` | Discord.Net's `ComponentBuilderV2` fluent API, confirmed vs. inferred method names, and a complete worked C# example (a "support ticket" panel). |
| `references/interactions.md` | The interaction lifecycle, catching button/select/modal interactions, `FindComponentById`, custom-ID design patterns, and modal handling. |
| `references/troubleshooting.md` | The `MessageFlags.ComponentsV2` flag, v3.18 breaking changes, and the most common build/runtime errors with fixes. |

## Mental model

Components V2 has exactly two kinds of components:

- **Shared components** (also valid in classic Components V1): Action Row, Button, the five Select types, Text Input. These have existed since V1 and behave the same way.
- **V2-only components**: Container, Section, Text Display, Thumbnail, Media Gallery, File, Separator. These only render if the message carries the `ComponentsV2` flag.

Once a message is sent with the V2 flag, **`content` and `embeds` stop working** — `Text Display` and `Container` are the replacements. The flag, once set on a message, **cannot be removed**. Attachments also stop auto-displaying; they must be referenced explicitly via a `File`, `Thumbnail`, or `Media Gallery` component using the `attachment://<filename>` URI scheme.

## Nesting cheat sheet

Get this table right and most "invalid component" exceptions disappear. "Top-level" means it can sit directly in the message's component array; everything else must be nested inside something else.

| Component | Type # | Level | Can contain | Must be inside |
|---|---|---|---|---|
| Container | 17 | Top-level | Action Row, Text Display, Section, Media Gallery, Separator, File | — |
| Section | 9 | Top-level or in Container | 1–3 Text Displays | — |
| Action Row | 1 | Top-level or in Container | Up to 5 Buttons **or** 1 Select (any select type) **or** 1 Text Input (legacy modal pattern, deprecated) | — |
| Text Display | 10 | Top-level, in Container, or in Section | — | — |
| Media Gallery | 12 | Top-level or in Container | 1–10 media items | — |
| Separator | 14 | Top-level or in Container | — | — |
| File | 13 | Top-level or in Container | — | — |
| Button | 2 | Interactive | — | Action Row, or a Section's `accessory` |
| Thumbnail | 11 | Accessory | — | A Section's `accessory` |
| String/User/Role/Mentionable/Channel Select | 3,5,6,7,8 | Interactive | — | Action Row |
| Text Input | 4 | Interactive | — | A Modal (ideally inside a `Label`; raw Action Row placement is deprecated) |

A message can hold **up to 40 total components**.

## Quick start

```csharp
[SlashCommand("ticket", "Open a support ticket panel")]
public async Task OpenTicketPanelAsync()
{
    var builder = new ComponentBuilderV2()
        .WithTextDisplay("# Support Center")
        .WithTextDisplay("Pick a category below and our team will jump in.")
        .WithActionRow([
            new ButtonBuilder("Billing", "ticket-open-billing", ButtonStyle.Primary),
            new ButtonBuilder("Technical", "ticket-open-technical", ButtonStyle.Secondary),
            new ButtonBuilder("Other", "ticket-open-other", ButtonStyle.Secondary)
        ]);

    await RespondAsync(components: builder.Build());
}
```

Note there is **no `MessageFlags` argument here** — `RespondAsync`/`Build()` with a `ComponentBuilderV2` sets the flag automatically on first send. The flag only needs to be set *manually* when you later `ModifyAsync`/`UpdateAsync` a message that didn't originally carry V2 components — see `references/troubleshooting.md`.

## Working rules of thumb

- **IDs vs custom IDs.** Every component gets a numeric `id` (auto-generated sequentially, unique per message, you may set your own). Interactive components (buttons, selects, text inputs) additionally need a developer-defined `custom_id` (string, 1–100 chars, unique per message) — this is what comes back in the interaction payload. Don't confuse the two: `FindComponentById` matches the numeric `id`; your interaction switch/case usually matches on `custom_id`.
- **Encode state in the custom ID.** A common, simple pattern is `"{action}-{entityId}"` (e.g. `"ticket-close-482910"`), then split on the last `-` when handling the interaction. For anything beyond a single ID, prefer a short delimited string over trying to cram JSON into the 100-character budget.
- **Link buttons never reach your code.** A `Style = Link` button has a `url` instead of a `custom_id` and Discord never sends your bot an interaction for it. Same for premium (`sku_id`) buttons.
- **Respond within 3 seconds, then you have 15 minutes.** The initial ack (`RespondAsync`, `DeferAsync`, `UpdateAsync`) must happen fast; after that you can keep editing/responding to that interaction for up to 15 minutes — handy for rebuilding a component tree after a button click without needing a brand-new message.
- **Validate before you debug Discord's response.** Discord.Net checks your component tree's structure client-side before making the HTTP call. If something's wrong, you'll usually get a local exception with a more useful message than the API's "Invalid Form Body" — read it first.
- **Some newer modal-only components are not yet documented in Discord.Net's builder.** Discord's API reference (as of this writing) also defines `Label` (18), `File Upload` (19), `Radio Group` (21), `Checkbox Group` (22), and `Checkbox` (23) — all modal-only. Discord.Net's own Components V2 documentation (as of v3.20.1) only enumerates types 1–14 and 17. Before relying on the newer modal types, check the installed Discord.Net package version/changelog — they may not have fluent builder support yet, even if the raw API supports them.

## When something looks off

1. Check `references/component-types.md` for the field's exact limit (most "invalid form body" errors trace back to a length/count limit).
2. Check `references/troubleshooting.md` for the `ComponentsV2` flag and v3.18 migration notes — by far the most common runtime issue.
3. Check `references/interactions.md` if the bot isn't responding to a click at all (wrong `custom_id` match, missed `DeferAsync`, or a >3s ack).
