# Osysharp.ReplyComposer

Where a person writes a reply — on a ticket, a case, an order, a lead. The kit never knows which: a composer is
pointed at a record by its id, and the app says once, in one class, what sending means for that kind of record.

```osy
use Osysharp.ReplyComposer@0;

policy ComposesReplies => IsAgent;          // who writes here
policy ManagesSavedReplies => IsAdmin;      // who keeps the TEAM's replies and macros

app.Composer = new ComposerSetup { Targets = [new DeskReplies()] };

public class DeskReplies : ComposerTarget {
  public override string Send(ComposerSend s) { ReplyToTicket(s.Item, s.Text, s.Note, …); return ""; }
  public override List<ReplyPlaceholder> Placeholders() { return [new CustomerFirstName { Key = "customer.first_name", … }]; }
  public override string Record { get { return typeof(Ticket).FullName; } }   // a macro's steps: the app's [Action]s on a ticket
  public override string Signature() { … }
}

Composer(ticket.Id, audience: "To Hanna Lindqvist · email", seen: lastTimelinePlace, keymap: DeskComposerKeys());
```

## What it does

| | |
|---|---|
| **Saved replies** | the team's and each writer's own; searched from the bar; `{{key}}` placeholders the APP declares, each a class that fills itself from the record — an undeclared key is refused when the reply is saved, an unfillable one refused at send |
| **Macros** | a reply and steps that are the app's own `[Action]` functions on the record, with the choices each offers; applying stages them, the send runs them in order as the writer; a step whose choice has gone is said, never skipped |
| **Formatting** | markdown; a bar and ⌘B/⌘I acting on the selection (`TextArea(selectionStart:, selectionEnd:)`); a second press undoes; Preview |
| **Drafts** | per record and per author, with files and staged steps; saved every 2 s while changing and on leaving the box; survive a reload; "Draft saved" read live; Discard with Undo |
| **Prepared drafts** | `PrepareDraft(kind, item, text, by)` — offered live in the composer, never sent by itself (the seam an assistant writes through) |
| **Signature** | `ComposerTarget.Signature()`, under every reply (never a note), with a switch to leave it off |
| **Files** | pasted, dropped or picked; checked by `ComposerTarget.Keep` while the writer can still act |
| **Send and set…** | `ComposerTarget.After(item)` — the app's choices; `ComposerSend.After` carries the one picked |
| **Mentions** (0.4.0) | typing `@` in a note offers `ComposerTarget.Mentionable(item)`; a name written out by hand counts too; `ComposerSend.Mentions` carries their keys; somebody who may not be named carries a `Refusal`, shown greyed and refused at send with the same sentence |
| **Settings** | `ComposerSettingsSections` announces "Saved replies" and "Macros"; the kit serves both pages |

## Its tests

The kit's test app is a building's maintenance desk, with its own reply rows, statuses and labels, so the kit is shown
working outside a helpdesk.
