# Assessment Quiz — Specification

Status: draft for implementation · Last revised 2026-10-08 (rev. 6: performance changes, autosave sequencing, archive, storage gauge, bank cache)

This document specifies the **assessment** version of the quiz web app: scored assignments and in-class tests drawn from the FAA question bank. It is a separate Apps Script project from the **practice quiz** and shares no code or deployment with it. Each project has its own copy of the shared logic (question formatting, figure links, grading), so a change to one never affects the other.

---

## 1. Platform and access

- Standalone Google Apps Script project, deployed as a web app.
  - **Execute as:** the teacher (owner).
  - **Who has access:** anyone in the school's Google Workspace domain.
- Students must be signed into a school account. The script identifies each student by their account email; there is no name or ID field to fill in.
- Students have no access to the question Sheet, the results spreadsheets, or the script source. They receive only the HTML the script returns.
- Set the project's time zone (Project Settings) to the school's time zone. All dates and times in configs and results use it.

### 1.1 Script settings

Seven settings are stored in the project's **Script properties** (Apps Script editor → Project Settings → Script properties), not in the source code, so replacing the code never loses them. Values are entered plain, with no quotes.

| Property | Blank allowed? | Value |
|---|---|---|
| `SHEET_ID` | no | ID of the question bank spreadsheet (the part of its address between `/d/` and `/edit`). |
| `SHEET_NAME` | yes | Tab holding the questions. Blank = the first tab. |
| `REPO_RAW` | no | Raw address of the repo, e.g. `https://raw.githubusercontent.com/USER/REPO/main/`. |
| `ASSIGN_DIR` | no | Repo folder holding the assignment configs, e.g. `Assignments/`. |
| `FIG_LIST` | no | Figure list file in the repo, e.g. `figureReferences.txt`. |
| `FIG_BASE` | no | Base address for figure paths, e.g. `https://USER.github.io/REPO/`. |
| `RESULTS_FOLDER_ID` | yes | ID of the Drive folder for results files (the part of its address after `/folders/`). Blank = top level of My Drive. |

These stay out of the public repo deliberately: whoever controls `SHEET_ID` controls who is an admin and which answer key is used.

Four tuning values remain in the source: `CONFIG_CACHE_SEC`, `LINK_CACHE_SEC`, `GRACE_MS`, and `DATE_FMT`.

The script keeps its own per-assignment state (access key, results file ID, feedback release, exclusions, a config summary) in the same store, packed into the entries `state:0` … `state:7`, plus `live`. These are not edited by hand. Each entry holds about 9 KB, enough for roughly 150–250 assignments in all. Archiving an assignment (§7) shrinks its record to a small marker; the admin page's Setup preview shows how full the storage is and warns at 75%.

A missing or unusable `SHEET_ID`, `SHEET_NAME`, `REPO_RAW`, `ASSIGN_DIR`, or `RESULTS_FOLDER_ID` makes every assignment "not available" to students, and attempts are never graded while the question bank cannot be read. A problem with `FIG_LIST` or `FIG_BASE` does not block students, but figure links are missing until it is fixed. The admin page reports all of these (§7).

## 2. Data sources

### 2.1 Question bank (Google Sheet)

Read from the Sheet and kept in the script's shared cache for 15 minutes (`BANK_CACHE_SEC`). Students therefore see Sheet changes within 15 minutes, or immediately after **Refresh question bank** on the admin page (§7). Grading at submission uses the cached copy; admin actions that change grades (End, Finalize, Regrade, Exclude) always read the Sheet fresh.

| Column | Content |
|---|---|
| A | Question # (permanent, unique ID) |
| B | Question text |
| C, D, E | Option A, B, C text |
| F | Correct answer (A, B, or C) |
| G | Category, as a `/`-separated path (e.g. `Preflight preparation/Weather`) |
| H, I, J | Reserved for per-option feedback on incorrect answers (§10). Ignored for now. |
| K+ | Ignored |

Rows missing column A, B, F, or G are ignored.

