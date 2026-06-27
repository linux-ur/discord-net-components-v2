Discord.Net Builder Guide


How to actually construct and send Components V2 messages with Discord.Net's ComponentBuilderV2, ActionRowBuilder/array syntax, ButtonBuilder, SelectMenuBuilder, and ModalBuilder.


Two builder "modes"


Discord.Net's Components V2 documentation draws a clear line between two ways components get added:


V2-only components (Container, Section, Text Display, Thumbnail, Media Gallery, File, Separator) are added to a ComponentBuilderV2 via WithX fluent/chain methods — confirmed in the official guide for WithTextDisplay(...) and WithMediaGallery([...]), and stated to follow the same pattern for the rest of the V2-only set.
Shared V1/V2 components (Action Row and its children — buttons, selects, text inputs) are built as component arrays/builders passed into WithActionRow([...]), the same way you'd build a ComponentBuilder for classic V1 components. You still need to know the nesting rules (see component-types.md) because Discord.Net validates structure locally and throws before calling the API if you get it wrong.


var builder = new ComponentBuilderV2()
    .WithTextDisplay("# Title")          // confirmed fluent method
    .WithMediaGallery([imageUrl])         // confirmed fluent method
    .WithActionRow([buttonOrSelectArray]) // shared V1/V2 pattern
    .WithTextDisplay("more text, can repeat as many times as needed");





Confirmed vs. inferred: WithTextDisplay and WithMediaGallery are demonstrated directly in Discord.Net's own examples. WithSection, WithThumbnail, WithFile, WithSeparator, and WithContainer follow from the stated "V2-only components use WithX methods" rule, but their exact parameter shapes (e.g. how a Container's nested builder is passed in) aren't shown verbatim in the source guide. Confirm signatures with IntelliSense/source before assuming a parameter order.




Building blocks


Text Display


builder.WithTextDisplay("# Heading\nSupports the same Markdown as a normal message.");



Pass a second argument to give it an explicit numeric id if you intend to look it up later with FindComponentById:


builder.WithTextDisplay("Queue position: 3", id: QueuePositionDisplayId);



Media Gallery


builder.WithMediaGallery([
    "https://cdn.example.com/banner.png"
]);



Accepts a list of URLs (or attachment://filename references); each becomes a gallery item.


Action Row — Buttons


var closeButton = new ButtonBuilder("Close ticket", $"{TicketCloseButton}-{ticket.Id}")
    .WithStyle(ButtonStyle.Danger);

builder.WithActionRow([closeButton]);



ButtonBuilder's constructor takes (label, customId); chain .WithStyle(...), .WithEmote(...), .WithDisabled(...) as needed. Always make the customId unique within the message — a common pattern is "{action}-{entityId}".


Action Row — Select menu


var prioritySelect = new SelectMenuBuilder(
    TicketPrioritySelect,           // customId
    options: [
        new SelectMenuOptionBuilder("Low", "low", isDefault: ticket.Priority == Priority.Low),
        new SelectMenuOptionBuilder("Normal", "normal", isDefault: ticket.Priority == Priority.Normal),
        new SelectMenuOptionBuilder("Urgent", "urgent", isDefault: ticket.Priority == Priority.Urgent)
    ]
);

builder.WithActionRow([prioritySelect]);



Remember: a select menu is the only component in its Action Row — you cannot put a select next to a button in the same row.


Modals (Text Input)


private static ModalBuilder BuildPriorityNoteModal(string existingNote)
{
    var textInput = new TextInputBuilder()
        .WithCustomId(TicketNoteInput)
        .WithLabel("Internal note")
        .WithValue(existingNote)
        .WithStyle(TextInputStyle.Paragraph)
        .WithMaxLength(500);

    return new ModalBuilder()
        .WithCustomId(TicketNoteModal)
        .WithTitle("Add a note")
        .AddTextInput(textInput);
}



Trigger it from a button click with component.RespondWithModalAsync(modal.Build()).


Full worked example — a support-ticket panel


This mirrors the structure shown in Discord.Net's own recipe-app walkthrough, adapted to a ticket system so you can see how the pieces compose end-to-end.


private const string TicketCloseButton = "ticket-close";
private const string TicketPrioritySelect = "ticket-priority";
private const string TicketNoteButton = "ticket-note-open";
private const string TicketNoteModal = "ticket-note-modal";
private const string TicketNoteInput = "ticket-note-input";
private const string TicketStatusDisplay = "ticket-status-display"; // used as an explicit numeric id

private static ComponentBuilderV2 BuildTicketPanel(Ticket ticket)
{
    var noteButton = new ButtonBuilder("Add note", TicketNoteButton)
        .WithStyle(ButtonStyle.Secondary);

    var closeButton = new ButtonBuilder("Close ticket", $"{TicketCloseButton}-{ticket.Id}")
        .WithStyle(ButtonStyle.Danger);

    var prioritySelect = new SelectMenuBuilder(
        TicketPrioritySelect,
        options: [
            new SelectMenuOptionBuilder("Low", "low", isDefault: ticket.Priority == Priority.Low),
            new SelectMenuOptionBuilder("Normal", "normal", isDefault: ticket.Priority == Priority.Normal),
            new SelectMenuOptionBuilder("Urgent", "urgent", isDefault: ticket.Priority == Priority.Urgent)
        ]
    );

    return new ComponentBuilderV2()
        .WithTextDisplay($"# Ticket #{ticket.Id} — {ticket.Subject}")
        .WithTextDisplay($"-# opened by <@{ticket.AuthorId}> · status: `{ticket.Status}`", id: TicketStatusDisplay)
        .WithActionRow([prioritySelect])
        .WithTextDisplay(ticket.Description)
        .WithActionRow([noteButton, closeButton]);
}

[SlashCommand("ticket", "Open a new support ticket")]
public async Task OpenTicketAsync([Summary("subject")] string subject, [Summary("description")] string description)
{
    Ticket ticket = await ticketService.CreateAsync(Context.User.Id, subject, description);

    await RespondAsync(components: BuildTicketPanel(ticket).Build());
}



Handling the interactions this panel produces is covered end-to-end in references/interactions.md, including how UpdateAsync rebuilds this exact component tree after the select menu changes or the note modal is submitted.


Sending vs. editing


First send (RespondAsync, FollowupAsync, SendMessageAsync with components:): Discord.Net sets the ComponentsV2 flag for you automatically. No extra step needed.
Editing an existing message (ModifyAsync, UpdateAsync): if the message already had V2 components, the flag carries over automatically. If you're converting a plain message into a V2 one for the first time, you must set the flag yourself — see references/troubleshooting.md.
