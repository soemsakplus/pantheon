# questions.md — Minimum Profile Questions

> Asked in Phase 2 of `INSTALL.md`. **One at a time, in order.** Do not batch.
> Phrasing below is a guideline — translate naturally into the user's detected language.

## Q1 — Preferred language (asked in BOTH languages, since not yet detected)

**EN:** "Which language would you like to use with this system? (e.g. English, Thai, mixed)"
**TH:** "อยากให้ระบบนี้คุยกับคุณเป็นภาษาอะไรครับ/คะ? (เช่น ไทย, English, ผสม)"

→ Save as `language`. Use this language for all following questions.

## Q2 — Name + preferred address

"What's your name, and what would you like the agents to call you? (e.g. full name 'Somsak', call me 'พี่บรีส' / 'Boss' / first name)"

→ Save as `name` (full or how they introduce themselves) and `address` (the call-me form).
→ If user gives only one, use it for both.

## Q3 — Pronoun

"What pronoun should agents use when referring to you? (he/him, she/her, they/them, หรือไม่ระบุก็ได้)"

→ Save as `pronoun`. If user prefers not to specify, save `unspecified`.

## Q4 — Role / what you do

"In one short sentence — what do you do? (your profession, what you'll mostly use this system for)"

→ Save as `role`.

---

**That's all 4.** After Q4, proceed to Phase 3 (confirm summary) in `INSTALL.md`.

Do NOT ask: timezone, working hours, tools, projects, glossary, people, confidentiality. Those are deliberately deferred to a later phase.