**Editing questions that are already in use:** fix wording freely; it shows up on the next load or review. To fix a wrong answer key, change column F and then regrade (§7). Do not swap the text between columns C, D, and E for a question that has been used. Saved answers and choice orders refer to those columns by letter, so a swap would silently change what students' recorded answers mean.

**Category tree rule:** questions belong only to *leaf* categories. A category either has subcategories or has questions, never both. The admin page's config check (§7) reports any violations it finds.

**Allowed markup in question and option text** (all Moodle-compatible):

- `<br>` (any case, `<br/>`, `<br />`) starts a new, indented line.
- `<sup>…</sup>` and `<sub>…</sub>` render as superscript and subscript.
- `&nbsp;` is a non-breaking space. Runs of it are used for table column offsets.
- Everything else is shown as literal text.

### 2.2 Figure references (`figureReferences.txt` in the repo)

One `label:path` entry per line:

```
Figure 9:Figures/Figure09.jpg
the Chart Supplement Legend:Figures/ChartSupplementLegendBW.pdf
```

- `Figure n` labels match "Figure n" or "Figure 0n" anywhere in the question text.
- Other labels match their exact wording, ignoring capitalization and extra spaces. The longest matching phrase wins.
- Paths are relative to the repo's raw base URL. A path starting with `http(s)://` is used as-is.
- Matches become links that open in a new tab. The file is cached for 10 minutes.

### 2.3 Assignment configs (repo folder `Assignments/`)

One text file per assignment, e.g. `Assignments/unit3-test.txt`. See §3. Configs contain nothing secret; keys live only in the script (§6).

## 3. Assignment config

### 3.1 Invocation

```
<web app URL>?a=unit3-test
```

The script loads `Assignments/<a>.txt` from a fixed base address (`REPO_RAW` + `ASSIGN_DIR`, §1.1). The `a` value may contain only letters, digits, `-` and `_`. The script never accepts a full URL or path from the link, so students cannot point it at a config of their own.

Configs are cached for 5 minutes, so an edit can take that long to reach students.

### 3.2 Format

One setting per line, `name: value`. Blank lines and lines starting with `//` are ignored. Names are case-insensitive.

```
title:         Unit 3 – Weather Systems Quiz
R3:            Preflight preparation/Weather
R2:            Preflight preparation/Human factors
R:             Post-flight procedures, emergency and night operations
#112
#245
shuffle:       yes
attempts:      3
score:         best
feedback:      missed
feedback-when: released
time:          30
open:          2026-10-12 08:00
close:         2026-10-19 23:59
key:           required
```

### 3.3 Question selection

| Line | Meaning |
|---|---|
| `Rn: category` | Draw *n* random questions from that category. `R:` alone means 1. |
| `#nnn` | Include question *nnn* (column A). |

- Lines may appear in any order and any number of times. Each attempt is assembled from all of them.
- **Category matching:** a category name matches itself and every category beneath it. `R3: Preflight preparation` draws from all of `Preflight preparation/…`. Matching ignores capitalization and extra spaces.
- **No repeats within an attempt:** a question added with `#nnn` is excluded from random draws, and overlapping `R` lines never draw the same question twice.
- If a category has fewer unused questions than requested, the assignment is considered misconfigured (§3.5).

### 3.4 Settings

| Setting | Values | Default | Meaning |
|---|---|---|---|
| `title` | text | the `a` value | Shown to students and used to name the results file. |
| `shuffle` | `yes` / `no` | `yes` | Randomize question order. Answer choices are always shuffled. |
| `attempts` | number / `unlimited` | `unlimited` | Maximum attempts per student. |
| `score` | `first` / `best` / `last` / `average` | `best` | Which attempt(s) count in the Summary tab (§5.2). |
| `feedback` | `none` / `missed` / `full` | `missed` | What a student sees when reviewing an attempt (§4.6). |
| `feedback-when` | `completed` / `close` / `released` | `released` | When feedback becomes visible (§4.6). |
| `time` | minutes | none | Time limit per attempt (§4.4). |
| `open` | `YYYY-MM-DD HH:MM` | none (open now) | Attempts cannot start before this time. |
| `close` | `YYYY-MM-DD HH:MM` | none (never closes) | Attempts cannot start after this time, and in-progress attempts end at it. |
| `key` | `required` | none | Students must enter the access key to start an attempt (§6). |

