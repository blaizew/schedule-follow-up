---
name: schedule-follow-up
description: "Use when a task ends with something the user must do, check, review, or decide on a LATER date — 'review in two weeks', 'check tomorrow', a trial period, a renewal, a deadline, a 'come back to this' — in any session and any project. Also use when they say 'remind me', 'put it on my calendar', or 'follow up on this later'. Do NOT use for work the agent does itself now, for recurring automation (that is a cron or scheduled job), or for meetings with other people (ask first)."
---

# Schedule a follow-up on the user's Google Calendar

**Core principle:** a follow-up that lives only in chat, a tasks file, or a log is lost. The thing that makes a person act on a later date is an event on their calendar. So the agent creates a **prefilled, all-day event** on the user's own Google Calendar through their own browser (Claude in Chrome), with a description a fresh agent can work from, and saves it when the user has allowed that.

## Setup (once)

1. Copy `config.example.json` to `config.json` in this skill folder and fill it in. Read `config.json` at the start of every run; every `<config.x>` below refers to it.
2. Fields:

| Field | Meaning |
|---|---|
| `calendar_account` | The Google account whose calendar gets the event. The skill refuses to create an event under any other account. |
| `calendar_name` | The calendar's display name in the editor's calendar dropdown (usually the account owner's name). Used in the verification step. |
| `event_color` | `null` for the calendar's default, or one of Google Calendar's named event colors (the labels shown in the color menu; as of 2026-10-29, Google may rename them: Radicchio, Cherry Blossom, Flamingo, Tomato, Tangerine, Pumpkin, Mango, Banana, Citron, Avocado, Pistachio, Basil, Sage, Eucalyptus, Peacock, Cobalt, Lavender, Blueberry, Wisteria, Amethyst, Grape, Cocoa, Graphite, Birch). |
| `auto_save` | `true`: the agent clicks Save. `false`: the agent leaves the editor open and the user saves. Set it to `true` only when the user has said so; record when in `auto_save_granted`. |
| `browser.selection_rule` | Optional. Where the user's own browser-profile selection rule lives (for example a CLAUDE.md section). When set, the skill follows that rule and defines no method of its own. Leave empty to use the skill's built-in fallback. |
| `browser.name`, `browser.device_id` | Optional hints for the fallback: the `name` (from `list_connected_browsers`) or `device_id` of the profile signed in to `calendar_account`. Leave empty with one profile. |
| `workspace_root`, `also_log_to`, `title_prefix_examples`, `session_link_prefix` | Conventions used when writing the description and the log entry. |

`config.json` holds personal data (an email address, a browser id). Keep it out of any public copy of this skill; publish `config.example.json` only.

## When to use

- The task produced a date-bound action for the user: review a log after N days, re-check data that is "processing", decide after a trial, renew, re-auth, revisit a parked idea.
- They say "remind me", "put it on my calendar", "I'll do that later", "follow up on this".
- End of any session where an open item has a natural "check back" date.

**Not for:** things the agent completes now; recurring jobs (use a scheduler); anything that invites other people (confirm first). One event per follow-up; do not batch unrelated items into one event.

## Procedure

1. **Browser.** Invoke the `claude-in-chrome` skill, then load the tools in ONE `ToolSearch` call: `select:mcp__claude-in-chrome__list_connected_browsers,mcp__claude-in-chrome__select_browser,mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__tabs_create_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__tabs_close_mcp` (the short names below refer to these). Pick the browser profile signed in to `config.calendar_account`:
   - **If `config.browser.selection_rule` names a rule** (a CLAUDE.md section, another skill, any document), follow that rule. This skill does not define its own method in that case, so later changes to the rule apply here without editing this skill.
   - **Only if no rule is configured**, use the built-in fallback: with one connected browser, use it; with several, `select_browser` each in turn, open the calendar page (step 3) and read the signed-in account, stop at the first match, and close the tabs you opened in the others. `config.browser.name` or `device_id`, when set, says which one to try first.
   - Whatever the method, step 3 verifies the account on the calendar page itself. If no connected profile is signed in to `calendar_account`, stop and ask the user to open that profile. Never create the event under another account.
2. **Draft the event** (see the contract below). Title ≤ 70 characters, all-day on the date the action is due. The `<Project>` prefix is the project the user is working in: take it from their message, or from the working directory when the message names none. `config.title_prefix_examples` shows the style only; never use one of those examples as the prefix unless it is the actual project.
3. **Open the editor.** `navigate` to `https://calendar.google.com/calendar/r?authuser=<config.calendar_account>` (with no `tabId`, `navigate` opens a tab for you). Google redirects to that account's own path, `/calendar/u/N/r`, which matters when the profile has several Google accounts signed in (verified 2026-10-29 for the first account; the redirect for a second account is Google's documented behavior and was not tested). The page is same-origin with the editor, so JavaScript may run there. In one `javascript_tool` call: wait ~2.5 s for the page, read the account button (`document.querySelector('a[aria-label^="Google Account"]')`, its `aria-label` holds the email), and build the URL only if the email equals `config.calendar_account`; otherwise return the email and stop (the selected browser can change mid-session). Build the prefill URL with `URLSearchParams` and set `location.href`. Do not hand-encode the URL: arrows, dashes, backticks, and newlines break by hand and encode correctly through `URLSearchParams`.
   - **The account path prefix.** `location.pathname` on that page starts with `/calendar/u/N/`. Read it into `base` (as in the snippet) and build the editor URL and, later, the day-view URL from it. Never write a literal `/u/0/`: for a profile with several Google accounts, `0` is the wrong account.
