# Yoga KB 🧘
**A safety-aware yoga knowledge base.** Tell it your health conditions and it shows which of about 500 yoga practices are *safe*, *need modification*, or should be *avoided*, with the reason and a source each time.

DBMS Experiential Learning project · PostgreSQL + MongoDB + Python

---

## Why
Yoga apps list poses, but none of the free ones can answer *"I have glaucoma and a bad knee, what's safe for me?"* with structured, sourced data. That's the gap we fill.

## Stack
| Part | Tech | Holds |
|---|---|---|
| Relational DB | PostgreSQL (Neon / Supabase) | Practices, names (English / Sanskrit / Kannada), categories, benefits, conditions, contraindications, sources, user data |
| NoSQL DB | MongoDB (Atlas) | Step-by-step instructions, practice session logs |
| App | Python + Streamlit | Browse, search, "safe for me", favourites, practice log, admin |

## Status
🟡 **Design phase.** The plan is complete and no code has been written yet. Live progress is in [AGENTS.md → Status Board](AGENTS.md#5-status-board).

## Docs: start here
| File | What's inside | Read time |
|---|---|---|
| [docs/PRD.md](docs/PRD.md) | What we're building: features, user stories, rubric mapping | 10 min |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | How it's built: diagrams, all 16 tables, security, decisions | 15 min |
| [docs/WORKFLOW.md](docs/WORKFLOW.md) | Step 0 → submission, with a checklist for each step | 10 min |
| [docs/RESEARCH.md](docs/RESEARCH.md) | Datasets, image sources, evidence, competitors | 10 min |
| [AGENTS.md](AGENTS.md) | Build manual, plus the live status, change log and open questions | 5 min |

## The plan in one picture
```
Design ──► Database ──► App ──► Deploy + freeze v1.0-mvp ──► Innovations ──► Report
(spec, ER,   (schema,     (search,   (safe working version     (fuzzy search,
 3NF)         seed ~500)   safe-for-me) saved)                   triggers, sequences…)
```

## Run it
*Coming in Step 0.* You'll need Python 3.12+, a `.env` file (copy `.env.example`), and access to the two cloud databases.

## Team
- Nishanth
- `<teammate>`

## Disclaimer
This is an educational database project, **not medical advice**. Talk to a qualified professional before starting any yoga practice if you have a health condition.
