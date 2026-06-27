Handling Interactions


How to catch and respond to clicks/selections/submissions coming from a Components V2 message, using Discord.Net's InteractionCreated event.


Lifecycle


The initial response to any interaction must happen within 3 seconds (RespondAsync, DeferAsync, or UpdateAsync).
After that initial ack, you have a window of up to 15 minutes during which you can keep responding to/editing for that same interaction token — this is exactly what you want when a button click needs to rebuild the whole component tree (new priority, new page, new state) without sending a brand-new message.


Catching the event


Hook DiscordSocketClient.InteractionCreated and branch on the concrete interaction subtype:


private async Task OnInteractionCreatedAsync(SocketInteraction arg)
{
    switch (arg)
    {
        case SocketMessageComponent component:
            await HandleComponentAsync(component);
            break;

        case SocketModal modal:
            await HandleModalAsync(modal);
            break;
    }
}



Matching on custom_id


The simplest and most common approach: switch on component.Data.CustomId for static, known IDs, and fall through to a parsing branch for dynamic ones (IDs that embed an entity ID).


private async Task HandleComponentAsync(SocketMessageComponent component)
{
    switch (component.Data.CustomId)
    {
        case TicketPrioritySelect:
            var priority = Enum.Parse<Priority>(component.Data.Values.First(), ignoreCase: true);
            Ticket ticket = await ticketService.SetPriorityAsync(GetTicketIdFromMessage(component), priority);

            await component.UpdateAsync(m => m.Components = BuildTicketPanel(ticket).Build());
            break;

        case TicketNoteButton:
            await component.RespondWithModalAsync(BuildPriorityNoteModal(existingNote: "").Build());
            break;

        default:
            // Dynamic IDs (e.g. "ticket-close-482910") fall through to here.
            var customId = component.Data.CustomId;
            var separatorIndex = customId.LastIndexOf('-');
            if (separatorIndex == -1) return;

            var action = customId[..separatorIndex];
            var entityId = customId[(separatorIndex + 1)..];

            if (action == TicketCloseButton)
                await CloseTicketAsync(component, int.Parse(entityId));

            break;
    }
}



This is the same pattern Discord.Net's own guide uses for its "recipe" buttons ("recipes-show-me-button-{id}"), generalized: suffix the static prefix with a delimiter + the entity ID, split on the last delimiter when handling it. Using the last occurrence (not the first) means your prefix itself can safely contain dashes.


Looking up a component already on the message


You often need to read back a value that's already displayed (e.g. a Text Display showing "current priority: Low") instead of re-querying your database. IMessage.Components exposes FindComponentById<T>, which searches by the numeric id (not custom_id):


TextDisplayComponent statusDisplay = component.Message.Components
    .FindComponentById<TextDisplayComponent>(TicketStatusDisplay);

string currentStatus = statusDisplay.Content;



This only works reliably if you assigned an explicit id when building the component (see builder-guide.md) — relying on auto-generated sequential IDs makes them a moving target as you add/remove other components.


Updating the message in place


UpdateAsync is how you "refresh" a component-driven UI after a click — rebuild your ComponentBuilderV2 from the new state and swap it in:


await component.UpdateAsync(m => m.Components = BuildTicketPanel(updatedTicket).Build());



This single call both acknowledges the interaction (satisfying the 3-second window) and replaces the message's components — no separate DeferAsync needed for the common case.


Modals


Opening a modal from a button


case TicketNoteButton:
    await component.RespondWithModalAsync(BuildPriorityNoteModal(existingNote).Build());
    break;



Note: a modal can only be opened in direct response to an interaction (button/select click, slash command) — you can't pop one up out of nowhere.


Handling submission


private async Task HandleModalAsync(SocketModal modal)
{
    if (modal.Data.CustomId != TicketNoteModal) return;

    var noteText = modal.Data.Components.First(c => c.CustomId == TicketNoteInput).Value;

    Ticket ticket = await ticketService.AddNoteAsync(GetTicketIdFromMessage(modal), noteText);

    await modal.UpdateAsync(m => m.Components = BuildTicketPanel(ticket).Build());
}



modal.Data.Components gives you the flat list of submitted Text Input values; match them by CustomId the same way you would for a select menu's values.


Practical tips


Defer if you need time. If rebuilding the component tree requires a slow operation (a DB write, an external API call) that might exceed 3 seconds, call component.DeferAsync() first, then component.FollowupAsync(...) or a deferred ModifyOriginalResponseAsync(...) once the work finishes.
Don't forget ephemeral vs. public. RespondAsync(..., ephemeral: true) for "only you can see this" responses (errors, confirmations) — Components V2 panels work fine ephemerally too.
One customId per component, no exceptions. Discord rejects (and Discord.Net will throw on) a payload with two interactive components sharing a custom_id in the same message.
Re-derive the entity from context, not just the click. When a panel can be clicked by anyone who can see the channel, decide deliberately whether the clicking user or the original author should drive permission checks — components don't enforce this for you.