If `open` and `close` are both omitted, the assignment is simply open.

`feedback-when: close` with no `close` date behaves like `released`.

### 3.5 Misconfiguration

If the config is missing or invalid, students see "This assignment isn't available right now. Please tell your teacher." Examples include an unknown setting, an unknown category or `#ID`, too few questions for an `R` line, `key: required` with no key set, and `feedback: full` with `feedback-when: completed` when more than one attempt is allowed (students would see the answers before retaking). The specific problem is shown only on the admin page.

## 4. Student experience

### 4.1 Landing page

Opening the assignment link shows:

- the assignment title and "Signed in as *student@school*";
- open/close times, the time limit, and the attempt allowance, where set;
- the student's previous attempts, each with its date, score (once feedback is visible), and a **Review** button when feedback is available;
- a **Start attempt** button (or **Resume attempt**, §4.3), with a key field if required. The button is replaced by an explanation when no attempt is possible: not yet open, closed, or no attempts left.

### 4.2 Taking an attempt

- The landing controls are hidden while an attempt is in progress.
- With an attempt limit, the header reads "You are on attempt #5/9". Without one, it reads "Attempt #5".
- With a time limit, a countdown is shown.
- Answers are saved to the server as they are chosen (autosave). Each save sends **all** current answers with an increasing sequence number, one save at a time: choices made while a save is under way go out together as soon as it returns, and the server ignores any save older than the one it already has. A failed save is covered by the next one, and by Submit, which also sends every answer.
- The bar at the top shows the save status: "Saving…", "All answers saved 10:42:13", or, if the server could not be reached, "Not saved yet. Trying again shortly."
- **Submit** first asks for confirmation: "This completes your attempt and will grade your answers. Continue?" On confirming, the attempt is graded. Unanswered questions count as wrong, with no separate warning. (An automatic submission at the time limit skips the confirmation.) The student then returns to the landing page, which shows feedback if `feedback-when` allows it.

### 4.3 Reloading and resuming

- An attempt is tied to the student's email, not the browser tab. Reloading, closing the tab, or switching computers and returning resumes the same attempt. It keeps the same questions, the same answer-choice order, and the autosaved answers.
- Resuming never draws new questions and never counts as a new attempt.
- If the assignment requires a key, resuming requires the **current** key (§6).
- Question and option text are re-read from the Sheet on every load. Corrections made after the attempt started therefore appear on resume and in later reviews.
- A student can have at most one unsubmitted attempt per assignment.

### 4.4 Time limits and closing

- When the countdown reaches zero, the answer choices are disabled and the attempt is submitted after a random wait of 0–20 seconds, so a class whose time ends together does not submit in the same instant. The server accepts a submission up to 60 seconds after the deadline (`GRACE_MS`); after that, the saved answers are graded.
- An attempt's time limit counts only time the attempt is **active**. Time while it is suspended (§4.5) doesn't count. Its deadline is the earlier of *now + remaining time* and the `close` time, recalculated whenever the attempt starts or is unsuspended.
- The server enforces the deadline. The page submits automatically when the countdown reaches zero, and the server accepts a submission up to 60 seconds past the deadline to allow for network delay.
- If an active attempt passes its deadline without being submitted (tab closed, computer off), its autosaved answers are graded automatically. This happens when the student next opens the assignment, when the admin page is opened, or within 15 minutes via a time-driven trigger.
- A teacher can also end in-progress attempts from the admin page (§7), for example at the end of class.
- A suspended attempt never runs out of time, but it is still ended by the `close` time.
- Results record how each attempt ended: `submitted`, `time limit`, `closed`, or `teacher`.
- An open attempt page checks in with the server every 30 seconds, as well as on every autosave. If the attempt has ended (time limit, close, or teacher), the page shows "This attempt has ended" and returns to the landing page. If it has been suspended, the page shows the paused screen (§4.5). Either way, every answer selected so far has already been saved.

### 4.5 Suspending and continuing later

