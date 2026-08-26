---
name: testing-question-paper-maker
description: How to serve, drive and verify the PPIS Question Paper Maker (single static index.html) — HTTP serving so the dictionary loads, Playwright+PyMuPDF harnesses for proving formatting reaches the printed PDF, Devanagari entry, and the known font-fallback and blank-trailing-page caveats.
---

# Testing the PPIS Question Paper Maker

Repo: `alishakanwar-crypto/ppis-question-papers`. The whole tool is **one self-contained
`index.html`** (no backend, no build, no credentials). `paper-maker.html` is only a redirect
(`<meta refresh>` + `location.replace('./')`) to the single maintained copy at `./`.

## Serve it over HTTP — never `file://`

```bash
cd /home/ubuntu/repos/ppis-question-papers && python3 -m http.server 8903
```

`index.html` does `fetch('words-en.txt')` for its 63k-word British-English dictionary. On
`file://` that fetch fails and the page silently degrades to "only common misspellings were
checked" — a real fallback, but it disables every dictionary test. Always confirm the dictionary
actually loaded before asserting on spelling:

```js
await pg.wait_for_function("typeof DICT_STATE !== 'undefined' && DICT_STATE !== 'loading'")
```

`DICT_STATE` is `loading` → `ready` / `failed`. Sanity numbers: `words-en.txt` is ~590,681 bytes /
63,627 words; `index.html`, `words-en.txt` and `ppis-logo.png` must all return 200.

## Element IDs worth knowing

| ID | What |
|---|---|
| `#lang` | `en` / `hi` language select |
| `#body` | the questions textarea |
| `#max`, `#grade`, `#subject`, `#title`, `#session`, `#duration`, `#instr` | paper details |
| `#font`, `#fsize`, `#fline`, `#find`, `#boldq` | box-3 Formatting controls |
| `#sample`, `#clear`, `#check`, `#fix` | buttons |
| `#savepdf`, `#print` | "Save the PDF on this computer" (red) / "Print copies" |
| `#pdfname` | `<code>` hint showing the filename the save will use |
| `#report` | the checker findings |
| `#sheet` | the preview root; gets classes `hi` (Hindi) and `plain` (bold unticked) |

Formatting is applied as CSS custom properties on `#sheet`
(`--pfont/--psize/--pline/--pind`); an **empty select value calls `removeProperty`**, which is how
"Default" falls back to `.sheet{--psize:11.5pt}` / `.sheet.hi{--psize:12.5pt}`.
Persistence: `FIELDS` includes `font,fsize,fline,find` and `CHECKS` includes `boldq`, so all of
box 3 is saved in localStorage with the paper and must survive F5.

## "Save the PDF on this computer" — filename from the paper's own fields

`#savepdf` sets `document.title = pdfFileName()`, calls `window.print()`, and restores the old
title after 1 s. Browsers name a Save-as-PDF after the document title, so the file lands as e.g.
`Class-II-ENGLISH-APPRAISAL-WORKSHEET-2.pdf` instead of `index.html.pdf`. `#print` is a bare
`window.print()` and must **not** touch the title.

`pdfFileName()` joins `'Class-'+grade`, `subject`, `title` (each only when `.trim()` is truthy),
strips `\ / : * ? " < > |`, then collapses `\s+` to `-`, falling back to `Question-paper`.
Stripping happens **before** the whitespace→hyphen pass, so `A / B` → `A-B`, not `A--B`.

**Testing it end to end.** The DOM preview cannot prove this — you must drive real Chrome:
click `#savepdf`, and in Chrome's print window press Save, which opens a GTK **Save File**
dialog whose Name field is pre-filled from `document.title`. Screenshot that field and then
`ls` the target folder — that is the only proof the OS really receives the name. Devanagari
survives into the actual filename on disk. Also confirm the tab title reverts afterwards.
For a non-invasive scripted check, listen for `beforeprint` and record `document.title` there
rather than stubbing `window.print` — it exercises the real print path.

