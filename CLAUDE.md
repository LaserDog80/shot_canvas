# CLAUDE.md

> **Purpose:** Persistent instructions for Claude when working in this project.  
> **Rule:** Edit specific sections. Never rewrite this file entirely.

---

## Project Overview

ShotBoard — A single-file HTML canvas app for visually organising video files into Shots and Scenes, then batch-renaming and exporting them.

---

## Tech Stack

- **Language:** HTML/CSS/JavaScript (single file, no build step)
- **Browser:** Chrome only (requires File System Access API)
- **Dependencies:** Google Fonts (Inter + JetBrains Mono) — no other external deps
- **Reference:** `_reference/shottriage.html` — CSS design system and file ingestion reused from this

---

## Development Workflow

```bash
# 1. Create virtual environment (first time only)
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Run tests
pytest

# 4. Run local server (if web-based)
python app.py  # or: flask run / uvicorn main:app

# 5. Before committing
pytest && git add -A && git commit -m "message"
```

---

## Code Standards

- Use type hints for function signatures
- Docstrings for public functions
- Keep functions under 30 lines where possible
- Prefer explicit over clever

---

## File Structure Conventions

```
project/
├── src/              # Main source code
├── tests/            # Test files (mirror src/ structure)
├── requirements.txt  # Dependencies
├── app.py            # Entry point (if web-based)
└── README.md         # User-facing documentation
```

---

## Known Issues & Workarounds

<!-- Append new issues here as they're discovered -->

- None yet

---

## Do NOT Do

<!-- Append items here when Claude makes mistakes -->

- Do not rewrite CLAUDE.md or LEARNINGS.md entirely—edit sections only
- Do not delete test files without explicit approval
- Do not change the directory structure without discussing first