For students whose extended time is split across sessions, for example half the test in class and the rest during a free period in a testing room.

- A teacher suspends individual attempts from the admin page (§7). Students cannot suspend their own attempts.
- Within 30 seconds, the student's open page replaces the questions with: "Your attempt is paused. Your answers are saved. Enter the current key to continue." The countdown stops.
- To continue, on any device, the student opens the assignment link and enters the **current** key. The attempt resumes with the same questions, choice order, and saved answers, and with the time remaining when it was suspended.
- Typical end-of-class sequence:
  1. Suspend the students who will finish later.
  2. **End all active** attempts. The suspended ones are not affected.
  3. Change the key.
  4. Give the new key to whoever supervises the later session.
- An attempt can be suspended and continued any number of times. Suspension sends no email; the receipt is sent when the attempt finally ends.

### 4.6 Feedback and review

`feedback` controls *what* a review shows:

| Mode | Review shows |
|---|---|
| `none` | Score only. |
| `missed` | Score; every question with the student's answer, missed questions marked. |
| `full` | As `missed`, plus the correct answer for each missed question. |

`feedback-when` controls *when* it becomes visible:

| Value | Visible |
|---|---|
| `completed` | As soon as the attempt is submitted. |
| `close` | After the assignment's `close` time. |
| `released` | After the teacher releases it on the admin page (§7). |

- Until feedback is visible, the landing page lists the attempt as "Submitted — feedback not yet available", with no score.
- Students review any time afterward by returning to the same assignment link and choosing **Review** on an attempt. A review shows the attempt's questions in the order they were taken, with the same choice order.
- Reviews reflect the most recent grading, including any regrade (§7). Question wording comes from the current Sheet.
- Questions excluded from grading (§7) are shown in reviews as "Not graded".

### 4.7 Email receipts

Whenever an attempt ends, however it ends, the student receives a receipt email. This includes a submission, a time limit, the assignment closing, and a teacher ending it. Students should be taught to treat the receipt as their proof of submission.

- **From:** the account the script runs as, with the assignment title as the sender name. Replies go to that account.
- **Subject:** `Receipt: <title> – attempt #n`
- **Body:**
  - the assignment title;
  - the attempt number;
  - when the attempt started and ended;
  - how it ended;
  - how many questions were answered out of the total;
  - a short receipt code (the first characters of the attempt token);
  - the assignment link for reviewing.
- **Score:** included only if feedback is already visible under `feedback-when`. Otherwise the email says the score will be available when feedback is released.
- Attempts that are finalized late are emailed at the moment they are finalized. This covers a tab closed before the time limit, which is graded on next access or by the 15-minute trigger.
- Regrades and exclusions do not send email.
- A failed email never blocks grading. The failure is recorded in the Attempts tab, and the admin page shows the count.

## 5. Results spreadsheet

One spreadsheet per assignment, named after its title, created in the `RESULTS_FOLDER_ID` folder (§1.1) when the first attempt starts. The script stores its ID and reuses it; a script lock prevents duplicates when the first submissions arrive simultaneously. Archive the folder at the end of the semester.

### 5.1 `Results` tab — one row per finished attempt

| # | Column | Example |
|---|---|---|
| 1 | Submit timestamp | 2026-10-14 10:42:13 |
| 2 | Email | student@school.org |
| 3 | Score | 7 |
| 4 | Out of | 8 |
| 5 | Percent | 87.5 |
| 6 | Attempt # | 2 |
| 7 | Started | 2026-10-14 10:21:50 |
| 8 | Active time (min) | 20.4 (excludes time suspended) |
| 9 | Ended by | submitted / time limit / closed / teacher |
| 10 | Question IDs (as served) | 112, 245, 703, … |
| 11 | Answers (original letters) | 112:B, 245:—, 703:C, … |
| 12 | Missed IDs | 245, 811 |
| 13 | Regraded at | 2026-10-16 15:02:40 (blank if never regraded) |
| 14 | Score before regrade | 6 (blank if never regraded) |
| 15 | Excluded IDs | 811 (questions in this attempt not counted; blank if none) |
| 16 | Receipt | the attempt token (its first 8 characters are the receipt code in the email) |