**Known live-hint staleness (check whether still present).** `refreshPdfName()` is wired only to
`input` on `grade`/`subject`/`title` plus one call at load. `#sample` and `languageChanged()`
assign `.value` programmatically, which does **not** fire `input`, and neither calls
`refreshPdfName()`. So after "Load a sample paper" or a language switch the hint can show the
*previous* paper's name while the file saves under the new one. Diagnostic that pins the root
cause: type one character into Title — the hint jumps to the correct value, proving
`pdfFileName()` is fine and only the refresh trigger is missing. A likely fix is calling
`refreshPdfName()` at the end of the sample loader and of `languageChanged()`.

**Filename edge cases worth asserting** (compare `#pdfname` against `pdfFileName()+'.pdf'` every
time, so a stale hint is caught): each field blank individually; all three blank →
`Question-paper`; a title of every forbidden character; leading/trailing/multiple spaces and
tab/newline; Devanagari; a 300-char title. Two cosmetic quirks that are **not** crashes: a field
consisting only of forbidden characters passes the `.trim()` truthiness test and contributes an
empty part, leaving a dangling `Class-` or trailing hyphen. And there is **no length cap** — a
300-char title yields a 321-byte name that Linux rejects with `Errno 36 File name too long`, so a
very long title cannot be saved. Verify a long name with a plain `open()` in `/tmp` rather than
driving the whole GUI.

## Prove formatting reaches the *printed PDF*, not just the preview

The high-value bug class is "control changes the preview but not the PDF". Print with Playwright
and inspect real PDF bytes with PyMuPDF (`pip install pymupdf`; **`poppler-utils` is absent** on
these boxes, so `pdftotext`/`pdftoppm` are unavailable).

Use `prefer_css_page_size=True` so the page's own `@page{size:A4 portrait}` decides the sheet —
otherwise you are asserting on a size your harness forced:

```python
await pg.pdf(path=out, prefer_css_page_size=True, print_background=True)
```

Change exactly **one** control per run against the same paper, then compare numbers — a
preview-only control shows an *identical* number. Measurable discriminators that all worked:

| Control | Assert in PDF bytes |
|---|---|
| `#fsize=16pt` | dominant span size 16.0 (default 11.5; Hindi default 12.49) |
| `#fline=1.8` | mean baseline gap ≈1.33× the `1.34` run |
| `#find=12mm` | part lines shift **+19.9pt** (12mm−5mm≈19.8pt); sub-parts ≈31.8pt, options ≈47.6pt (the ×1.6 / ×2.4 multipliers) |
| `#font=Times New Roman` | font family flips `LiberationSans` → `LiberationSerif` |
| `#boldq` unticked | bold char count drops sharply but **stays > 0** — the number and marks must remain bold |

For indent, extract per-line `bbox[0]` for lines matching `^([a-c]|[A-B]|i)\.\s` and diff the two
PDFs; the size/font counters alone cannot see indent.

### Font-fallback caveat (do not overclaim)
**Microsoft fonts are not installed on these boxes.** Chrome collapses every stack into just
`LiberationSans` (Default/Calibri, Arial, Verdana) or `LiberationSerif` (Times New Roman, Cambria,
Georgia, Comic Sans, both Hindi stacks). So **Calibri vs Arial is indistinguishable here** — only
the sans→serif *direction* is provable. Report the computed `font-family` string on `.page` as DOM
evidence and say plainly that individual named fonts would only differ on the school's Windows PCs.
Hindi PDFs render via `FreeSans` + a `Type3` font; that is normal.

## Checker behaviours that are correct and must not be "fixed"

- Words starting with a capital are **skipped** by the dictionary branch (`!/^[A-Z]/`), so Indian
  proper nouns (Rohan, Vinay, Nehru, Diwali, Krishna, Janmashtami) never false-positive. A
  capitalised typo like `Definately` is instead caught by the built-in `TYPOS` table.
