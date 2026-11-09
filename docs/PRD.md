# Product Requirements Document (PRD): Yoga Knowledge Base

| | |
|---|---|
| **Product** | Yoga KB: a safety-aware, queryable yoga knowledge base |
| **Course** | DBMS Experiential Learning (EL), batch of two |
| **Version** | 1.0 (MVP scope) |
| **Status** | Approved for build |
| **Last updated** | 2026-11-09 |
| **Related docs** | [ARCHITECTURE.md](ARCHITECTURE.md) · [WORKFLOW.md](WORKFLOW.md) · [RESEARCH.md](RESEARCH.md) · [../AGENTS.md](../AGENTS.md) |

---

## 1. Problem statement
Plenty of yoga apps list poses. None of the free ones can answer this simple question with data:

> "I have glaucoma and a knee injury. Which poses are safe for me, and why?"

Existing tools either:
- put safety information in free text that you can't filter (Pocket Yoga), or
- sit behind a paywall (Yoga Journal), or
- are closed sequence generators with no browsable knowledge (Down Dog).

The open datasets are small and mostly don't say where their facts come from.

## 2. Product vision
A yoga knowledge base where **every safety claim is structured, sourced and queryable**. A user records their health conditions once, and the system labels about 500 practices *avoid*, *modify*, *caution*, or *no known rule*. It shows the reason and the source for every rule it finds. *No known rule* means no sourced rule was found, and it is not a promise that the practice is safe.

**One-line pitch:** *"Not the biggest pose list, but the safest one to search."*

## 3. Goals and non-goals

### Goals (MVP)
| ID | Goal |
|---|---|
| G1 | Store about 500 yoga practice entries (asanas, their variations, pranayama, mudra, bandha, kriya, meditation) in a normalized (3NF) relational schema. |
| G2 | Show integrated SQL + NoSQL design. PostgreSQL holds structured, integrity-critical data. MongoDB holds document-shaped content and logs. |
| G3 | Provide a working **"safe for me"** filter driven by contraindication data that cites its source. |
| G4 | Let a user browse, search and filter practices, and view a full detail page. |
| G5 | Let a user keep favourites and a practice log. |
| G6 | Run a simple deployment that works as a demo for the course and the resume. |

### Non-goals (MVP), parked for phase 2
- Multi-user accounts, login, roles. *The MVP has one user.*
- Sharing sequences, social features.
- The automatic sequence builder.
- Pose relationships (prepares-for, counter-pose) beyond "variation of".
- Kannada instructions (Kannada **names** are in scope; Kannada **steps** are not).
- Devanagari script names.
- An expert review workflow (no reviewer is available yet).
- Camera or computer-vision pose correction (dropped entirely, out of scope for a DBMS course).
- A mobile app.

## 4. Users and personas
| Persona | Description | MVP? |
|---|---|---|
| **Practitioner (primary)** | Someone who practises yoga and has one or more health conditions. They want to know what's safe. In the MVP this is the single app user. | ✅ |
| **Curator (us)** | The two project members. We load, edit and source the data through admin screens or scripts. | ✅ |
| **Guest** | Browses without an account. In the MVP the single user covers this. | Phase 2 |
| **Reviewer** | A yoga teacher or physio who approves safety data. | Phase 2 |

## 5. User stories and acceptance criteria
Priority uses **MoSCoW**: Must, Should, Could, Won't (this release).

