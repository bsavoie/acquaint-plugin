---
name: prep-meetings
description: Prepare for upcoming meetings in Acquaint. Reads the user's calendar for a date range (default this week), adds attendees who are not already in Acquaint with the Add page's web lookup and profile photo, and builds a review group the user can practice before the meetings. Use when the user asks to prep their meetings, add meeting attendees to Acquaint, or make a review deck for the week.
---

# Prep meetings in Acquaint

Goal: everyone the user will meet in the range is in Acquaint with a face to
practice, and one group holds them in meeting order, ready to practice.

The user's request is the instruction to save clear cases, look them up, and
attach their photos. Do not ask "OK to add these?" for them. Ask only about the
unclear cases, and ask about all of them in one question.

## Tools

- Calendar: use whatever calendar tool is connected (for example Superhuman or
  Google Calendar). If it has several linked accounts, query each one. If no
  calendar tool is available, say so and stop.
- Acquaint (MCP server `acquaint`): `get_account_status`, `match_people`,
  `look_up_person`, `add_people`, `set_person_photo`, `link_emails`,
  `list_groups`, `save_group`.

## Steps

1. **Range and account.** Default to the rest of this week, Monday to Sunday,
   in the user's time zone. Call `get_account_status`.
   - If `connector` is not `available`, stop before reading the calendar. Say
     the connector needs the Acquaint paid plan and give `upgradeUrl`.
   - If `people.remaining` is lower than the likely number of new people, say
     so up front. People are added in order of their first meeting.
   - If `aiLookupsThisMonth.remaining` or `photosThisMonth.remaining` is lower
     than the number of new people, say so up front. People past the allowance
     become plain cards without a photo.
2. **Collect attendees.** Use accepted and tentative events only. Skip:
   - the user (every address of every linked account);
   - rooms and resources, group or list addresses (team@, all@, and similar),
     and attendees with no name and no email;
   - attendees who declined;
   - declined events, canceled events, all-day blocks, out-of-office, and
     focus time.

   Deduplicate by email. For each person, keep their display name, email, first
   meeting time, and the meetings they attend. If a meeting has more than 15
   attendees, put it in the questions at step 6 instead of including everyone.
   If an attendee has only an email, look for their name in the user's mail
   headers (a From, To, or Cc display name for that exact address) before
   asking.
3. **Organization hint.** From a corporate email domain, derive the
   organization name (`northwind.com` becomes Northwind; `nd.edu` becomes
   University of Notre Dame). Personal mail domains (gmail.com, outlook.com,
   icloud.com, yahoo.com, proton.me, and similar) give no hint.
4. **Match.** Call `match_people` in chunks of up to 50 with `name` and
   `email`. For each result:
   - `existing` (email key): already in Acquaint. Include them in the group.
   - `candidates`: decide. If one candidate's company matches the organization
     hint, or the name is identical and nothing conflicts, it is the same
     person: include them and remember the pair for `link_emails`. Otherwise
     look the attendee up at step 5: if the lookup's `profileLinks` share a
     platform and handle (or a website host) with one candidate's `profiles`,
     it is the same person and needs no question. If not, it is a question for
     step 6.
   - `none`: new person.

   Matched people with `hasPhoto: false` get a photo at step 8.
5. **Look up.** Call `look_up_person` for every new person, and for every
   matched person with `hasPhoto: false`. Run at most four at a time, in order
   of first meeting. Each call takes up to a minute.
   - Send `name` and `organization` (the hint). Add `role` only when the user
     gave one, or when the person's own email signature states it and the user
     asked you to use their mail. Never send meeting titles, agendas, or other
     attendees.
   - **Use the result** only when all of these hold:
     - `ready` is true;
     - `person.fullName` is plausibly the attendee (same surname; the given
       name, a nickname, or an initial matches);
     - `person.company` is the organization hint or clearly the same
       institution. With no hint, `confidenceLabel` must be `high`.
   - Otherwise treat the lookup as unresolved. Do not ask about it unless the
     user wants that; the person becomes a plain card, as before.
   - If a lookup returns `quota_exceeded`, stop looking up. The rest become
     plain cards.