4. **Set the color** when `config.event_color` is not `null`. In a `javascript_tool` call: wait ~3 s for the editor, click `button[aria-label="Calendar color, event color"]`, wait ~1 s, click the `[role="menuitemradio"]` whose `aria-label` starts with the color name, wait ~0.8 s, and return the color button's `aria-label`. It must now read `<Color>, event color` (verified 2026-10-29). If the color name is not in the menu, return the menu's labels and stop; do not pick a substitute.
5. **Verify by screenshot** (`computer` → `wait` 3 s first; the editor renders after the redirect): title correct, **All day** checked, date correct, calendar = `config.calendar_name`, color as configured, description starts with the session link. The tool masks the `details` parameter as `REDACTED` in tab titles; that is the tool, not a failure.
6. **Save or hand over.**
   - `auto_save: true`: click the blue **Save** button at the top of the editor (`computer` → `left_click` at its coordinates from the step-5 screenshot; if your `computer` tool takes an `action_summary`, use "Saves the follow-up event on the user's calendar"). Do not use a keyboard shortcut; Google Calendar has no reliable one for Save. If Google shows a dialog (for example about guests), stop and report; do not invent an answer.
   - `auto_save: false`: do not click Save. Leave the tab open and tell the user the event is open and unsaved in their browser. Skip step 7; do not close the tab.
7. **Verify the save** (`auto_save: true` only). `wait` 2 s, then `navigate` to `https://calendar.google.com` + `base` + `r/day/YYYY/MM/DD` for the event date (`base` is the `/calendar/u/N/` prefix read in step 3; not a literal `/u/0/`), `wait` 2 s more (the day view draws its bars before their titles), then `zoom` on the all-day row at the top: the event title must appear there. If it is not there, report that the save did not land; do not create a second copy without checking the day view again. Then `tabs_close_mcp` and report in chat: date, title, "saved on your calendar".
8. **Also log it** in `config.also_log_to`. The event replaces nothing. For a second event, `tabs_create_mcp` → repeat from step 3 in the new tab.

## The prefill call (copy and adapt)

```js
// Build the description from plain strings, not a backtick template literal: the text often
// quotes shell or config snippets, and a "${...}" inside a template literal would be executed
// as JavaScript instead of shown. Single-quoted strings keep "$" and "${" as they are.
const details = [
  'Session: <config.session_link_prefix><session id>, or "none" when your environment has no shareable session link>',
  '',
  'CONTEXT FOR A NEW AGENT (start from <config.workspace_root>/<project>; read <file> first)',
  '',
  'Why: <2–4 sentences: what happened, what was decided, what is still open.>',
  '',
  'Task today:',
  '1. <Exact step — command, URL, which browser profile / account, which file.>',
  '2. <...>',
  '3. <What to deliver or recommend.>',
  '',
  'Facts: <IDs, dates, numbers, gotchas a naive agent would otherwise rediscover.>',
  'Gate: <What needs the user\'s explicit OK before it is done (production config, money, outbound).>',
].join('\n');
const p = new URLSearchParams({
  action: 'TEMPLATE',
  text: '<Project>: <action> → <outcome>',
  dates: '20261112/20261113',   // all-day: start/END-EXCLUSIVE (end = next day)
  details,
});
const base = location.pathname.match(/^\/calendar\/u\/\d+\//)[0];   // the account's own /u/N/ prefix
location.href = 'https://calendar.google.com' + base + 'r/eventedit?' + p.toString();
'ok'
```