### Must
| ID | User story | Acceptance criteria |
|---|---|---|
| US1 | As a practitioner, I want to **browse all practices** so I can explore what exists. | Shows a paginated list with English name, Sanskrit name, type and difficulty. It can be filtered by type. |
| US2 | As a practitioner, I want to **search by name** so I can find a pose quickly. | Matches the English, Sanskrit (IAST) or Kannada name or any alias, ignoring case. "Vrikshasana" and "Vrksasana" both find Tree Pose, through the alias table. |
| US3 | As a practitioner, I want to **filter by category, difficulty, body part and benefit**. | Filters can be combined (AND). The result count is shown. |
| US4 | As a practitioner, I want a **detail page** for each practice. | Shows names in 3 languages, type, difficulty, description, categories, body parts, benefits, steps (from MongoDB), primary image with credit, contraindications with severity and source, and the parent and child variations. |
| US5 | As a practitioner, I want to **record my health conditions** with optional start and end dates. | I can add or remove conditions from the condition list. A condition whose end date has passed becomes inactive automatically. |
| US6 | As a practitioner, I want a **"safe for me" filter**. | Every practice is labelled *no known rule*, *caution*, *modify* or *avoid* against my active conditions, and *no known rule* carries a line explaining that it does not mean safe. *Avoid* practices can be hidden. Each label shows the condition, the note and the source. |
| US7 | As a practitioner, I want **favourites**. | I can add and remove a favourite and see the list. Adding the same favourite twice is blocked by the database. |
| US8 | As a practitioner, I want to **log a practice session**. | A session has a date, items (practice, hold seconds, felt 1–5, pain yes/no) and notes. It's saved to MongoDB, and I can see my past sessions. |
| US9 | As a curator, I want **admin create/read/update/delete (CRUD)** for practices, conditions, contraindications and sources. | A contraindication can't be saved without a source (enforced by the database). |
| US10 | As a curator, I want to **prove the entry count**. | A count view shows totals by type and a grand total of at least 450. |

### Should
| ID | User story |
|---|---|
| US11 | Simple **stats** on the practice log: sessions per week, most-practised poses (MongoDB aggregation). |
| US12 | A **medical disclaimer** on every safety screen. *(Treat as Must for any public deploy.)* |
| US13 | **Image credits** shown everywhere an image appears. |

### Could
| ID | User story |
|---|---|
| US14 | A Kannada UI label toggle. |
| US15 | Export the practice log as CSV. |

### Won't (this release)
Everything under Non-goals in section 3.

## 6. Functional requirements
| ID | Requirement | Traces to |
|---|---|---|
| FR1 | The system stores practices with English, Sanskrit (IAST) and Kannada names, a type, difficulty (1–3), a description and an optional parent (variation-of). | US1, US4 |
| FR2 | The system stores alternate spellings (aliases) with a normalized search key. | US2 |
| FR3 | The system links practices to categories, body parts (with an action) and benefits, all many-to-many. | US3 |
| FR4 | The system stores contraindications at two levels: per practice and per category. Each has a severity (avoid / modify / caution), a note, an evidence level and a **mandatory source**. | US6, US9 |
| FR5 | **Effective contraindication rule** (the key business rule). For a practice P and a condition C, decide in this order: (1) if P has its own rule, use it; otherwise (2) take the highest severity among the nearest ancestor's own rule and all category rules for the categories of P and of its ancestors; if there are none, (3) *no known rule*. Only a rule written for that exact practice can relax a stricter one. | US6 |
| FR6 | A user's active conditions are those where `start_date ≤ today` and (`end_date` is empty or `end_date ≥ today`). | US5, US6 |
| FR7 | Instruction content (steps, tips, mistakes, breathing cue) is stored per practice per language in MongoDB. | US4 |
| FR8 | Practice sessions are stored as documents in MongoDB, referring to practices by `practice_id`. | US8 |
| FR9 | Images are stored as URLs or paths with author and licence (both mandatory). Only CC0, CC BY and CC BY-SA images are allowed. | US4, US13 |
| FR10 | Every SQL statement the app runs is parameterized (embedded or dynamic SQL through the Python driver). | Security |

## 7. Non-functional requirements
| Category | Requirement |
|---|---|
| **Data integrity** | Primary keys, foreign keys, UNIQUE, CHECK and NOT NULL constraints enforce the rules *in the database*, not only in the app. |
| **Normalization** | The relational schema is in 3NF, with the reasoning written down. |
| **Performance** | Any page loads in under 2 s on free-tier hosting with about 500 practices. Indexes on every foreign key and search key. |
| **Security** | No secrets in the repo (`.env`). Parameterized SQL. The app connects with a least-privilege database role. User input never reaches a Mongo query as an operator (`$ne`, `$gt`, …). |
| **Privacy** | Health conditions are sensitive. Store only what's needed, keep them deletable, and never log them. |
| **Licensing** | No copyrighted text or images. Facts are rewritten in our own words, and sources are cited. |
| **Usability** | Pure-Python UI, readable on a laptop and a phone browser. |
| **Portability** | The schema is set up from scripts in the repo (`db/`). The data is reloaded from seed scripts. |
| **Encoding** | UTF-8 end to end, so Kannada script renders correctly. |
| **Time** | All timestamps are UTC. |

