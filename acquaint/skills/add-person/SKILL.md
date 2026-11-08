---
name: add-person
description: Add one or a few people to Acquaint from what the user provides, such as a name with context, a pasted bio, an email signature, or a business card. Checks for duplicates, runs the Add page's web lookup, and attaches the best profile photo. Also adds a photo to someone already saved. Use when the user asks to add, save, or remember someone in Acquaint, or to find a photo for someone in Acquaint.
---

# Add a person to Acquaint

Use the Acquaint MCP tools (`get_account_status`, `match_people`,
`look_up_person`, `add_people`, `set_person_photo`, `link_emails`,
`list_groups`).

1. **Account.** Call `get_account_status` once per session. If `connector` is
   not `available`, say the connector needs the Acquaint paid plan, give
   `upgradeUrl`, and stop.
2. **Extract.** Take only what the user gave you: full name, preferred name,
   company, position title, position type (one of the listed values, or leave
   it empty), pronunciation, tags, links, and email. A link to the person's own
   profile or website (their LinkedIn, X, GitHub, or site) is a profile link;
   a post or an article that mentions them is a source link. Never guess a
   title, company, or link, and never add notes the user did not write or ask
   for.
3. **Check duplicates.** Call `match_people` with the name and email.
   - `existing`: the person is already saved. Say so and give `appUrl`. If
     `hasPhoto` is false, go to "Photo for someone already saved" below;
     otherwise stop.
   - `candidates`: if a candidate's `profiles` include a profile link the user
     gave (same platform and handle, or same website host), it is the same
     person: say so and stop, or link the email. Otherwise show them (name,
     company, title, profiles) and ask whether this is one of them or a
     different person. If it is the same person and you have an email, call
     `link_emails`.
   - `none`: continue.
4. **Look up.** Call `look_up_person` with `name`, `organization` (the company
   the user gave, or the organization from a corporate email domain), `role`
   if the user gave one, and `profileUrl` if the user gave a profile link (the
   first one, such as their LinkedIn).
   - Use the result when `ready` is true, `person.fullName` is plausibly the
     same person, and `person.company` agrees with what the user gave. With no
     company to compare, `confidenceLabel` must be `high`.
   - If `status` is `ambiguous` and `options` are listed, show the options
     (name, company, title, location, and each option's profiles) and ask
     which one, or whether none fits.
     Call `look_up_person` again with the chosen option as `selectedIdentity`,
     unchanged.
   - Otherwise save a plain card from what the user gave.
   - The user's own details always win over the lookup's.
5. **Group (optional).** If the user named a group, call `list_groups` and use
   its `groupId`. Only manual groups can take members. If the group does not
   exist, ask before creating one with `save_group`.
6. **Confirm.** Show the card in a few lines, including where the details and
   the photo come from (source host), and ask before saving, unless the user
   already said to save it.
7. **Save.** Call `add_people` with a `requestId` such as
   `add-<yyyymmdd>-<name slug>`. Reuse it if you retry, so no duplicate is
   created. With a used lookup, pass its `company`, `positionTitle`,
   `positionType`, `sourceUrls` (from `sources`), and `fromLookup: true`. Pass
   the lookup's `profileLinks` URLs as `profileUrls` only when
   `confidenceLabel` is `high`. Profile links the user gave go in `profileUrls`
   too, with `linksFromUser: true`. Never write a profile URL yourself.
8. **Photo.** If the used lookup returned `photoCandidates`, call
   `set_person_photo` with the new `personId`, `personName`, and `photo` set to
   the first candidate, unchanged. Skip one that is clearly a logo, a group
   photo, a stock image, or someone else; try the next. On `failed`, try the
   next candidate once.
9. **Report.** Give the result, `appUrl`, and whether a photo was saved. If
   not, say why and that the person's page in the app can take one.

## Photo for someone already saved

When the user asks for a photo for a saved person, or step 3 found one without
a photo: call `look_up_person` with their saved name and company, apply the
rules in step 4, and call `set_person_photo` with their `personId` and the
first usable candidate. Never change their saved fields. A person who already
has a photo keeps it (`kept`) unless the user asks to replace it; only then
pass `replace: true`.

Errors: `upgrade_required` means the account is not on the paid plan or has no
people slots left. `quota_exceeded` means this month's AI lookups or photos are
used up; save a plain card. `card_permanently_deleted` means the user deleted
this person before, so do not retry. `group_not_found` means the group is
missing, smart, or archived. `rate_limited` means too many people were added in
the last hour; tell the user to try again later.