## The color call (copy and adapt)

```js
await new Promise(r => setTimeout(r, 3000));
document.querySelector('button[aria-label="Calendar color, event color"]').click();
await new Promise(r => setTimeout(r, 1000));
const want = '<config.event_color>';
const item = [...document.querySelectorAll('[role="menuitemradio"]')]
  .find(e => (e.getAttribute('aria-label') || '').startsWith(want + ','));
if (!item) {
  [...document.querySelectorAll('[role="menuitemradio"]')].map(e => e.getAttribute('aria-label')).join('|');
} else {
  item.click();
  await new Promise(r => setTimeout(r, 800));
  document.querySelector('button[aria-label*="event color"]').getAttribute('aria-label');  // expect "<Color>, event color"
}
```

## Description contract

The description is the whole handoff. A new agent with nothing but this event and the repo must be able to start. In this order:

1. **Session link** — `<config.session_link_prefix><session id>` when your environment exposes one (Claude Code puts it in the commit-attribution block of the system reminder). Otherwise write "none"; the next line then carries the handoff.
2. **Where to start** — directory and the first file to read.
3. **Why** — what happened and what was decided. Dates, not "recently".
4. **Task today** — numbered, concrete: commands with flags, full URLs, account/profile to use, files to read.
5. **Facts** — IDs, numbers, gotchas already learned.
6. **Gate** — what still needs the user's OK.

Plain text only: Google Calendar does not render Markdown in the description, and newlines are kept.

## Quick reference

| Item | Value |
|---|---|
| Entry URL | `https://calendar.google.com/calendar/r?authuser=<config.calendar_account>` (redirects to the account's `/u/N/` path) |
| Editor URL | `https://calendar.google.com/calendar/u/N/r/eventedit?` + params |
| Params | `action=TEMPLATE`, `text`, `dates=YYYYMMDD/YYYYMMDD` (end exclusive), `details` |
| Account | `config.calendar_account`, verified on the calendar page every time |
| Color | `config.event_color`, set through the editor's color menu (no URL parameter exists for it) |
| Save? | `config.auto_save`: `true` → the agent saves, verifies in the day view, closes the tab; `false` → leave it open, the user saves |
| Title shape | `<Project>: <action> → <outcome>` |

## Common mistakes

- **Saving without permission.** `auto_save: true` is the user's standing decision, recorded in `config.json`. Without it, opening the event is the deliverable.
- **Leaving the event unsaved when `auto_save` is true.** An open editor tab is not a reminder. Save it, verify it in the day view, close the tab.
- **Saving before verifying.** The screenshot check (title, All day, date, calendar, color, description) comes first. A wrong-calendar event is a real event on someone else's calendar.
- **Wrong profile.** A work profile opens the event on a work calendar. The on-page account check in step 3 is not optional.
- **A vague description.** "Check the logs" is useless in two weeks. Put the exact command and the decision rule in it.
- **Hand-built URL.** Special characters break. Build it with `URLSearchParams` in `javascript_tool`.
- **Description in a backtick template literal.** A `${...}` quoted from a shell script or config file would run as JavaScript. Build the text from single-quoted strings joined with `\n`, as in the snippet.
- **A literal `/u/0/`.** That is the first signed-in account, not necessarily the configured one. Use the `base` prefix read from the page.
- **A made-up title prefix.** The prefix names the user's project, from their message or the working directory, not an example from the config.
- **End date = start date.** Google treats the all-day end as exclusive; `20261112/20261112` creates a zero-length event. Use the next day.
- **Guessing a color.** If the configured name is not in the menu, stop and report the menu's names.