The script finds an attempt's row by its Receipt value when regrading. Rows can be sorted and columns added after column 16, but columns 1–16 should stay where they are.

Score and Out of count only graded questions. An attempt of 8 questions with one excluded is scored out of 7.

Answers are recorded as column-F letters, so they compare directly with the Sheet regardless of display order. `—` means unanswered.

### 5.2 `Summary` tab — one row per student

Rewritten after every finished attempt. Columns: Email, Attempts used, Score, Out of, Percent, Last submitted. The Score columns follow the config's `score` rule: `first`, `best`, `last`, or `average` (average of percents).

### 5.3 `Attempts` tab — internal state

Holds each attempt's token, email, start time, deadline, served questions and choice order, status, and (column J, `Answers`) the latest autosaved answers with their sequence number. Column J is written only by autosave and submission; teacher actions write columns A–I. The script uses this tab to resume, enforce deadlines, and build reviews. It is not intended for hand editing.

## 6. Access keys

- `key: required` in the config means students must enter the key to **start** an attempt. They must enter the **current** key again to **resume** one after a reload, on another device, or after a suspension (§4.5).
- The key itself is set on that assignment's admin page (`?a=unit3-test&admin`) and stored in the script's private Script properties (§1.1), per assignment. It never appears in the repo, the page source, or the results.
- Matching ignores capitalization and leading or trailing spaces.
- The teacher shares the key in class and can change it at any time, for example between class periods or at the end of an in-class test. After a change, nobody can start or resume with the old key. A page that is already open keeps working until its attempt is ended or suspended, so to stop students who walk out with a page still open, end or suspend their attempts from the admin page (§7).
- A wrong key gets "That key isn't correct". Nothing is served until the key is accepted.

## 7. Admin page

```
<web app URL>?a=unit3-test&admin
```

**Who can open it:** anyone with edit access to the question Sheet, plus the account the web app runs as. The script checks the signed-in account against the Sheet's editor list (cached for 5 minutes). Granting or removing someone's edit access to the Sheet therefore also grants or removes admin access, with no separate list to maintain. The deploying account is always allowed, so it can open the page to fix a missing or wrong `SHEET_ID`. Anyone else who adds `&admin` simply gets the normal student view. Because the project is standalone, Sheet editors do not see or edit the script itself.

Opened without an assignment (`<web app URL>?admin`), the page shows only the script settings check and the setup preview.

All admin actions apply to the assignment named in the link. The page provides:

- **Script settings** (shown only when there is something to report): each setting from §1.1 that is missing or not working, and each blank setting whose fallback is in use. Settings that are set and working are not listed. Checks:
  - `SHEET_ID` opens a spreadsheet, it has a tab named `SHEET_NAME`, and that tab has usable questions;
  - `REPO_RAW`, `ASSIGN_DIR`, `FIG_LIST`, and `FIG_BASE` are set;
  - the figure list can be retrieved and is not empty;
  - `RESULTS_FOLDER_ID` opens a Drive folder.
- **Config check:** the parsed config, the questions available for each `R` line, and any errors (§3.5) or category-tree violations (§2.1).
- **Key:** set, change, or clear the access key.
- **Feedback release:** release or withdraw feedback when `feedback-when: released`.
- **Results:** a link to the results spreadsheet.
- **In-progress attempts:** each unfinished attempt, with its status (active or suspended), start time, how many questions are answered so far, active time used, and time remaining. Actions on selected attempts:
  - **Suspend selected** pauses those attempts (§4.5).
  - **End selected** grades the selected attempts, active or suspended, immediately from their autosaved answers (Ended by: `teacher`) and sends receipts.
  - **End all active** does the same for every active attempt. Suspended attempts are left alone.

  Typical use at the end of an in-class test: suspend the extended-time students, end everyone else, then change the key (§4.5).
