---
name: seminar-schedule
description: >
  Enriches academic seminar day schedules by automatically researching each person on the meeting list
  and adding key background information. Use this skill whenever a visiting academic or researcher
  shares a seminar schedule, visit itinerary, or list of one-on-one meetings — even if they just
  paste a table of names and times, upload a PDF schedule, or say "can you help me prepare for my
  seminar visit", "I have a list of meetings with academics", or "I'm giving a talk at [institution]".
  Takes a schedule (PDF, pasted table, or spreadsheet) plus optional visitor context (draft paper or
  website), researches every person on it, and returns a single-page beautifully formatted PDF with
  biography, research interests, co-authors, and editorial roles for each meeting slot.
---
 
# Seminar Schedule Enricher
 
You help academics prepare for seminar visits by enriching their meeting schedule with background
research on every person they'll meet. Output is a polished, **single-page** landscape PDF.
 
---
 
## Inputs
 
1. **Schedule** (required). May arrive as:
   - A **PDF file** — extract with `pdfplumber` (see Step 0)
   - A plain text or Markdown table pasted into chat
   - A .xlsx or .csv file (use the xlsx/file-reading skill if needed)
 
2. **Visitor context** (optional, one or both):
   - **Draft paper**: PDF or text of the paper being presented
   - **Visitor website / CV URL**: The seminar visitor's own homepage
 
---
 
## Output
 
A **single-page landscape PDF** (A4) containing:
 
1. **Header**: "Seminar Schedule — [Institution] — [Date]" with visitor name and paper title
2. **Enriched schedule table** — one row per meeting slot (PhD slots combined; see below)
 
**No** highlights or "Key Connections" section — keep everything on one page.
 
### Columns
 
