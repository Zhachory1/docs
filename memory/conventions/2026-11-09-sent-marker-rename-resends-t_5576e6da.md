---
kind: convention
source: t_5576e6da
profile: ralph-author
date: 2026-11-09
---

# Renaming the sent-once marker files of a live reply loop re-sends to every thread that already has one

`bootstrap/chat-reply` records "this reply already went out" as a marker file
per operation and kind (`chat-reply:27,145-151`): `.receipt-sent`,
`.reply-sent`, `.final-sent`. `scan()` and `handle()` skip an operation whose
marker exists, and that is the only sent-once state there is — by design, so no
second datastore appears.

`dd-ralph-redmine-chat` §4.3 splits `.reply-sent` into `.clarify-sent`,
`.decline-sent` and `.reply-sent`, one per `reply_kind`. The loop runs in
production and answers requesters every sixty seconds. So the first scan after
that change looks for `.clarify-sent` in a directory that holds `.reply-sent`,
finds nothing, and sends a second reply into every thread already answered.
Nothing in the ceilings stops it: they count markers too.

**The convention.** When a running loop keys "already done" on a filename,
adding or splitting those names is a data migration, not a rename. Either keep
reading the old name as satisfying the new ones, or write the new names for
existing state before the code that reads them ships. State which, in the pull
request, next to the rename.

The generalisation: any sent-once guard whose key is derived — a filename, a
hash, an idempotency key — re-fires for every record whose key changes. The
same hazard appears when two producers stage the same message under different
operation ids, which is why `plan-chat.md` settles on replacing the existing
producer rather than running a second one beside it.

Links: `Zhachory1/docs` `projects/ralph-redmine/plan-chat.md` (PR 22,
assumption 1 and PR 2's inertness paragraph); `projects/ralph-redmine/dd-chat.md`
§4.3, §4.4; `Zhachory1/ralph` `bootstrap/chat-reply`.
