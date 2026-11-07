# Acquaint plugin for Claude Code

Adds people to Acquaint and builds meeting-prep groups from Claude Code.
Requires the Acquaint paid plan. Acquaint is at
[getacquainted.app](https://getacquainted.app).

## Install

```bash
claude plugin marketplace add bsavoie/acquaint-plugin
```

```bash
claude plugin install acquaint@acquaint
```

Then, in an interactive Claude Code session, run `/mcp`, choose `acquaint`,
and sign in. A browser opens Acquaint's consent page. Approve it there. Claude
Code stores and refreshes the access token, and the token is never shown to the
model.

## Use

- `/acquaint:prep-meetings`: "Look up my meetings for this week and add people
  to Acquaint that aren't already in there. Create a review deck for me to
  prepare." This needs a calendar tool connected in Claude Code.
- `/acquaint:add-person`: "Add Priya Raman, CFO at Northwind, to my Board
  group." or "Find a photo for Tom Okafor in Acquaint."

## What it can do

The connector can check whether people are already saved, look people up on
the public web the way the Add page does, add new people, copy the best
profile photo onto a person, remember attendee emails as private match keys,
and create or extend manual groups. It does not read notes, change or delete
saved people, replace a saved photo unless asked, or touch practice history.
Its sign-in can reach Acquaint only through these tools: the database refuses
it direct access to your tables and photos, and it is limited to 100 new people
an hour. You can disconnect it in Acquaint under Settings > Account >
Connected apps.
