# Kanji Adults

Parent–child paired-associate Kanji memory study with questionnaires, ported
from Qualtrics. One SQL migration builds it from stock SelfHelp components; it
registers no hooks and ships no PHP.

The research team's guide — page flow, editing, export, every column — is the
admin-only `documentation` page the migration installs. It exists twice, as the
`@md_*` blocks in the migration and as `docs/handbook.html`; change both.

## Requirements

- [SelfHelp](https://github.com/humdek-unibe-ch/sh-selfhelp) **v7.10.0+** — the
  landing page is a `languagePicker` section
- [sh-shp-survey_js](https://github.com/humdek-unibe-ch/sh-shp-survey_js)
  **v1.7.0+** — needs its guest-row fixes, without which participants on the
  shared guest account overwrite each other, and `_meta_language`
- [sh-shp-lab_js](https://github.com/humdek-unibe-ch/sh-shp-lab_js) **v1.3.0+**

## Install

1. Copy `content/kanji_labjs.css` and these from `assets/` into the served `/assets`:

   ```
   ID_Brief.png  Logo_Universitaet_Bern.png  aufmerksamkeit_2c_ausrufezeichen.png
   Vignette_Franz.jpg  Vignette_Geo.jpg  Vignette_Math.jpg  Vignette_Deut.jpg
   00_ablenkung_konfetti_luftb.png
   ```

   `@base_path` at the top of the migration must match `BASE_PATH` in
   `globals_untracked.php`; every asset URL is built from it. The task's own
   images are embedded in the lab.js segments; the other 164 files in `assets/`
   are the originals they were made from.

2. Run the migration. **Without the charset flag every umlaut is corrupted:**

   ```
   mysql --default-character-set=utf8mb4 -u root <database> < server/db/v1.0.0.sql
   ```

   The largest statement is about 4.3 MB, so `max_allowed_packet` must exceed it.

3. Clear the CMS cache.

Re-running is safe but resets the study content: pages are `INSERT IGNORE`, while
surveys and task segments are matched by title or name and overwritten. Copy CMS
edits back into the migration first, or they are lost.

## Pages

One component per page, each saving to its own table, in this order. Pauses 1
and 3 fill the retention interval between learning a list and recalling it.

| Keyword | CMS name | Table |
|---|---|---|
| `home` | language picker, the site root | — |
| `kanji-adults-survey` | Kanji – Teil 1: Einverständnis und Code | `Kanji_Part1` |
| `kanji-adults-demographics` | Kanji – Teil 2: Angaben | `Kanji_Demographics` |
| `kanji-adults-task-1` | Kanji Aufgabe 1: Instruktion, Übung, Lernen Liste A | `Kanji_Task1` |
| `kanji-adults-pause-1` | Kanji – Pause 1: Französische Vokabeln | `Kanji_Pause1` |
| `kanji-adults-task-2` | Kanji Aufgabe 2: Abfrage Liste A | `Kanji_Task2` |
| `kanji-adults-pause-2` | Kanji – Pause 2: Geografie Quiz | `Kanji_Pause2` |
| `kanji-adults-task-3` | Kanji Aufgabe 3: Lernen Liste B | `Kanji_Task3` |
| `kanji-adults-pause-3` | Kanji – Pause 3: Mathematik und Aufsatz | `Kanji_Pause3` |
| `kanji-adults-task-4` | Kanji Aufgabe 4: Abfrage Liste B, Abschluss | `Kanji_Task4` |
| `kanji-adults-pause-4` | Kanji – Pause 4: Aufsatz | `Kanji_Pause4` |
| `kanji-adults-questions` | Kanji – Teil 2: Gerät und Abschlusscode | `Kanji_Part2` |
| `kanji-adults-prize-draw` | Kanji – Verlosung | `Kanji_PrizeDraw` |

The four task pages are one lab.js study split in four: 93 trials, 2 practice
learning and 1 practice recall, then 30 learning and 15 recall per list.
Changing items or instruction screens means rebuilding the segments from
`content/`.

Participants arrive from a letter without logging in, so every page grants read
access to all groups, has an `acl_users` row for the guest user, and is headless.

## Counterbalancing

The CMS names say list A on tasks 1–2 and list B on tasks 3–4, which holds only
for order `AB`. With `BA`, tasks 1–2 learn and recall list B and tasks 3–4 list
A. The instructions say "first round" and "second round", so they read right
either way.

Every segment carries both lists and keeps the one its position plays. Task 1
assigns the order once per code: the order given out less often so far, counting
everyone who started task 1, with a tie going to `AB`. Participants therefore
alternate by start time, and drop-outs still count, so the groups who finish can
end up uneven. Tasks 2–4 read the code's order back from `Kanji_Task1` through
`data_config`; opened without a task 1 row they run `AB`. A reload keeps the
order.

## How a run holds together

Part 1 collects the code and redirects to `kanji-adults-demographics/{{ID_1}}`.
Every later component has `url_params` on, so it stores the code as
`extra_param_code` and passes it on in its `redirect_at_end`. The code is a path
segment, not a query parameter, because only route parameters reach a style's
`data_config`.

Each component sets `update_based_on` to `extra_param_code`, so reopening an
unfinished page resumes its row rather than adding one: one row per participant
per table. **Part 1 is the exception** — the code is typed there, so there is
nothing to match yet. Every visit writes a row, most abandoned before submit, and
only `finished` rows carry a code.

Nothing is kept in the server session, so a run survives a lost session, a
second device or a login part-way. A task cannot resume mid-block, so the task
pages set `warning_on_reload`.

## A finished page is not repeated

Every page between part 1 and the prize draw holds two conditional containers
with the same `data_config`: find this page's row for the code with
`triggerType = 'finished'` and put it in `@page_state`, or `not-finished` if
there is none. `-open` shows the component on `not-finished`, `-done` shows
"already completed" otherwise, so a half-done page still opens and resumes.

The migration writes each condition twice: `content` is what runs, `meta` is
what the CMS Condition Builder displays.

## Recorded data

Eleven tables joined on `extra_param_code`, plus the prize draw.

| Table | Contents |
|---|---|
| `Kanji_Part1` | `EV`, `ID_1` — consent and code |
| `Kanji_Demographics` | `Demo_*` |
| `Kanji_Task1` | practice block and the first list's learning block |
| `Kanji_Pause1` | `P1_Vignette_Franz_*` |
| `Kanji_Task2` | the first list's recall block |
| `Kanji_Pause2` | `P2_Vignette_Geo_*` |
| `Kanji_Task3` | the second list's learning block |
| `Kanji_Pause3` | `P3_Vignette_Math_*` |
| `Kanji_Task4` | the second list's recall block |
| `Kanji_Pause4` | `P3_Vignette_Deut_*` — moved here from pause 3, name kept |
| `Kanji_Part2` | `Device`, `ID_2`, `Finished_Study` |

Block columns are named after the list shown (`extra_data_trials_recall_A` is
list A wherever it ran), so which task table holds a list depends on the order.
Every task row stores it as `extra_data_counterbalance`. Questionnaire columns
keep the Qualtrics names without the language suffix (`Demo_2`, not
`Demo_2_DE`), and the trial JSON keeps the Qualtrics question numbers.

Survey tables also carry `response_id`, `_json` and `_meta_*` (timings, screen,
browser, `_meta_language`). Task tables carry `labjs_response_id`, `_raw_data`,
`extra_data_n_*` counts and `extra_data_UserLanguage`. The R export in `export/`
drops `_json`, `_raw_data` and the screen, browser and account columns, which the
CMS Data page and the API still have; the handbook describes every file it writes.

Participants share the guest account, so the code is all that separates them.
Two people given one code are one participant: the second resumes the first's
unfinished page or is stopped at the first page the first one finished.

`Kanji_PrizeDraw` stores an e-mail address and no code, so an entry cannot be
tied to anyone's answers. The export writes it to its own file, never joined.

## Editing the study

The database is the live system — questionnaires under **Module SurveyJS**, the
task under **Module LabJS** — and edits take effect immediately. Renaming either
breaks the match the migration uses, and the next run seeds a second copy.

Demographic answer values are sequential `1..n` in display order, and
`visibleIf` / `defaultValueExpression` reference them. Renumbering an option
means updating those expressions in the same edit, or questions silently stop
appearing.
