# PPIS Question Paper Maker

A single page that formats a question paper in the school appraisal worksheet layout,
checks it for mistakes and prints it as a PDF on the school letterhead.

Live: https://alishakanwar-crypto.github.io/ppis-question-papers/

The paper stays on the teacher own computer; nothing is uploaded.

A paper already typed in Word can be used too: under **2 Questions**, choose the `.docx` file.
The page reads it in the browser, checks it and opens the Save-as-PDF window. With **Keep my Word
layout** ticked (the default) the paper prints as it was laid out in Word: every box, table, text
box, drawn box and picture stays where the teacher put it; only the school logo, the page border and
the school font are added (Word's own letterhead logo is left out). Text boxes and shapes become boxes
of the same size just under their line. Unticked, the questions are rewritten in the maker's own
lines (`Q1.`, `a.`, `(a) … (b) …`, `MATCH:`, `BOX:`, `TABLE:`, `FRAME:`, `LINES:`, `PIC:`) and laid
out in the maker's format; editing box 2 after an upload also switches to that layout. The Word
layout is drawn with [docx-preview](https://github.com/VolodymyrBaydalka/docxjs) 0.3.6 (Apache-2.0)
and [JSZip](https://stuk.github.io/jszip/) 3.10.1 (MIT), kept in `vendor/`. Old `.doc` files must first be saved as `.docx` in Word.