- **Finalize now:** grade any expired attempts immediately (§4.4).
- **Archive this assignment** (in the Results section): available once no attempts are unfinished. Students can no longer open the assignment ("This assignment is no longer available."). Its stored state shrinks to the title, archive date and results file ID; the access key, exclusions and feedback release are cleared. The results spreadsheet is not touched. **Unarchive** reconnects it. Consider also removing the config file from the repo.
- **Regrade:** re-score every finished attempt for this assignment against the current column F. A preview first lists the attempts whose scores would change. On confirmation, the script:
  - updates Score, Percent, and Missed IDs in the Results tab;
  - fills in Regraded at, and records Score before regrade the first time an attempt's score changes;
  - rewrites the Summary tab;
  - makes reviews show the corrected grading.

  Questions since deleted from the Sheet keep their original grading. Regrading never changes which questions an attempt contained or the answers recorded.
- **Exclude from grading:** for a question that can't be fixed by correcting the answer key. Enter its question # and see a preview of the attempts affected; on confirmation:
  - the question no longer counts for anyone in this assignment. It is removed from both Score and Out of in every finished attempt, listed in Excluded IDs, and the Summary tab is rewritten;
  - attempts still in progress show it but won't count it;
  - new attempts don't serve it. A random draw takes another question from the same `R` line instead; a `#nnn` question is simply dropped;
  - reviews show it as "Not graded".

  An exclusion can be undone with **Include again**, which regrades accordingly. Exclusions apply only to this assignment. To retire a question from every assignment, fix it or delete it in the Sheet, which also excludes it from future draws.
- **Setup preview** (bottom of the page): how many questions were read, from which tab and when, any rows skipped for missing columns, the first question rendered as students see it (with its answer key and category), the first entry in the figure list shown as an image, and how full the assignment storage is. A **Refresh question bank** button beside the heading re-reads the Sheet for everyone at once.
- **Email problems:** the number of receipts that failed to send, if any.

## 8. Performance and concurrency

Each request from a page is a short, separate run of the script on Google's servers, as the deploying account. Google allows that account **30 simultaneous runs** across all of its scripts, including the practice quiz and this admin page.

To keep requests short and few:

- The question bank, configs and the figure list are cached (§2).
- The open page checks in every 30 seconds using the cache only.
- One script-wide lock makes requests that change shared records take turns. It is held only for the write itself:

| Action | Takes the lock? |
|---|---|
| Page load, landing page, resuming an active attempt, autosave, check-in, review, viewing the admin page | No (landing and the admin page do only if an expired attempt must be graded) |
| Starting a new attempt, continuing a suspended one | Yes, briefly |
| Submitting (including at the time limit) | Yes, briefly, plus a moment to record the receipt status |
| Grading expired attempts, admin changes (suspend, end, key, release, regrade, exclude, archive, refresh) | Yes, briefly |

If Google refuses or interrupts a request (for example, too many at once), the page retries it automatically after a short random wait, up to five times over about 30 seconds, showing "Still saving…" or "Still submitting…". Every request is safe to repeat: a repeated save stores the same answers, a repeated start returns the same attempt, and a repeated submit reports the attempt as submitted. Messages meant for the student, such as a wrong key, are shown at once and not retried.

## 9. Security notes

- Answer keys never leave the server. Pages receive question text and shuffled choices tagged with original letters, never column F.
- Submissions are accepted only for a valid, unsubmitted attempt token belonging to the signed-in student. Only the question IDs recorded for that attempt are graded.
- Time limits, open/close times, attempt limits, and keys are all enforced on the server, not just in the page.
- An attempt can continue anywhere a page is still open until it is ended or suspended. Changing the key stops new starts and resumes, but only ending or suspending the attempt stops an already-open page, within about 30 seconds.
- Not covered: the script cannot stop students from using notes, other tabs, or each other during an attempt. In-class assessments still depend on supervision.

## 10. Out of scope (for now)

- Resetting an individual student's attempt. Technical problems are handled ad hoc from the Results tab.
- Feedback for incorrect answers on individual questions. This will use columns H, I, and J of the question Sheet.
- Different time limits for individual students. Extended time is handled with a generous `time` setting plus suspending and ending attempts individually (§4.5, §7).
- Partial credit, question types other than three-option multiple choice, and syncing with a gradebook or LMS.
