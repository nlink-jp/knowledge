# Graphics and fonts

Lessons on generating images, loading fonts and drawing text.
Each entry is Symptom → Why → How to apply. matplotlib's font order inside
containers is in [containers-and-infra](containers-and-infra.md).

---

## Go's `font.Drawer` draws missing characters as tofu and drops drawing failures silently — decide glyph presence yourself

**Symptom:** while designing an engine that renders mermaid diagrams to images (2026-09), an
independent verification pass measured `golang.org/x/image` v0.46.0:

- `font.Drawer`'s `DrawString` / `MeasureString` ignore the `ok` that `GlyphAdvance` and friends
  return (glyph index not 0). A character missing from the face is drawn as `.notdef` (tofu); a
  character whose `LoadGlyph` fails draws nothing; neither is an error.
- Faces exist that load fine (`ParseCollection` → `Font` → `NewFace`) and then fail `LoadGlyph` on
  specific characters: 9 faces among the macOS system fonts (8 ITFDevanagari faces and 1
  KohinoorTelugu face, `unrecognized CFF 2-byte operator (12 37)`). Two ITFDevanagari faces cannot
  draw the digit `1`.
- Apple Color Emoji has non-zero glyph indices but no outlines (sbix bitmaps, which `x/image`
  cannot draw).

**Why:** `font.Drawer` is a convenience that assumes the face covers the whole string; it has no
path to report a gap or a failure to the caller. A successful load does not promise that every
glyph can be drawn.

**How to apply:**
- Write the per-character face selection loop yourself. A glyph index of 0, or an error from
  `LoadGlyph`, means "missing from this face": try the next face. If no face has it, return an
  error instead of an image with the character missing (a label with a missing character is a
  wrong picture, not an ugly one).
- Skip characters that carry no glyph (variation selectors U+FE00–FE0F, ZWJ U+200D, ZWSP U+200B)
  without drawing them. Counting them as missing makes strings such as `⚠️` fail for no reason.
