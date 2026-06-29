# Troubleshooting & Breaking Changes

## The `MessageFlags.ComponentsV2` flag

This is the single most common source of confusion when adopting Components V2 in Discord.Net.

**When you don't need to think about it:**
- Sending a brand-new message with a `ComponentBuilderV2` (`RespondAsync`, `SendMessageAsync`, `FollowupAsync`) — Discord.Net sets the flag automatically.
- Editing a message that *already* has V2 components — the flag is already set and stays set.

**When you must set it manually:**
- You're editing (`ModifyAsync`/`UpdateAsync`) a message that did **not** originally have Components V2, and you now want to swap its components for a V2 tree.

```csharp
MessageFlags flags = (component.Message.Flags ?? MessageFlags.None) | MessageFlags.ComponentsV2;

await component.UpdateAsync(m =>
{
    m.Flags = flags;
    m.Components = cv2Builder.Build();
});
```

**Important:** once a message carries this flag, **it cannot be removed**. There's no "downgrading" a message back to classic `content`/`embeds`.

**A known edge case:** in rare situations the flag may not get auto-set even when it should — one observed trigger is a message using more than 5 Action Rows. If your component update silently fails or behaves oddly, set the flag manually as a safe default even when you believe it should already be set.

## v3.18 breaking changes

Discord.Net introduced Components V2 around v3.18, and it came with two structural changes worth knowing about even if you're not actively using V2 yet:

### 1. `IMessage.Components` changed shape

The collection type backing `IMessage.Components` changed. If existing code assumed it was always `IEnumerable<ActionRowComponent>`, recover that behavior with a filter:

```csharp
IReadOnlyCollection<IMessageComponent> components = message.Components;
IEnumerable<ActionRowComponent> legacyRows = components.OfType<ActionRowComponent>();
```

This matters because a V2 message's top-level components are *not* all `ActionRowComponent` anymore — `OfType<ActionRowComponent>()` will silently (and correctly) skip Containers, Sections, Text Displays, etc.

### 2. Nested component builder types changed

The way components nest under `ActionRowBuilder`/`ComponentBuilder` changed type signatures. If you have older code building component trees, expect to need updates — current syntax is shown throughout `builder-guide.md`.

## Common errors and what they usually mean

| Symptom | Likely cause | Fix |
|---|---|---|
| Local exception thrown when calling `.Build()` | A component is nested somewhere it's not allowed (e.g. a Button outside an Action Row/Section accessory, two selects in one Action Row, a Container nested inside another Container). | Check the nesting table in `SKILL.md` / `component-types.md`. Discord.Net validates structure before any network call, so the exception message usually names the offending component. |
| "Invalid Form Body" from the Discord API itself (not a local exception) | A field limit was exceeded (label > 80 chars, `custom_id` > 100 chars, > 25 select options, > 10 media items, > 40 total components, etc.) | Cross-check the limits table at the bottom of `component-types.md`. |
| Component updates silently don't apply / message looks unchanged | Forgot to set `MessageFlags.ComponentsV2` when converting a non-V2 message via `ModifyAsync`/`UpdateAsync`. | See the flag section above. |
| Bot never receives an interaction for a button | The button is a Link (`url`) or Premium (`sku_id`) style — these never fire `InteractionCreated`. | Confirm the button's intended style; only Primary/Secondary/Success/Danger send interactions. |
| `custom_id` collision error | Two interactive components in the same message share a `custom_id`. | Generate IDs that embed the entity they refer to (e.g. `"close-{id}"`) rather than reusing a static string across rows. |
| Modal `required` / `disabled` interaction is rejected | `disabled` is being set on a component inside a modal — modals can't have disabled components, full stop. | Remove `disabled` from any modal-bound component; it's message-only. |

## Debugging strategy

1. **Trust the local exception first.** Discord.Net validates your component tree's shape before it ever talks to Discord — read the exception message; it's almost always more specific than what the API would return.
2. **If it passes local validation but Discord still rejects it,** set a breakpoint immediately before the send call (`RespondAsync`/`ModifyAsync`/`UpdateAsync`/...) and step through within the 3-second interaction window — the actual HTTP response from Discord will usually carry a more specific field-level error than a generic timeout would suggest.
3. **Double check the flag** any time you're editing (not creating) a message and components silently don't take effect.
