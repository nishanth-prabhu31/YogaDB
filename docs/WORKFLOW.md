# Workflow: Step 0 to Final Submission

| | |
|---|---|
| **Last updated** | 2026-10-19 |
| **Approach** | Build a working **MVP** first, freeze it (`v1.0-mvp`), then add innovations one at a time. Each step ends with a **gate**: the team reviews the step before the next one starts. |
| **Live progress** | See the Status Board in [../AGENTS.md](../AGENTS.md#5-status-board). This file holds the *plan*, and AGENTS.md holds *where we are*. |

Every step is laid out the same way:
- **Goal:** why the step exists
- **Inputs:** what you need before starting
- **Tasks:** what to do
- **Deliverables:** the files it produces
- **Done when:** checks that must all pass
- **Learn:** the DBMS concepts this step teaches (for the viva)

---

## Phase A: Design

### Step 0: Project setup
- **Goal:** a clean repo both members can run.
- **Inputs:** a GitHub repo, Python 3.12+, a Neon or Supabase account, a MongoDB Atlas account.
- **Tasks:**
  1. Create the folder layout from [ARCHITECTURE.md §5.3](ARCHITECTURE.md#53-repository-layout).
  2. Add `.gitignore` (Python, `.env`, `__pycache__`, `.venv`, `data/raw/*` large files if needed).
  3. Add `.env.example` listing `PG_DSN`, `MONGO_URI`, `MONGO_DB`, `APP_USER_ID`.
  4. Add `requirements.txt`: `psycopg[binary]`, `pymongo`, `python-dotenv`, `streamlit`, `pytest`.
  5. Create the free cloud databases. Put the real credentials only in local `.env` files.
  6. Write a tiny `scripts/ping.py` that connects to both databases and prints their versions.
  7. Decide OQ2 (Streamlit or Flask) and record it as an ADR.
- **Deliverables:** folder skeleton, `.gitignore`, `.env.example`, `requirements.txt`, `scripts/ping.py`.
- **Done when:** both members can run `python scripts/ping.py` and see both database versions.
- **Learn:** client–server databases, connection strings, why secrets stay out of git.

### Step 1: Lock the data requirement specification
- **Goal:** a final, agreed list of entities, attributes, data types and constraints. This is a graded rubric item.
- **Inputs:** [ARCHITECTURE.md §6](ARCHITECTURE.md#6-data-architecture), [PRD.md](PRD.md).
- **Tasks:**
  1. For each of the 16 tables, write: purpose, every attribute, data type, NULL/NOT NULL, key, constraint, and an example value.
  2. List all ENUM values (practice_type, cat_group, body action, cond_type, severity, evidence, licence).
  3. Write down the business rules in plain words (FR5 inheritance, FR6 active conditions).
  4. Write the 2 Mongo document shapes with field types.
  5. Write sample data: 3 practices (one base, one variation, one pranayama) shown across every table.
- **Deliverables:** `docs/data-spec.md`.
- **Done when:** the team signs off, and every attribute has a type and a constraint decision.
- **Learn:** entity, attribute, domain, key, constraint, cardinality.

### Step 2: ER diagram *(separate step)*
- **Goal:** a clear, readable ER diagram for the report and the viva.
- **Inputs:** `docs/data-spec.md`.
- **Tasks:**
  1. Settle OQ1: which tool? (Discuss first. Candidates: dbdiagram.io, draw.io / diagrams.net, ERDPlus for Chen notation, Mermaid `erDiagram` kept in the repo.)
  2. Draw the **conceptual ER** (Chen notation: entities, relationships, attributes, cardinality, participation). This is the textbook diagram the course expects.
  3. Draw the **logical ER** (crow's foot: tables, PK/FK). This becomes the schema.
  4. Check readability: core tables in the middle, lookups around the edge, no crossing lines where avoidable.
- **Deliverables:** `docs/er/` (source file + PNG/SVG export for both diagrams).
- **Done when:** every table and relationship in the data spec appears exactly once, and the team can explain every line.
- **Learn:** Chen vs crow's foot notation, weak vs strong entities, recursive relationships, M:N becoming junction tables.

### Step 3: Relational mapping and 3NF normalization
- **Goal:** prove the schema is in 3NF. This is a graded rubric item.
- **Inputs:** the ER diagram.
- **Tasks:**
  1. Map ER → relations (show the rules: entity → table, M:N → junction, recursive 1:N → self-FK).
  2. Start from a deliberately messy "one big table" (a pose with repeated benefits, conditions and category info) and normalize it step by step:
     - **UNF → 1NF:** remove repeating groups.
     - **1NF → 2NF:** remove partial dependencies, which only matter when a key has two columns.
     - **2NF → 3NF:** remove transitive dependencies.
  3. List the functional dependencies (FDs) for every final table and show each one satisfies 3NF.
  4. Note any deliberate design choices (for example, name columns per language, ADR-006).
- **Deliverables:** `docs/normalization.md`.
- **Done when:** every table has its FDs listed, and the final tables match the ER diagram.
- **Learn:** functional dependency, candidate key, partial and transitive dependency, 1NF/2NF/3NF/BCNF.

**🚦 Gate A:** the design is frozen. Changes after this point need a logged ADR.

---

## Phase B: Build the database

### Step 4: Physical schema (PostgreSQL)
- **Goal:** turn the design into runnable SQL.
- **Tasks:**
  1. `db/postgres/01_types.sql`: ENUM types.
  2. `02_tables.sql`: all 16 tables with PK, FK, UNIQUE, CHECK and NOT NULL.
  3. `03_indexes.sql`: indexes on every FK and on `alias.norm_key`, plus the partial unique index on the primary image.
  4. `04_views.sql`: `v_practice_lineage`, `v_effective_contra`, `v_user_active_condition`, `v_user_practice_safety`, `v_practice_count`.
  5. `05_roles.sql`: the `yoga_app` least-privilege role.
  6. A `db/postgres/README.md` that explains the run order.
- **Done when:** the scripts run cleanly on an empty database, run a second time without errors where designed to, and these hand tests pass: a duplicate favourite is rejected, a contra without a source is rejected, a difficulty of 4 is rejected.
- **Learn:** DDL, data types, constraints, indexes, views, recursive CTEs, GRANT.

### Step 5: Document schema (MongoDB)
- **Goal:** set up the collections with validation.
- **Tasks:** create `practice_content` and `practice_log` with `$jsonSchema` validators and indexes (`db/mongo/setup.py`).
- **Done when:** an invalid document (missing steps, felt = 9) is rejected and a valid one is accepted.
- **Learn:** schema-on-read vs schema-on-write, document validation, compound indexes.

### Step 6: Seed data pipeline, to about 500 entries
- **Goal:** real, licence-safe, sourced data.
- **Inputs:** [RESEARCH.md §9](RESEARCH.md) datasets.
- **Tasks:**
  1. **Sources first:** add rows to `source` for every dataset and reference used.
  2. **Lookups:** categories (position / action / effect / style), body parts, benefits, conditions (start with about 20 common ones: glaucoma, hypertension, pregnancy, knee injury, lower-back pain, neck injury, heart disease, vertigo, osteoporosis, recent surgery…).
  3. **Base asanas:** import alexcumplido/yoga-api (48) and the Wikipedia List of asanas (about 90). Merge duplicates using `norm_key`. Write descriptions in our own words.
  4. **Variations:** add child rows with `parent_id` (for example: supported, revolved, bound, half, wall-assisted).
  5. **Other types:** pranayama, mudra, bandha, kriya, meditation from the AYUSH CYP and other sources.
  6. **Kannada names:** transliterate, and spot-check a sample by hand.
  7. **Contraindications:** category rules first (biggest coverage), then pose-level rules, each with a source.
  8. **Images:** Wikimedia Commons / yoga-api CC0, with author and licence recorded.
  9. **Mongo content:** English steps for base asanas.
  10. Run `check_consistency.py` and `SELECT * FROM v_practice_count`.
- **Deliverables:** `scripts/import_*.py`, `scripts/seed.py`, `data/curated/*`, `scripts/check_consistency.py`.
- **Done when:** the grand total is ≥ 450, there are 0 orphaned Mongo references, every contra row has a source, and every image has a licence.
- **Learn:** ETL (extract, transform, load), dedupe, load order under FK constraints, data provenance.

**🚦 Gate B:** the database is complete and verified with plain SQL queries.

---

## Phase C: Build the app

### Step 7: Data-access layer
- **Goal:** all queries live in `repos/`, are parameterized and are tested.
- **Tasks:** `core/db.py` (connection pools from `.env`), `core/validate.py` (input validation + Mongo `$` guard), one repo file per table group or collection.
- **Done when:** a `pytest` test for every repo function passes against a test database, and grep finds no f-string or `+`-concatenated SQL.
- **Learn:** embedded/dynamic SQL, parameter binding, injection prevention.

### Step 8: Services and UI (MVP features)
Build in this order. Each one is a small, demo-able increment:
1. Browse + filter (US1, US3)
2. Search (US2)
3. Detail page (US4): the SQL + Mongo join
4. My health conditions (US5)
5. Safe-for-me (US6) + disclaimer (US12)
6. Favourites (US7)
7. Practice log + stats (US8, US11)
8. Admin CRUD (US9)
9. Count page (US10)
- **Done when:** every Must story's acceptance criteria in the PRD pass when checked by hand in the running app.
- **Learn:** layered architecture, separation of concerns.

### Step 9: Testing and verification
- **Tasks:**
  1. **Safety test matrix:** at least 10 cases covering own rule, inherited rule, category rule, override that relaxes a rule, highest-severity tie-break, inactive (expired) condition, no rule.
  2. Constraint tests (each one should fail on purpose).
  3. Injection tests (SQL `' OR 1=1 --` and Mongo `{"$ne": null}`).
  4. Consistency script is clean.
- **Done when:** all tests pass in `pytest`, and the results are recorded in `docs/test-report.md`.

### Step 10: Deploy and freeze the MVP
- **Tasks:** deploy to Render or Streamlit Community Cloud, set secrets, smoke-test every page, update the README with the live link and run instructions, then tag **`v1.0-mvp`**.
- **Done when:** the live URL works on both a laptop and a phone, and the tag exists.

**🚦 Gate C:** the MVP is frozen. A working version is saved and safe.

---

## Phase D: Innovations (phase 2)

### Step 11: Add innovations one at a time
Rule: **one feature per branch**. Each feature must keep all MVP tests green, gets its own ADR, and is merged only when done.

Suggested priority (highest rubric and resume value per hour first):
| # | Feature | Rubric / value |
|---|---|---|
| 1 | Fuzzy multilingual search (`pg_trgm` + `unaccent`) | Recent trend |
| 2 | `audit_log` filled by PL/pgSQL triggers | Security + advanced SQL |
| 3 | Practice relations + recursive "what prepares for X" | Advanced SQL |
| 4 | Rule-based sequence builder (+ `sequences` collection) | Innovation |
| 5 | Multi-user: hashed passwords, roles, row-level security | Security |
| 6 | Kannada steps (`lang: "kn"` documents) | Societal |
| 7 | NLP condition search ("my knee hurts") | Recent trend |
| 8 | Therapeutic use ("helps") table | Feature depth |
| 9 | Review workflow (when a reviewer is found) | Safety |

---

## Phase E: Submission

### Step 12: Report and viva prep
- **Tasks:**
  1. Report: abstract, problem, literature/existing systems, requirements (PRD), ER, normalization, schema, NoSQL design, SQL+NoSQL integration, security, recent trends, societal impact, testing, screenshots, conclusion, references (IEEE).
  2. Viva sheet: 20 likely questions with short answers (3NF, why Mongo, why not JSONB, the inheritance view, injection, etc.).
  3. Demo script: a 5-minute walkthrough.
- **Done when:** the report is complete, every rubric row in PRD §10 points to a page, and both members can run the demo.

---

## Step summary
| Step | Name | Phase |
|---|---|---|
| 0 | Project setup | A Design |
| 1 | Data requirement spec | A |
| 2 | ER diagram | A |
| 3 | Mapping + 3NF | A |
| 4 | Postgres schema | B Database |
| 5 | Mongo schema | B |
| 6 | Seed to ~500 | B |
| 7 | Data-access layer | C App |
| 8 | Services + UI | C |
| 9 | Testing | C |
| 10 | Deploy + freeze `v1.0-mvp` | C |
| 11 | Innovations | D |
| 12 | Report + viva | E |
