# schedule-follow-up

A [Claude Code](https://claude.com/claude-code) skill that puts follow-ups on your Google Calendar.

A task ends with "review this in two weeks" or "decide after the trial", and then nothing happens, because the reminder lived in a chat window. This skill fixes that. The agent creates an all-day event on your own Google Calendar through your own browser, using the [Claude in Chrome](https://claude.ai/chrome) extension. The description is written for a future agent: session link, where to start, why, exact steps, facts, and what still needs your OK. Open the event when it comes up, paste the description into a new session, and the work picks up where it stopped.

There are no API keys and no OAuth app to set up. It drives the calendar you are already signed in to.

## What it does

1. Picks the browser profile that is signed in to your calendar account, and verifies the account on the calendar page before doing anything.
2. Opens Google Calendar's event editor with the title, date, and description prefilled.
3. Optionally sets the event color.
4. Checks the editor by screenshot: title, all-day, date, calendar, color, description.
5. Saves the event if you have allowed that in the config, or leaves it open for you to save.
6. Confirms the event in the day view and closes the tab.

## Install

```bash
cd your-project/.claude/skills   # or ~/.claude/skills for all projects
git clone https://github.com/blaizew/schedule-follow-up.git
cd schedule-follow-up
cp config.example.json config.json
```

Edit `config.json`:

| Field | What to put |
|---|---|
| `calendar_account` | The Google account whose calendar gets the events. The skill refuses any other account. |
| `calendar_name` | The calendar's name as shown in the editor's calendar dropdown. |
| `event_color` | `null` for the default, or a Google Calendar color name such as `"Tangerine"`. |
| `auto_save` | `true` lets the agent click Save. Start with `false` if you want to save each event yourself. |
| `browser.selection_rule` | Optional. If you already have a rule for choosing among browser profiles (for example in your CLAUDE.md), name it here and the skill follows it. |
| `browser.name`, `browser.device_id` | Optional hints for the built-in profile picker. Leave empty if you have one browser profile. |
| `workspace_root`, `also_log_to`, `title_prefix_examples`, `session_link_prefix` | Conventions used in the event description and the log entry. |

`config.json` is in `.gitignore`. It holds your email address and browser ids, so keep it out of any repo.

## Requirements

- Claude Code with the Claude in Chrome extension connected to a browser that is signed in to your Google account.
- The extension's site permission for `calendar.google.com`.

## Notes

- The skill never invites guests, never edits or deletes existing events, and never creates an event under a different account than the one in the config.
- Google Calendar's prefill URL has no color parameter, so the color is set through the editor's color menu. The color names are the labels in that menu.
- Verified against Google Calendar's web UI in October 2026. If Google changes the editor, the selectors in SKILL.md are the first thing to check.

## License

MIT