- `known()` splits on hyphens/apostrophes, so `well-behaved`, `part-time` and curly-quoted
  `‘woman’` pass.
- British spellings (`colour`, `practise`, `organise`) are in the list.
- A wrong word in an **option row** (`i. tooth ii. tooths`) is downgraded to a **warning** ending
  `— in an option row, so left as typed`, and autofix (`#fix`) only ever rewrites `TYPOS`/`AMERICAN`
  entries — unknown-dictionary words are never touched. Assert this by **byte-diffing `#body`**
  before/after, and press `#fix` twice to prove idempotence.
- Inline `*bold*` / `~plain~` are stripped by `stripMarks()` before checking, so they must produce
  **no** "space before punctuation" / "starts with a small letter" findings, and no `*`/`~` may
  appear in the PDF text layer. Note `*bold*` on an already-bold question line is visually a no-op —
  only observable on plain lines (part lines, or a `~plain~` line).
- `isProse()` = `wordCount > 25`: a long passage typed on the `Q1.` line prints plain while the
  number and marks stay bold; a short instruction stays fully bold. Test both sides of 25.

## Pagination

`groupForPrint()` wraps a heading plus up to `KEEP_MAX = 4` short blocks in
`.secstart{break-inside:avoid}`, so headings are not stranded. To attack it, pad the paper with
N extra one-line questions before the second section and walk N so the boundary crosses the page
break — N from 0 to 18 covered it. Assert, per page, that a `SECTION`/`खंड` heading has **≥2 text
lines below it on the same page**.

**Known pre-existing defect:** at some lengths (N=10..13 with the appraisal sample) the PDF gains a
completely **blank trailing page** (`fonts=[]`, no ink). Verified byte-for-byte identical in the
pre-PR#2 `origin/main` copy, so it is **not** a regression of the file move — but it may still be
worth fixing, and a future change in this area might make it better or worse.

## Hindi

Select `#lang=hi`, then `#sample` loads a Hindi paper and sets Hindi title/subject/duration.
Expect Devanagari labels (`शैक्षणिक सत्र`, `कक्षा`, `विषय`, `समय`, `अधिकतम अंक`, `सामान्य निर्देश`,
`नाम`, `वर्ग`), `प्र1.` instead of `Q1.`, `खंड` headings, sub-parts `क./ख./ग./घ./ङ.`, and the
"Flag American spellings" row hides itself. The Hindi sample should report
`No errors found. 7 questions checked.` — English dictionary checks are switched off for Hindi.
In this repo the Hindi sample's General Instructions **are** in Hindi (an earlier sibling repo
shipped English ones — worth re-checking after any sample edit).

`xdotool type` drops non-ASCII, so **type Devanagari via clipboard** (`xclip -selection clipboard`
then ctrl+v) or `page.fill()`. Beware: **PyMuPDF text extraction drops the repha** (`वर्ग`→`वग`,
`पर्स`→`पस`), so an extraction "miss" is not a rendering bug — rasterise at ~200 dpi
(`page.get_pixmap(dpi=200)`) and confirm the glyphs visually before reporting anything.

## UI-driving gotchas

- The `#body` textarea is scrollable; a `scroll` over it scrolls the textarea, not the page. Scroll
  with the cursor over an area **outside** the textarea (e.g. the right column) to move the page.
- Loading a sample auto-reruns the checker if a report is already displayed, so `#report` may
  populate without you clicking `#check`.
- Formatting changes and `#lang` switches re-render the preview immediately; no button needed.

## Printed layout: option rows, trailing blanks, pictures

The printed sheet is meant to read like the school's own appraisal worksheet. Reference PDFs from
the school are the oracle — rasterise them with PyMuPDF and build a **labelled side-by-side
composite** with PIL, then open it in Chrome (`file:///…png`). That single image is the most
convincing artefact for a teacher reviewing the change; compare structure (header block, marks
column, one-line option rows, right-hand blanks, box-beside-picture, ruled answer lines), not
pixels — the school's Calibri cannot be resolved here.

