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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.ReplyComposer;
using Osysharp.Storage;
using ComposerKitTests;

[Test(Building)]
[runas(Ada)]
void a_draft_with_its_files_and_staged_steps_is_there_when_the_composer_opens_again() {
  var request = RequestId(101);
  var shot = KeepComposerFile("", Uploaded("radiator.png",
    [137, 80, 78, 71, 13, 10, 26, 10, 0, 0, 0, 13], "image/png"));
  Assert.True(shot.Picture);
  Assert.NotNull(SaveDraft("", request, "The fitter says **Thursday**.", false, [shot],
                           [new StagedStep { Action = "ComposerKitTests.SetRequestStatus", Parameter = "status", Choice = "Waiting", Label = "Set status: Waiting on the tenant" }]));
  // A reload: everything comes back from the server, nothing from the page.
  var opened = OpenComposer("", request);
  Assert.Equal("The fitter says **Thursday**.", opened.Text);
  Assert.Equal("radiator.png", opened.Files.Single().Name);
  Assert.Equal("Set status: Waiting on the tenant", opened.Steps.Single().Label);
  Assert.NotNull(opened.SavedAt);
  // Another record has its own.
  Assert.Equal("", OpenComposer("", RequestId(102)).Text);
  // Emptied, the draft is gone — not kept as an empty row.
  Assert.Null(SaveDraft("", request, "  ", false, [], []));
  Assert.Equal(0, ComposerDraft.Count());
}
```

From [`tests/drafts.test.osy`](tests/drafts.test.osy).

```osy
using Osysharp.ReplyComposer;

[Test]
void bold_wraps_the_selection_and_a_second_press_takes_it_off() {
  var once = Wrap("hello world", 6, 11, "**", "bold");
  Assert.Equal("hello **world**", once.Text);
  Assert.Equal(8, once.From);
  Assert.Equal(13, once.To);
  var twice = Wrap(once.Text, once.From, once.To, "**", "bold");
  Assert.Equal("hello world", twice.Text);
  Assert.Equal(6, twice.From);
  // A selection that includes the marks unwraps too.
  Assert.Equal("hello world", Wrap("hello **world**", 6, 15, "**", "bold").Text);
  // Nothing selected: the marks around a word that is selected, ready to type over.
  var empty = Wrap("Hi ", 3, 3, "_", "italic");
  Assert.Equal("Hi _italic_", empty.Text);
  Assert.Equal("italic", empty.Text.Substring(empty.From, empty.To - empty.From));
}
```

From [`tests/format.test.osy`](tests/format.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.ReplyComposer;
using ComposerKitTests;

[Test(Building)]
[runas(Ada)]
void applying_a_macro_stages_its_steps_and_the_send_runs_them_in_order() {
  var id = SaveMacro(null, "", "Heating fixed", "Heating", "Hi {{tenant.first_name}}, the radiator is fixed.", false, true,
                     [Step("label", "heating"), Step("status", "Done")]);
  var request = RequestId(101);
  var applied = ApplyMacro(id, request);
  Assert.Equal("Hi Hanna, the radiator is fixed.", applied.Text);
  Assert.Equal(["Add a label: heating", "Set status: Done"], applied.Steps.Select(s => s.Label).ToList());
  // Applying changes nothing on the record — the writer may still edit the reply or take a step off.
  Assert.Equal(RequestStatus.Open, Request.Single(r => r.Id == request).Status);
  var sent = SendFromComposer("", request, applied.Text, false, [], "", -1, true, applied.Steps);
  Assert.Equal("Sent. Labelled heating. Status set to Done.", sent.Said);
  var r = Request.Single(x => x.Id == request);
  Assert.Equal(RequestStatus.Done, r.Status);
  Assert.Equal("heating", r.Labels);
}
```

From [`tests/macros.test.osy`](tests/macros.test.osy).

## Learn more

- [The Osysharp.ReplyComposer reference](https://osysharp.com/reference/ui/composer-kit/)
- [Osysharp.ReplyComposer in the package catalogue](https://osysharp.com/packages/#osysharp-replycomposer)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