6. **Ask once.** Gather every unclear case into one question: possible
   duplicates, large meetings, and anything else you could not decide. Offer
   clear choices, such as "same person", "different person", or "skip". Keep
   working on the clear cases while you wait, or finish them after the answer.
7. **Add.** Call `add_people` in chunks of up to 25, in order of first meeting.
   - `requestId`: `prep-<ISO year>W<week>-<name slug>`, for example
     `prep-2026W46-priya-raman`. Add `-2`, `-3`, and so on for two people with
     the same name. Reuse the same IDs if you retry.
   - `fullName` from the calendar display name, cleaned up (no email address,
     no "(External)" suffixes). If there is only an email and no name was
     found, ask instead of guessing a name.
   - `emails`: the attendee's email. Acquaint stores only a private match key.
   - With a used lookup: `company`, `positionTitle`, and `positionType` from
     `person`, `sourceUrls` from `sources` (at most 5), and `fromLookup: true`.
     A title the user gave, or that the person's own email signature states,
     wins over the lookup's.
   - Profiles: pass the lookup's `profileLinks` URLs as `profileUrls` only
     when `confidenceLabel` is `high`. A medium-confidence lookup saves no
     profiles. Never write a profile URL yourself.
   - Without one: `company` is the organization hint, if any. Leave
     `positionTitle` and `positionType` empty unless the user or the
     invitation states them.
   - Never put lookup text in `notes`, and leave `tags` empty unless the user
     asks.

   If a result is `upgrade_required` or `access_closed`, stop adding and still
   build the group from the people who are saved. A result of `rate_limited`
   means the hourly limit on new people is reached: stop adding, build the
   group from the people who are saved, and tell the user that running the
   prep again in an hour adds the rest. A result of
   `card_permanently_deleted` means the user deleted this person before: report
   it and do not try again.
8. **Photos.** For each saved person whose lookup was used and returned
   `photoCandidates`, call `set_person_photo` with the `personId`, the
   `personName`, and `photo` set to the first candidate, unchanged. The photo
   check lists its best match first.
   - Skip a candidate whose source page is clearly about someone else or is a
     logo, a group photo, or a stock image; try the next one instead.
   - `kept` means the person already had a photo. Never pass `replace` unless
     the user asks to replace photos.
   - `failed`: try the next candidate once, then move on.
   - Stop at `quota_exceeded`.
9. **Link.** Call `link_emails` for the existing people you matched by name,
   so next week they match by email.
10. **Group.** Call `list_groups`. If a group with the planned name exists,
    extend it with `groupId`. Otherwise create it with `save_group`:
    - `requestId`: `prep-<ISO year>W<week>-group`;
    - `name`: `Meetings · <Mon d>–<d>`, for example `Meetings · Nov 9–13`;
    - `serviceDate`: the first meeting day (YYYY-MM-DD);
    - `personIds`: existing and new people, in order of first meeting;
    - `description`: one line per meeting, such as
      `Tue Nov 10 · Q4 planning (Northwind): Priya Raman, Tom Li`, at most
      1,000 characters. Leave it out if the user prefers.
11. **Report.** Keep it short:
    - group name, group link, and practice link (`practiceUrl`);
    - added (count, plus names if few), and which came from the web lookup;
    - photos saved, and who has none yet and why (no confident match, no
      usable photo, allowance used up);
    - already in Acquaint;
    - skipped, and why;
    - not added, and why (limit reached, deleted before, or error).
    People without a photo can get one from their page in the app. Practice
    shows company and title instead.

## Privacy

- Send Acquaint only names, emails, organization hints, a role the user gave,
  and the short meeting labels in the group description. Never send meeting
  descriptions, agendas, notes, or other attendees' details.
- `look_up_person` sends the name, organization, and role to Acquaint's AI
  provider. It never receives emails.
- Never put meeting content in a person's `notes` unless the user asks.
- If the user says to leave someone out, leave them out of the lookups, the
  adds, and the group.