**Option rows** (`.opts` flex + `.opts span{white-space:nowrap}`). Assert per row: all spans share
one `top` where they fit; `scrollWidth <= clientWidth + 1` (a nowrap column that wrapped internally
clips instead, so this catches it); no overlap for spans sharing a `top`; `span.m` (marks) is
rightmost and unstretched. **Long options can overflow the page at large font sizes** — at 16pt two
spans measured past the page's right content edge. Always re-measure span `right` against the page
right edge at 10pt *and* 16pt; a pass at the default size proves nothing.

**Trailing blanks.** `TRAIL_RE = /^(.+?)\s*_{3,}\s*$/` splits a trailing underscore run into a
`.tb` cell so the blank sits right, before the marks. Two things to check every time:
- `(.+?)` must consume ≥1 char, so a line whose text is **only** ≥4 underscores (`a. ____`) leaves
  a **literal `_`** in the text plus a blank. `___` (exactly 3) does not match and is safe. This bug
  class has been "fixed once" already and regressed — verify by reading the **PDF text layer** and
  counting literal `_` characters, not just eyeballing the preview.
- `.tb .blank{width:32mm}` is **fixed**, so trailing blanks are *not* length-scaled even though
  mid-line blanks (`withBlanks()`, `max(30,min(70,len*3))mm`) are. Different underscore run lengths
  print identical trailing blanks — a design inconsistency, not a crash.
Re-run trailing-blank checks at changed `#fsize`, `#fline` and `#find`: if the finding is identical
at all four settings it is a parser bug, not a layout artefact, and that distinction belongs in the
report.

**Pictures (`PIC:` lines, `#picfile`).** `PIC_RE = /^\s*(?:PIC|PICTURE)\s*:?\s*(\d+)?\s*$/i`,
default height 42mm. `pairBoxAndPicture()` merges a `BOX:` **immediately** followed by `PIC:` into a
`.boxpic` flex row; a blank line between them (kind `blank`) prevents pairing, and `PIC:` then
`BOX:` never pairs. `PICS[]` is module-level, indexed by **file** order, not persisted in
localStorage — so pictures survive typing / formatting changes / `#lang` switches / reloading the
sample, but are lost on F5. Useful fixtures: solid-colour PNGs with a giant white digit, matched by
**mean pixel colour** of each frame in the rasterised PDF so a mis-ordered mapping is obvious.
Gotchas:
- The school **logo** is also an image XObject (460×130 in these papers). Exclude it by dimension
  when counting picture images, or you will report phantom extra pictures.
- Fewer files than `PIC:` lines ⇒ empty bordered frames; the `<i>picture</i>` hint is
  `visibility:hidden` in print, so the word "picture" must not appear in the PDF text layer while
  the border must still be drawn. More files than `PIC:` lines ⇒ surplus silently ignored.
- `PIC: -5` and `PIC: abc` do not match `PIC_RE` and fall through to ordinary text (correct).
- **`PIC: 9999` has no cap** — it produced 36 pages, 34 of them blank. Always try a silly height.
- Arm a `page.on('request')` recorder *before* choosing files to prove nothing is uploaded; data
  URIs mean the expected result is **zero** new requests.

**Pagination.** `pic`/`boxpic` now break out of `groupForPrint()`'s section-keep group, so re-check
"no `SECTION` heading with <2 lines under it" across padding variants. The **blank trailing page**
is threshold-sensitive: PR-level changes shift *which* lengths produce it rather than creating or
curing it. Always print the same padded papers from a `git worktree` of `main`
(`git worktree add /tmp/…/mainwt main` + a second `http.server`) and present the two columns side by
side before attributing it to the change under test.

## Devin Secrets Needed

None — the page is static with no backend, no login and no API keys.
