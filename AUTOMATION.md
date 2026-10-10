# Automation Guide

This file is the contract for the daily lesson routine (a scheduled cloud Claude Code routine). It defines state, lesson format, validation, and what the routine may and may not touch. Edit it to change automation behavior without editing the routine's prompt.

---

## State: `progress.json`

| Field | Meaning |
|-------|---------|
| `last_read_day` | Highest day the owner has marked as read (by replying "done" to the lesson email, or by editing this file) |
| `last_published_day` | Highest day whose lesson file exists in the repo |
| `last_notified_day` | Day number mentioned in the most recent email |
| `last_notified_date` | Date (YYYY-MM-DD, America/New_York) of the most recent email |
| `reminder_count` | Consecutive nudges sent for the same unread day; reset to 0 when a new lesson is published or the day is marked read |

`progress.json` is the only source of truth. `PROGRESS.md` is personal, gitignored, and must not be read or written.

---

## Daily decision

1. If Gmail has a reply (from dhananjay.nikam711@gmail.com) of "done" or "read" on the latest lesson email, set `last_read_day = last_published_day` and `reminder_count = 0`.
2. If `last_read_day < last_published_day`: the owner has an unread lesson. Do **not** generate. Increment `reminder_count`, email a short nudge ("Day N is waiting — M days"), commit `progress.json`.
3. Otherwise (`last_read_day == last_published_day`): generate Day `last_published_day + 1`, validate, commit, push, email a link.
4. Never generate more than one lesson per run. Never stop at Day 84.

---

## File naming and location

- Lesson path: `week-XX/day-YY.md` (two digits each). Week = `ceil(day / 7)`.
- Find the day's topic and expected path in `COURSE_PLAN.md`.
- Create the week folder if it does not exist.
- Days 14, 21, 28, ... are assessment/review days. Follow the topic given in `COURSE_PLAN.md`; use the same sections as other lessons, where Core Concepts is a structured review and the Hands-On Exercise is the assessment.

---

## Lesson format

Match `week-01/day-07.md` (the most recent full lesson) for tone, depth, and layout. Read it and the previous lesson before writing. Required `##` headings, in this order:

1. Spaced Review (previous days; pick material from 1, 3, and 7 days back)
2. Learning Objectives
3. Why This Matters
4. Prerequisites
5. Core Concepts (numbered `###` subsections; include ASCII architecture diagrams where useful)
6. Mental Model
7. Tradeoffs
8. Common Failure Modes
9. Hands-On Exercise
10. Knowledge Check
11. Self Assessment
12. Key Takeaways
13. Summary
14. What You Should Be Able to Explain
15. Next

Title line: `# Day N — <topic from COURSE_PLAN.md>`.

Content rules (from `README.md`): engineering understanding over mathematics; for each concept cover what/why/how/when to use/when not to/tradeoffs/production implications; enterprise examples; no interview-prep framing.

---

## Validation (run before committing)

A lesson passes only if all hold:

- All required headings above are present, in order.
- 3,000–4,500 words (recent lessons are ~3,500–4,300).
- No placeholder text (`TODO`, `TBD`, `[...]`, "Lorem", "stub").
- Hands-On Exercise has at least 2 exercises; Knowledge Check has at least 5 questions.
- Topic matches `COURSE_PLAN.md` for that day.

If validation fails, do not commit the lesson. Email a failure notice naming the check that failed.

---

## Commit rules

- Commit to `main`. Message: `Add Day N: <title>`.
- A lesson commit may change only: the new lesson file, `progress.json`, the `README.md` "Available Lessons" table (add a row; create a section for the week on its first lesson), and the `Current Status` rows in `README.md`.
- Nudge-only runs change only `progress.json`.
- Never edit or delete existing lessons, `COURSE_PLAN.md`, `AUTOMATION.md`, or `.gitignore`.
- Never force-push.

---

## Email format

- To: dhananjay.nikam711@gmail.com
- New lesson: subject `Day N: <title>`; body is one line on what the lesson covers, then the GitHub link `https://github.com/dhananjaynikam711/AI-Engineer-Learning/blob/main/week-XX/day-YY.md`, then "Reply 'done' when you've finished it."
- Nudge: subject `Day N is waiting`; body is how many days it has been waiting, the link, and the same reply instruction.
- Failure: subject `Lesson generation failed (Day N)`; body names the failed check.
- Keep all emails short and plain text.