- On a line mixing faces, take the height from the largest ascent plus the largest descent of the
  faces used. They differ a lot (at 28px: Hiragino Sans 24.6/3.4, Helvetica 21.6/6.4, Avenir Next
  28/10.25; Hiragino's line gap is 1.5em).
- Checking at load time is not enough. Route drawing-time failures into the same "missing
  character" path.

## A `.ttc` can have readable and unreadable faces — decide readability per face, not per file

**Symptom:** in the same design, the 535 font files on a macOS machine were read with `x/image`
v0.46.0. `ParseCollection` succeeded for **every** file with a font extension (the contents are
evaluated lazily). Failures appeared only at `Collection.Font(i)`: 12 files, 32 faces could not be
read. HelveticaNeue reads 5 of its 14 faces, Songti 5 of 8. Causes: unsupported cmap formats,
invalid kern, CFF and head tables.

**Why:** parsing a collection reads only its header; each face's tables are checked when the face
is taken out. Faces in the same file are built differently.

**How to apply:**
- Decide readability, and report errors, per face. Never say "this file works".
- If users choose a font, take the chosen face out at startup, check it, and return the reason
  (which table) when it cannot be read. Drawing-time failures go through the entry above.

## `sfnt.Name` cannot choose a language — to select faces by name, walk the name table yourself

**Symptom:** the same design specified selecting a face inside a `.ttc` "by the name Font Book
shows". The verification pass measured that `x/image` v0.46.0's `sfnt.Name` returns the **first**
Mac Roman or Windows UCS-2 record whatever its language: `Hiragino Sans W6` for Hiragino Sans W6.
Font Book on a Japanese system (`NSFont.displayName`) shows 「ヒラギノ角ゴシック W6」. The name
table has the Japanese names, but this API cannot return them. The family name (ID 1) also
disagrees between records: `Hiragino Sans` on the Mac side, `Hiragino Sans W6` on the Windows side.

**Why:** the name table holds the same ID several times, per language and platform. `sfnt.Name` is
a convenience returning one of them, with no promise that it matches what the user sees on screen.

**How to apply:**
- To select by name, match the full name (ID 4) and the PostScript name (ID 6) against **every
  language's** value, case-insensitively.
- Recommend the PostScript name (for example `HiraginoSans-W6`): it does not depend on language.
- When nothing matches, list the face names in the file in the error, so the user can pick again.
- Before writing "the same name the screen shows", check the screen in the user's language
  environment.

## Core Text's line breaking slows with the length of the string it is given — set long paragraphs a window at a time

**Symptom:** report-studio-mcp broke a paragraph into lines by repeating
`CTTypesetterCreateWithAttributedString` → `CTTypesetterSuggestLineBreak`. An independent review
measured 487 s for a manuscript that was one 2 MiB paragraph. Our own per-line offset work was
quadratic too (it counted from the start each time to turn a UTF-16 offset into a character
index), but with that made constant time, Core Text's own line breaking was still slow
(measured, 2026-11-04).

**Why:** The cost of one `SuggestLineBreak` grows with the length of the whole string it was given
(measured behaviour; Core Text's internals are not documented). Handing it the whole paragraph for
every line makes the total quadratic.

**How to apply:**
- Split the paragraph into windows of a few thousand UTF-16 units (4096 here) and set each window.
  A window's last line may have been cut by the window's end, so drop it and start the next window
  at that line's start. Never split a surrogate pair. The manuscript above took 0.9 s.
- This rests on a line break depending only on that line's content. Compare in a test against the
  whole paragraph set at once (make the window size a variable and shrink it). The line widths were
  also checked against the width of the line actually drawn.
- Do not count from the start per line in your own offset or decoration work either; ask once per line.

## Core Text breaks a number in a table cell inside its characters when the unit it cannot separate is wider than the column — full-width digits break whatever the width

**Symptom:** In a typesetting MCP server (2026-11), numbers in a wide table broke as `1,234,5`/`67` and
`12.34`/`%`. Ten numbers were set in 13 contexts, 14 line positions and 3 column widths (macOS 27.2).

- Half-width numbers (`1,234,567`, `12.34%`) have no break opportunity inside. Even so, when what Core Text
  cannot separate from the number — an opening bracket before it, closing brackets and full stops after
  it, letters run into it, and the character before a leading `-` — is wider than the column, it finds no
  break and breaks inside the characters.
- Full-width digits (`１，２３４`) can break between any two characters; they broke even with the column at
  1.3 times the number.

**Why:** Line-breaking rules (UAX #14 or equivalent) forbid breaks between half-width digits, separators
and `%`, and attach brackets to the number. Full-width digits behave like ideographs and may break
anywhere (measured behaviour; Core Text's internals are not documented).

**How to apply:**
- Make a column's minimum width at least its widest "number with what cannot be separated from it".
- Put a WORD JOINER (U+2060) between the number's characters. Full-width digits then stay whole too
  (0/42); the width does not change and PDFKit's text extraction does not return it (measured).
- Do not put it in a .docx: Word keeps it as a real character and it comes along when the number is
  copied.
- Do not treat a word in which a letter comes before a digit (hash, UUID, CVE ID) as a number: counted in
  the minimum width, a hash widens its column and breaks the real numbers. Letters after the digits
  (`100mg`) are a unit and stay part of the number.
- When it still cannot fit, warn and name the columns (never leave it broken silently).

## Keep-with-next must measure with the same computation that places — and a reuse key must include the style

**Symptom:** In the same server, a rule kept a heading on the same page as the start of the next block.
Counting the start separately disagreed with the placement (it counted "two lines" of a paragraph that the
widow/orphan rule never splits at three lines). Sharing the computation by reusing a paragraph's lines
under a key (block and width) then drew the first paragraph of a quote after a heading in the body colour:
it was measured in the body style and placed in the quote's, and the key had no style (found by an
independent review).

**Why:** The "start" is decided by the placement's own rules (widows and orphans, a table's header and
first row, shrinking a figure). Counted separately, the rule lives in two places and only one changes.
Reuse is correct only when every input of the computation (width, style) is in the key.

**How to apply:**
- Get a start's height from the same function that places the block; never set it a second way just to
  measure.
- Put every input that changes the result in the reuse key (block, width, style); take the entry out when
  placing.
- Measure a start inside a container (quote, list) in the style and width the container gives its children.
- When two mechanisms guard the same result (the key and the style used to measure), break each alone in a
  test.

