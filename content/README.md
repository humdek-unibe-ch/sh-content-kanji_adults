# content/

The study definition — every question, instruction screen and trial — as the
readable sources behind the migration. Nothing here is read at runtime:
`server/db/v1.0.0.sql` embeds all of it.

Once installed, the database is the live system and CMS edits take effect
immediately. A change made only in the CMS is lost on a fresh install unless it
is ported back here. Regenerating the migration is the dev's job; ask for a build
rather than hand-editing `v1.0.0.sql`.

## Files

| File | Contents |
|---|---|
| `kanji_part1.surveyjs.json` | part 1 — consent and the ID code |
| `kanji_demographics.surveyjs.json` | demographics |
| `kanji_pause{1..4}.surveyjs.json` | the vignette pauses between the task pages |
| `kanji_part2.surveyjs.json` | part 2 — device, closing code |
| `kanji_prizedraw.surveyjs.json` | prize draw |
| `instructions.json` | instruction, orientation and closing screens, all four languages |
| `items_learn.csv`, `items_recall.csv` | trial item tables |
| `kanji_labjs.css` | task stylesheet |
| `kanji_labjs.seg{1..4}.builder.json` | the four lab.js segments, one per task page |

The `.builder.json` files are what Module LabJS holds and what the lab.js Builder
opens for a preview. They are build output: a change to the item CSVs or
`instructions.json` means a rebuild, not an edit here.

**The segments are the full study.** `items_learn.csv` is 30 A + 30 B + 2
practice, `items_recall.csv` 15 A + 15 B + 1 practice — 93 trials. For
counterbalancing every learning and recall loop holds both lists and keeps one at
runtime; loop titles still say A or B, but the save names the block after the list
shown. The CSVs follow the research team's `Lists_Kanji_Adults.xlsx`, except for
file names where the list spells an image differently (`Dunkel`,
`Gefaehlich_Kanji`, `Tickets`).

## Worth knowing

- **The correct image's side is fixed per item.** `correct_pos_original` in
  `items_recall.csv` (1 left, 2 right) comes from the researchers' list, and each
  recall row's images must follow it. Only the trial order is shuffled.
- **Image paths use `{{ASSET_BASE}}`**, which the migration replaces with the
  served path. Write the placeholder, not a URL.
- **Task images are embedded** in the segments as `embedded/<hash>` entries, so
  they need no upload. Each is the original from `../assets/` scaled to at most
  800 px, flattened onto white and saved as JPEG quality 72; the hash is the
  SHA-256 of those bytes. Embed a new image the same way or it will not match.
- **List A repeats an image and skips one.** `Gefaehrlich.jpg` is the distractor
  for `Vorsicht` and a target of its own, and `Fels` is learned but never tested,
  as in the research team's list.
- **`kanji_labjs.css` is linked, not inlined.** A markdown section on each task
  page renders a `<link>` to the served copy. The labJS `css` field only takes
  class names, and a `<style>` inside the study is wiped by the next screen.
- **Questionnaire pages are styled with Bootstrap utilities** on each section's
  `css` field, plus one inline `<style>` per page for what a utility cannot
  reach: the core `.cms-edit` element, the `body` background (`styles.min.css`
  forces `#fff`), SurveyJS internals, and on the pauses an overlay asking a
  phone held in landscape to turn back to portrait.

How the pages run and what they record is in the README one level up.