## 8. Data requirements (summary)
The full specification lives in [ARCHITECTURE.md §6](ARCHITECTURE.md#6-data-architecture).

| Store | Holds |
|---|---|
| PostgreSQL (16 tables) | practice, alias, category, practice_category, body_part, practice_body_part, benefit, practice_benefit, condition, contra, category_contra, source, media, app_user, user_condition, favourite |
| MongoDB (2 collections) | practice_content, practice_log |

**Volume target:** about 500 practices. The rough split is 150–250 base asanas, 150–250 variations, 15–20 pranayama, 30–50 mudra, 4 bandha, 6–15 kriya and about 10 meditation.

## 9. Success metrics
| Metric | Target |
|---|---|
| Practice entries | ≥ 450 (aim for 500) |
| Contraindication rows without a source | 0 (enforced) |
| Base asanas with at least one image (with credit) | ≥ 60 % |
| Practices with English steps in MongoDB | ≥ 80 % of base asanas |
| Practices with a Kannada name | ≥ 90 % |
| Safety-filter test cases passing | 100 % |
| Course rubric items covered | 100 % (see section 10) |

## 10. Course rubric mapping
| Rubric item | Where it's met |
|---|---|
| Full data requirement specification | This PRD (sections 6–8) + ARCHITECTURE section 6 |
| ER diagram | WORKFLOW step 2 |
| 3NF normalization | WORKFLOW step 3 (`docs/normalization.md`) |
| Integrated SQL + NoSQL | ARCHITECTURE sections 5–6, practice detail page, practice log |
| Embedded / dynamic SQL | Python data-access layer (psycopg, parameterized) |
| Security section | ARCHITECTURE section 9 |
| Recent trends | Polyglot persistence; fuzzy multilingual search with `pg_trgm` (phase 2); NLP condition search (phase 2) |
| Societal concern | Preventive health, AYUSH / International Day of Yoga alignment, regional language (Kannada), safety for older users and people with chronic conditions |

## 11. Assumptions and constraints
- Free tiers only: Neon or Supabase (Postgres), MongoDB Atlas M0, and Render or Streamlit Community Cloud.
- Two team members. Neither is a yoga or medical expert.
- Safety data comes from published sources (AYUSH Common Yoga Protocol, peer-reviewed studies, reputable references). Where no source exists, **no rule is published**.
- This is not medical advice. It's an educational database project.

## 12. Risks
| Risk | Impact | Mitigation |
|---|---|---|
| A real contraindication is missed or shown too leniently | High (user safety) | Mandatory sources; category rules as a safety net; disclaimer; a phase 2 reviewer |
| Can't reach about 500 entries | Medium (grading) | Count variations honestly; add pranayama, mudra, kriya; the prof accepts some flexibility if features are strong |
| Image licence violations | Medium | NOT NULL author and licence; only CC0, CC BY and CC BY-SA |
| Two databases drift out of sync | Medium | No hard deletes; a consistency-check script |
| Free-tier limits or sleep | Low | Small data; keep-alive not needed for a demo |
| Scope creep | High | MVP freeze tag before any phase 2 work |

## 13. Release plan
| Release | Contents |
|---|---|
| **v1.0-mvp** | Everything marked Must, plus US12 |
| **v1.x (phase 2)** | Prioritized backlog in WORKFLOW.md step 11 |
| **Final submission** | v1.x + report + viva prep |

## 14. Open questions
| # | Question | Owner | Status |
|---|---|---|---|
| OQ1 | Which ER diagram tool to use? | Team | Closed 2026-11-09: draw.io |
| OQ2 | Streamlit or Flask for the UI? (Default: Streamlit) | Team | Open (decide in step 0) |
| OQ3 | Deadline and the split of work between the two members | Team | Open |
| OQ4 | Find a yoga teacher or physio reviewer for phase 2 | Team | Open |

## 15. Change history
| Date | Version | Change |
|---|---|---|
| 2026-10-19 | 1.0 | First PRD, built from research (2026-10-15) and architecture draft v0.3 (2026-10-18). |
| 2026-11-09 | 1.0 | OQ1 closed. No change to requirements. |
| 2026-11-09 | 1.1 | FR5 tightened (ADR-012). The *safe* label is renamed *no known rule* (ADR-013). |
