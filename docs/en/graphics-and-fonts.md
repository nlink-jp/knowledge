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
