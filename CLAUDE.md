# CLAUDE.md — gcse-flashcards (student app)

## Project overview
Student-facing GCSE revision flashcard platform. Built with React + Vite. Deployed at https://gcse-flashcards-two.vercel.app via Vercel. Connected to a shared Supabase database.

## Companion app
`~/gcse-admin` — local-only admin tool for uploading PDFs and generating flashcards via Claude API. Never deployed. Shares the same Supabase database.

## Tech stack
- **Frontend:** React + Vite
- **Database:** Supabase (PostgreSQL) — project ID `wfplztunqgmifjlumkqw`
- **Hosting:** Vercel
- **Auth:** Not yet implemented (future state)
- **Styling:** Plain CSS (App.css) — no Tailwind, no component library

## Repository
- GitHub: `https://github.com/ozy010/gcse-flashcards`
- Branch: `main`
- Every push to main triggers an automatic Vercel redeploy

## Environment variables
Set in `.env` locally and in Vercel dashboard:
```
VITE_SUPABASE_URL=https://wfplztunqgmifjlumkqw.supabase.co
VITE_SUPABASE_ANON_KEY=sb_publishable_...
```

## File structure
```
gcse-flashcards/
├── src/
│   ├── App.jsx          — main app, all screens in one file
│   ├── App.css          — all styles
│   ├── supabaseClient.js — Supabase client initialisation
│   └── main.jsx         — React entry point
├── .env                 — local env vars (not committed)
├── index.html
├── package.json
└── vite.config.js
```

## App structure
Single file app (`App.jsx`) with three screens managed by a `screen` state variable:

- `SCREEN.SELECT` — paper/year/topic selector
- `SCREEN.CARDS` — flashcard flip interface
- `SCREEN.SUMMARY` — session results with retry missed cards

## Database schema
Six tables in Supabase:

### `exam_board`
| column | type | notes |
|--------|------|-------|
| id | uuid PK | |
| name | text | e.g. 'Edexcel' |
| slug | text | e.g. 'edexcel' |

### `subject`
| column | type | notes |
|--------|------|-------|
| id | uuid PK | |
| board_id | uuid FK | → exam_board |
| name | text | e.g. 'Geography B' |
| qualification | text | e.g. 'GCSE' |
| slug | text | e.g. 'geography-b' |

### `paper`
| column | type | notes |
|--------|------|-------|
| id | uuid PK | |
| subject_id | uuid FK | → subject |
| paper_number | text | e.g. '2', '3' |
| paper_name | text | e.g. 'The Human Environment' |
| year | int | e.g. 2024 |
| season | text | 'summer' or 'november' |
| source_type | text | 'past_paper', 'textbook', 'revision_guide' |

### `topic`
| column | type | notes |
|--------|------|-------|
| id | uuid PK | |
| subject_id | uuid FK | → subject |
| name | text | e.g. 'Urbanisation' |
| slug | text | e.g. 'urbanisation' |
| display_order | int | controls sort order in UI |

### `flashcard`
| column | type | notes |
|--------|------|-------|
| id | uuid PK | |
| paper_id | uuid FK | → paper |
| topic_id | uuid FK | → topic |
| question_ref | text | e.g. 'Q1a' |
| question | text | |
| answer | text | |
| mark_scheme_notes | text | optional examiner notes |
| marks | int | |
| command_word | text | define, describe, explain, evaluate, assess, state, suggest |
| difficulty | text | easy, medium, hard |
| is_published | boolean | only published cards shown to students |

### `user_progress`
| column | type | notes |
|--------|------|-------|
| id | uuid PK | |
| user_id | uuid | nullable — null = anonymous session |
| flashcard_id | uuid FK | → flashcard |
| result | text | 'knew' or 'missed' |
| ease_factor | numeric | SM-2 spaced repetition |
| next_review | timestamptz | SM-2 scheduling |
| reviewed_at | timestamptz | |

## RLS policies
- All reference tables (exam_board, subject, paper, topic) — public read
- flashcard — public read where is_published = true
- user_progress — anyone can insert; users can only read their own rows
- Auth not yet enforced (PoC mode)

## Seeded data (current PoC)
- **Exam board:** Edexcel
- **Subject:** GCSE Geography B
- **Papers:** Paper 2 and Paper 3, years 2022, 2023, 2024 (summer)
- **Topics:** Urbanisation, Global development, Resource management, Climate change, Ecosystems, UK landscapes
- **Flashcards:** 9 cards for Paper 2 (2024) and 9 cards for Paper 3 (2024). Years 2022 and 2023 have no cards yet.

## Key Supabase queries used in the app

### Load papers (select screen)
```js
supabase.from("paper")
  .select(`id, paper_number, paper_name, year, season,
    subject!inner ( id, name, qualification,
      exam_board!inner ( name, slug ) )`)
  .eq("subject.exam_board.slug", "edexcel")
  .order("paper_number")
  .order("year", { ascending: false })
```

### Load topics
```js
supabase.from("topic")
  .select("id, name, slug, display_order")
  .eq("subject_id", selectedPaper.subject.id)
  .order("display_order")
```

### Load flashcards for a session
```js
supabase.from("flashcard")
  .select("id, question_ref, question, answer, marks, command_word, difficulty, topic_id")
  .eq("paper_id", selectedPaper.id)
  .eq("topic_id", topicId)   // omit for all topics
  .eq("is_published", true)
```

## Conventions
- Supabase client imported from `./supabaseClient` in all files
- All env vars prefixed with `VITE_` for Vite compatibility
- No TypeScript — plain JavaScript throughout
- CSS uses plain class names, no modules
- Cards shuffled client-side on session start using Fisher-Yates-style sort

## Current student UX flow
1. Select paper (Paper 2 or Paper 3)
2. Select year (2022, 2023, 2024)
3. Select topic (or All topics)
4. Click Start revision
5. Cards shown one at a time — tap to flip, mark Got it / Missed it
6. Session summary shows score breakdown
7. Option to retry missed cards

## Future work
- [ ] Add more flashcards for 2022 and 2023 papers
- [ ] Add more subjects (AQA, OCR, other Edexcel subjects)
- [ ] Optional student login with progress tracking across sessions
- [ ] Spaced repetition (SM-2) for logged-in users
- [ ] Multiple choice card type (schema ready, needs card_type field)
- [ ] Custom domain
- [ ] Filter cards by command word (describe, explain, evaluate etc.)
- [ ] "Smart revision" mode surfacing weakest cards

## Admin tool (gcse-admin)
Content is added via the separate `~/gcse-admin` local tool — see that repo's CLAUDE.md for details. The admin tool uploads PDFs, sends them to Claude API for Q&A extraction, allows review, then inserts approved cards directly into the shared Supabase database.
