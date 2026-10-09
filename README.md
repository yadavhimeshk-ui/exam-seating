# exam-seating
A platform that generate the exam seating arrangement using uploaded excel or csv file
# Automated Exam Seating Arrangement Generator

Upload a student list → enter exam details → configure rooms → preview → download a
Word (.docx) file with **one page per room**.

## Run
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn backend.main:app --reload
# open http://127.0.0.1:8000
```
Tests (no server needed): `python -m tests.test_core`

## Features added in v2
* **Landing page** with hero, 4-step walkthrough, important-notice card, developer credit, footer badges.
* **English / हिंदी** switch in the top bar. All UI text updates instantly; the exported DOCX/PDF use
  Hindi labels when Hindi is selected (roll numbers, room codes and text you typed stay unchanged).
* **Editable review grid**: edit roll numbers, column headings, course headings and room names; swap two
  seats (click one ⇅ handle then another, or drag). Duplicates are highlighted. Edits are sent to the
  server with the export request (`plan` field) and re-validated in `backend/plan_edit.py`.
* **DOCX and PDF export**. PDF is converted from the same DOCX (one room per page) via LibreOffice
  (or `docx2pdf` + MS Word). If neither is installed the app opens the browser print dialog instead,
  with one room per printed page.
* **Different rooms, different sizes**: every room has its own Columns / Rows / Seats-per-bench (step 3).
  In the review step each room also has +/− Rows and +/− Columns to adjust after seeing the result
  (a row/column can only be removed when it is empty).
* **Unequal columns**: in step 3 press **Custom** on a room to give every column its own number of rows
  (capacity = sum of the rows × seats per bench). The review step has a +/− on each column as well as
  +/− for the whole room, and the Word/PDF page draws each column at its own length.
* **One page per room, always**: each room picks its own orientation (portrait / landscape) and tightens
  spacing and type size only when needed. Standard rooms are unchanged. If a room is so large that it cannot
  fit even in the tightest layout, the preview shows a warning instead of silently spilling onto a 2nd page.
* **Generate Again** asks for confirmation, then deletes the server-side session and returns to `/`.

### PDF export setup
Install LibreOffice (free, https://www.libreoffice.org). It is found automatically on Windows, macOS
and Linux. Optional alternative on a PC with Microsoft Word: `pip install docx2pdf`.

## Structure
```
backend/parsing.py       xlsx/xls/csv/pdf -> DataFrame, column auto-detection
backend/allocator.py     course grouping, lane interleaving, room distribution, warnings
backend/docx_builder.py  python-docx page layout (header, grid, summary, signatures)
backend/plan_edit.py     validates + recomputes a plan edited in the browser
backend/pdf_export.py    DOCX -> PDF (LibreOffice / docx2pdf)
backend/main.py          FastAPI: /api/upload, /api/preview, /api/generate, /api/generate-pdf,
                         DELETE /api/session/{id} + static UI
frontend/                landing page + 4-step wizard (plain HTML/CSS/JS, no build step)
                         i18n.js = English/Hindi dictionary
sample/Student_Records.xlsx
```

## How seating works
* A bench has *S* seats (default 2) → each column has *S* **lanes** (the sub-columns).
* Courses are packed into lanes largest-first; a lane is filled top-to-bottom, column after
  column, in roll-number order. Neighbours on a bench therefore always come from different
  lanes (= different courses). When a lane holds two courses, the switch is labelled in the
  cell (like "BTCS II" in Column III of the printed sheet).
* A course is split across lanes only when a lane would overflow. If one course is bigger than
  half the seats, a warning reports how many benches pair same-course students.
* Fill strategy: *sequential* (fewest rooms, last room partly filled) or *balanced* (even load).
* Capacity shortage → warning with the number of students that cannot be seated.

## Course column missing?
`Student_Records.xlsx` has no Course/Branch column. In step 1 either map one, define
roll-number ranges per course (e.g. `BCOM II: 26100091 – 26100150`), or tick *single group*.