| # | Column | Notes |
|---|--------|-------|
| 1 | **Time** | Line-break at the en-dash: "9:45–<br/>10:15". If a room number or location is provided in the schedule, add it on a third line in small grey text: "[E264]". Zoom/location tag on a separate line if present. |
| 2 | **Name** | Clickable hyperlink to homepage. Role in smaller grey text on the line directly below the name — never on the same line. E.g. `Juliane Begenau` (link) / `Assoc. Prof.` (grey, smaller). |
| 3 | **Short Bio** | Compact timeline — **all years abbreviated to two digits** ('10 not 2010). Format: `PhD [School] '[YY] → [School] '[YY]–'[YY] → [Current] since '[YY]`. |
| 4 | **Research Interests** | 3–5 keyword phrases |
| 5 | **Co-authors** | Name (Institution), one per line, up to 5 |
| 6 | **Editorial Roles** | Role + journal abbreviation. "—" if none. |
| 7 | **Comment** | Always present. If a connection to the visitor exists (shared program, overlapping research): 1–2 short phrases. If no connection: 2–3 plain-language sentences describing what the person works on. Never mention joint publications. |
 
### Seminar slot rule
 
If the schedule includes a seminar presentation slot:
- Add a **single stand-out divider row** at the correct time position
- Background: **medium marine blue** `#1e6091`, white bold text, centred — sits visually between the dark navy header (`#1a2e4a`) and the light dinner blue (`#d8eaf7`), giving a clear three-level blue hierarchy across the page
- Text: `SEMINAR  ([duration if known, e.g. 90 min])`
- The row spans **all columns** (including Time) — no research content, no links
- Keep the row height minimal (it is a visual divider, not a data row)
 
```python
SEMINAR_ROW = len(table_data)
table_data.append([
	Paragraph(f'<b>SEMINAR  ({duration})</b>', seminar_sty),
	*[Paragraph("", cell_sty)] * (len(HEADERS) - 1)
])
# Then in TableStyle:
('BACKGROUND', (0, SEMINAR_ROW), (-1, SEMINAR_ROW), colors.HexColor('#1e6091')),
('SPAN',        (0, SEMINAR_ROW), (-1, SEMINAR_ROW)),
('ALIGN',       (0, SEMINAR_ROW), (-1, SEMINAR_ROW), 'CENTER'),
('TOPPADDING',  (0, SEMINAR_ROW), (-1, SEMINAR_ROW), 3),
('BOTTOMPADDING',(0,SEMINAR_ROW), (-1, SEMINAR_ROW), 3),
```
 
### PhD meeting slot rule
 
If the schedule has a "PhD Meeting" slot listing **multiple PhD students**:
- Collapse them into **one row** in the table
- Name cell: "PhD Meeting: Name1 · Name2\n(PhD students)"
- Bio, interests, co-authors: combine concisely across all students
- If only **one** PhD student is listed, still use one row labelled clearly
 
### Dinner slot rule
 
If the schedule includes a dinner:
- Add a **visually distinct dinner row** at the correct time position (dark navy background, white text, spans all columns except Time)
- Format: `Dinner @ [Restaurant]  ·  Name1 (Role) · Name2 (Role) · Name3 (Role)`
- Each name should be a clickable link to their homepage
- If dinner participants are **already on the schedule**, link back to their existing rows — no need to repeat their full research profile
- If dinner includes **new people not on the daytime schedule**, research them just like any other slot and add a full row (or a condensed separate section below the main table)
 
---
 
## Step 0 — Parse the Input Schedule
 
### PDF input
 
```bash
pip install pdfplumber --break-system-packages
```
 
```python
import pdfplumber, pandas as pd
 
with pdfplumber.open("schedule.pdf") as pdf:
	rows = []
	for page in pdf.pages:
		tables = page.extract_tables()
		if tables:
			for t in tables:
				rows.extend(t)
		else:
			print(page.extract_text())   # inspect manually
 
if rows:
	df = pd.DataFrame(rows[1:], columns=rows[0])
```
 
If no table structure is found, parse raw text line-by-line. Common patterns:
- `10:00  John Smith  Stanford`  (whitespace-delimited)
- `10:00 - John Smith (Stanford Finance)`  (parenthetical)
 
For **scanned PDFs** (no selectable text):
```bash
pip install pytesseract pdf2image --break-system-packages
```
```python
from pdf2image import convert_from_path
import pytesseract
images = convert_from_path("schedule.pdf")
text = "\n".join(pytesseract.image_to_string(img) for img in images)
```
 
Extract at minimum: **time slot**, **name**, **affiliation**. Preserve all other original columns.
 
---
 
## Step 1 — Research Each Person
 
**Always use `web_search` and `web_fetch` for every person — do not rely on training-data memory.** People move institutions, change roles, and publish new work continuously. Even for well-known researchers, search first: a single hallucinated affiliation or stale editorial role undermines the whole document. The only acceptable exception is if you just researched the same person moments earlier in the same session.
 
For **each person** on the schedule, run these steps. Work through all people before generating the PDF.
 
### 1a — Find the homepage
 
```
Search: "[Full Name] [Institution] economics finance homepage"
```
 
Prefer the person's **own university faculty page**. `web_fetch` it; look for CV link (always fetch if present), bio blurb, publications list, editorial/service section.
 
### 1b — Short bio timeline
 
Extract from CV or homepage:
1. **PhD**: Institution and year conferred
2. **Career history**: All positions with years
3. **Current position**: Institution and year joined
 
Format as compact timeline with **two-digit years**: `PhD [School] '[YY] → [School] '[YY]–'[YY] → [Current] since '[YY]`
 
Example: `PhD Chicago '07 → UT Austin '07–'17 → Kellogg since '17`
 
If the PhD year is not clearly stated, omit the year rather than guess.
 
### 1c — Research interests
 
From CV "Research Interests" / "Fields" section, or infer from paper titles. 3–5 concise phrases. Prefer stated interests over inferred.
 
### 1d — Co-authors
 
Identify names appearing **3+ times** on the publications list (or 2+ if fewer than 10 papers). Find each co-author's current institution. Format: `Firstname Lastname (Institution)`. Up to 5.
 
### 1e — Editorial roles
 
**Source A — CV / homepage** ("Professional Service" / "Editorial" section). Most reliable.
 
**Source B — Aggregator**: Try fetching `https://christiankontz.com/editors` (JS-rendered — if `web_fetch` returns blank, skip to C).
 
**Source C — Direct search**:
```
Search: "[Full Name]" editor "Journal of Finance" OR "RFS" OR "JFE" OR "AER"
```
Or fetch editorial boards directly:
- JF: `https://afajof.org/editorial-board/`
- JFE: `https://www.sciencedirect.com/journal/journal-of-financial-economics/about/editorial-board`
- RFS: `https://academic.oup.com/rfs/pages/Editorial_Board`
- AER: `https://www.aeaweb.org/journals/aer/editors`
 
Abbreviations: JF, JFE, RFS, AER, QJE, ReStud, JPE, JFQA. Roles: "Editor", "Assoc. Editor", "Editorial Board".
 
### 1f — Comment column
 
The column header is always **"Comment"**.
 
Write in **second person** ("your paper", "your model"). Never refer to the visitor by name or in third person. Use neutral, factual language — no evaluative phrases.
 
**The primary purpose is to summarise what the person is currently working on** — active papers, recent drafts, ongoing projects. If there is natural institutional overlap (same PhD institution, same prior employer), note it briefly. Do not manufacture research connections.
 
**Structure to aim for:**
- 2–3 sentences on their current/recent work, specific enough to be useful
- 1 interesting tidbit if available: unusual career path, notable dataset, policy role, hobby, long institutional tenure, etc.
 
**Finding current work**: look at the "Working Papers" section of their homepage, not just published papers. These reflect what they'll actually want to talk about.
 
**Institutional overlap**: only mention if genuine — same PhD institution, postdoc, or prior faculty position as you. Don't mention unless the overlap is specific.
 
**Do not:**
- Refer to the visitor by name
- Mention joint publications (the presenter knows their own co-authors)
- Force a thematic connection if none is natural
- Use phrases like "important meeting", "great conversation starter", "rising scholar"
 
---
 
## Step 2 — Ambiguity Handling
 
- **Name collisions**: Use affiliation to disambiguate; if still uncertain flag with ⚠
- **PhD students**: Note "PhD student, [Nth yr]" and advisor if known; leave editorial blank
- **No web presence**: After 2 failed searches, leave enrichment fields blank; add ⚠
 
---
 
## Step 3 — Generate the PDF (single page)
 
```bash
pip install reportlab --break-system-packages
```
 
### Layout
 
- **Page**: Landscape A4 (`from reportlab.lib.pagesizes import A4, landscape`)
- **Margins**: 1.2 cm all sides to maximise usable width
- **Font sizes**: Header row 7pt bold; cells 6.0–6.5pt (reduce if schedule is long); name 6.3–6.8pt bold
- **Header row**: Dark navy `#1a2e4a`, white bold text, gold `#c8a84b` rule below
- **Body rows**: Alternating white / light grey `#f0f4f8`; PhD student rows in light green `#e8f0e8`
- **Dinner row**: Light steel-blue `#d8eaf7`, dark navy text `#1a2e4a`, gold rule above. Cols 1–6 spanned into one cell. **Do not use a dark background** — it makes the row too heavy visually.
- **Grid**: thin `#c8d0d8`
- **Cell text**: All via `Paragraph` so text wraps (never clips)
- **One-page enforcement**: if the schedule has 12+ rows, reduce cell padding to 1pt and font to 5.9pt
 
### Clickable name with role on second line
 
The name is a hyperlink; the role appears below it in smaller grey text — never on the same line:
 
```python
def name_cell(name, role, url):
	return Paragraph(
		f'<a href="{url}" color="#2563a8"><u>{name}</u></a><br/>'
		f'<font size="5.8" color="#4a5568">{role}</font>',
		name_style
	)
```
 
### Time cell formatting
 
Break at the en-dash so start and end times read on separate lines. If the schedule provides a room number or location, add it on a third line in smaller grey text:
 
```python
def time_cell(slot, room=None):
	"""slot: e.g. '9:15–9:45', room: e.g. '9-93' or 'E264' (optional)"""
	text = slot.replace('–', '–<br/>')
	if room:
		text += f'<br/><font size="5.6" color="#4a5568">[{room}]</font>'
	return Paragraph(text, time_style)
```
 
When parsing the schedule, extract the room/location column and pass it to `time_cell`. If no room is given, pass `room=None` and it is omitted cleanly.
 
### Column widths (fractions of usable width)
 
| Column | Fraction |
|--------|----------|
| Time | 5.5% |
| Name (Role) | 10.5% |
| Short Bio | 18.5% |
| Research Interests | 14% |
| Co-authors | 13.5% |
| Editorial | 8.5% |
| Connection | 29.5% (or redistribute if omitted) |
 
### One-page enforcement
 
After building the story, call `doc.build(story)`. If the PDF overflows to 2 pages:
1. Reduce cell font size from 6.5 to 6.0pt
2. Reduce top/bottom cell padding from 3 to 2pt
3. Shorten bio strings (drop pre-PhD positions if necessary)
4. Shorten connection strings to one phrase
 
```python
# Quick page-count check
from pypdf import PdfReader
n = len(PdfReader(output_path).pages)
if n > 1:
	# reduce sizes and rebuild
```
 
### Important: never use Unicode sub/superscripts
 
Use `<sub>`/`<super>` XML tags in Paragraph objects instead.
 
---
 
## Tips
 
- Passing the visitor's website/CV always pays off — the Connection column is the most valuable.
- For 10+ people, research will take several minutes.
- If an output looks wrong: "Re-research [Name] — I think they moved to [Institution]."
- If a PhD year cannot be confirmed, omit it rather than guess — accuracy matters.