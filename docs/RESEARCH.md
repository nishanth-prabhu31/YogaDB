# Research: Yoga Knowledge Base
_2026-10-15 · background research for the project. The decisions made since are in [ARCHITECTURE.md](ARCHITECTURE.md) (ADRs) and [PRD.md](PRD.md). Where this file and those disagree, those win._

## 0. Base idea (as inferred)
A yoga knowledge base — asanas, categories, benefits, contraindications, variations, pranayama — on an SQL + NoSQL backend, with a light frontend and a few basic features.

## 1. Fact-check of the earlier plan
| Claim | Verdict | Notes |
|---|---|---|
| "84 asanas" is symbolic | ✅ True | The Hatha Ratnavali (17th c.) is credited as the first text to actually name 84. |
| Light on Yoga ≈ 200 asanas | ✅ True | It has 200+ postures and ~600 photos. |
| Dharma Mittra chart = 908 | ✅ True (date is disputed) | Sources give both 1984 and 1974. The book edition has 608 poses. The chart is copyrighted, so we can only use it to cross-check names. |
| Mr. Yoga's 2,100 | ✅ It exists, but it's an outlier | Don't use it as a source. |
| A Kaggle dataset with benefits, contraindications and difficulty | ⚠️ Couldn't verify | Every Kaggle yoga dataset I found was image-only (5, 16 or 38 poses). Treat the claim as unconfirmed until we have a link. |
| 500–700 entries is "achievable" | ❌ Risky | Wikipedia's structured table has only ~90 core rows. Hand-writing *safe* contraindication data for 500+ poses is the real bottleneck. It's also a safety risk if we get it wrong. |
| Choose SQL vs NoSQL per data type | ✅ Correct principle | We still need a clear answer to "why not just Postgres JSONB?" (see §4). |

## 2. What already exists (the gap we fill)
- **Yoga Journal Pose Library:** about 50 poses, browsable by type, anatomy and benefit. It's **paywalled**.
- **Pocket Yoga:** 500+ illustrated poses in a searchable dictionary. Benefits are written as prose, with no structured safety filtering.
- **Down Dog:** generates sequences and lets you like or dislike poses. It's closed and has no browsable knowledge base.
- **Namaste Yoga (Ministry of AYUSH, 2021):** finds trainers, centres and events, has asana guides and tracks activity. There's no condition-aware filtering.
- **Open APIs (LunaticPrakash/yoga-api, Stuwert/yoga-builder):** small, unlicensed and still in development. They're useful only as references for field design.
- **Yoga Ontology (BioPortal, 2020):** exists but has been reported as unavailable. I found no peer-reviewed paper that models asana contraindications as structured data.

**The gap:** no free tool lets a user say *"I have glaucoma and a bad knee, show me what's safe"* and get an answer backed by data, with sources attached.

## 3. Core improvements (ranked)
1. **Change the pitch from "biggest pose count" to "safety-aware, queryable yoga knowledge base."** Quality and safety become the selling point. Raw count isn't one, because Pocket Yoga already has 500+.
2. **Count entries honestly by modelling variations.** Aim for about 150–250 deeply modelled base poses. Store variations as child rows (a self-referencing FK), which gives 400+ total entries without padding.
3. **Make contraindications first-class data:**
   - Link Pose to Condition through a mapping table that has `severity` (avoid / modify / caution).
   - Each mapping also gets a `modification_note`, a `source_id` and an `evidence_level` (study / clinical guideline / expert / traditional).
   - Real evidence to cite: the Cramer et al. 2013 systematic review (PLoS One) and the 2015 PLOS One IOP study. The IOP study found inversions raise eye pressure in glaucoma patients, with downward dog going from about 17 to 28 mmHg.
4. **Model the relationships between poses** (`preparatory_for`, `counter_pose_of`, `variation_of`, `progression_of`). This is a strong DBMS showcase: recursive CTEs ("everything that leads up to Wheel pose"), self-joins and constraints.
5. **Make the sequence builder rule-based and run it from the DB.** Pick a peak pose. Pull its preparatory poses ordered by difficulty, add counter-poses, then drop anything contraindicated for the user. No existing source has a formal algorithm for this, so it counts as our innovation.
6. **Search that handles transliteration.** Sanskrit spellings vary (Svanasana / Shvanasana, Vrksasana / Vrikshasana). Fuzzy matching with Postgres `pg_trgm` + `unaccent`, plus an alias table, solves a real problem, and it's a good "recent trends" item. It's cheaper and more defensible than an ML feature.
7. **Use authoritative, licence-safe sources:**
   - Ministry of AYUSH Common Yoga Protocol (2022 edition, free, government-published): the Indian standard.
   - Wikipedia List of asanas (CC BY-SA, with attribution): gives type, effect on spine, source text and date.
   - Wikidata item Q466797, which can be queried with SPARQL.
   - Iyengar and Mittra for cross-checking names only. We copy none of their text or photos.
8. **Track where every fact came from.** A `sources` table, with each benefit and contraindication citing one. That's academically honest and rare in student projects.

## 4. SQL + NoSQL split (defensible version)
**SQL (PostgreSQL)** holds the canonical, relational, integrity-critical data:
- poses, aliases, categories, body parts and muscles
- conditions, the contraindication mapping, benefits, sources
- pose relationships
- users and roles

**NoSQL (MongoDB)** holds variable-shape, append-heavy, user-generated data:
- practice session logs (each session has different poses, holds and notes)
- saved sequences (nested, ordered documents)
- per-pose content in multiple languages (English / Kannada / Hindi instruction blobs)
- user preferences and activity feed

**Link between them:** the app layer references `pose_id` across the two stores. There's no duplicate source of truth.

**The "why not JSONB?" answer:** JSONB is a valid alternative, and we considered it. We chose Mongo because:
- logs and sequences are document-native, append-heavy and schema-fluid
- the aggregation pipeline handles analytics (streaks, most-practised poses)
- it lets us show the polyglot pattern deliberately

The trade-off we accept is keeping the two stores in sync and running two systems.

## 5. Feature set (light but complete)
- **Must:**
  - search and filter (category, difficulty, body part, benefit)
  - pose detail page (relations, sources)
  - a "safe for me" filter based on the user's conditions
  - favourites
  - practice log
  - admin CRUD
- **Should:**
  - sequence builder
  - stats dashboard (aggregations from both DBs)
  - fuzzy Sanskrit search
- **Could:** multilingual toggle, streaks.
- **Drop:** camera pose-correction (MediaPipe). It's scope creep and needs a separate computer-vision project.

## 6. Report sections
- **Societal:**
  - preventive health
  - fits AYUSH and International Day of Yoga goals
  - accessibility (regional languages, safety filters for older users and people with chronic conditions)
  - a clear medical disclaimer
- **Security:**
  - role-based access (user / reviewer / admin)
  - parameterised SQL
  - **NoSQL operator-injection guard** (blocking `$ne`/`$gt` in input), which most students miss
  - hashed passwords
  - health conditions treated as sensitive data: minimal, user-controlled, deletable
- **Innovation:** evidence-tagged contraindications, the rule-based sequence engine, and transliteration-aware search.

## 7. Stack (decided; all free tier)
- **Relational:** PostgreSQL ✅ confirmed. Host on Supabase or Neon.
  - `pg_trgm`: fuzzy Sanskrit search
  - `unaccent`: diacritic-free matching keys
  - partial unique indexes, CHECK constraints, ENUM types
  - recursive CTEs, triggers, PL/pgSQL procedures
  - `GRANT` / row-level security for health data
- **NoSQL:** MongoDB Atlas M0.
- **App:** Python. The frontend can be pure Python (Streamlit or Flask templates); HTML/JS is optional.
- **Deployment (for the resume):** Render for the app. Not Vercel, which fits Flask + DB poorly.

## 8. Answers so far
1. **Course:** DBMS EL, a batch of two.
   - **Rubric:** full data requirement spec; integrated SQL + NoSQL; 3NF; a security section; report; recent trends + societal concern.
   - **Allowed:** "any database management tool", plus embedded/dynamic SQL.
2. **Team size / deadline:** parked for now.
3. **Languages:** English, with Kannada + Sanskrit where possible.
4. **Deployment:** yes, a simple one, since it doubles as a resume project.
5. **About 500 entries:** expected by the prof. Some flexibility if the features are strong.

## 9. Starter datasets & images (verified 2026-10-15)
**Tier A: safe to use directly (with attribution)**
- **alexcumplido/yoga-api** (MIT code). 48 poses in categories. Fields: english/sanskrit/adapted names, translation, description, benefits, difficulty, SVG/PNG images. Red images are CC0; pink and green images are Flaticon and need attribution. Live at https://yoga-api-nzy4.onrender.com/v1/poses. This is the best seed.
- **Wikipedia List of asanas** (CC BY-SA 4.0). About 90 rows, with columns: type, effect on spine, suitable for meditation, earliest text and date, yin variant.
- **Wikimedia Commons Category:Asana**. About 159 per-pose subcategories under "Asanas by name" and about 270 files, including a 116-file Mr. Yoga set. Each file has its own licence, so check each one.
- **AYUSH Common Yoga Protocol 2022 PDF** (government). Use it for facts and protocol order. Write all text in our own words.

**Tier B: cross-check only (no licence or unclear licence)**
- **HF omergoshen/yoga_poses**. 160 poses with fields name, sanskrit_name, expertise_level, pose_type, **followup_poses** (useful for pose relationships). The photos come from pocketyoga.com, so don't reuse them.
- **LunaticPrakash/yoga-api** (no licence). Use it as a reference for field design only.

**Tier C: image-only, needed only if we add a computer-vision angle**
- Yoga-82: images scraped from the web, research use only.
- Roboflow sets: CC BY 4.0.
- Zenodo 7818789: 10 asanas, labelled correct vs incorrect form.
- Kaggle Yoga-16.

## Sources
- https://en.wikipedia.org/wiki/List_of_asanas
- https://en.wikipedia.org/wiki/Light_on_Yoga
- https://en.wikipedia.org/wiki/Dharma_Mittra
- https://www.yogajournal.com/poses/library/
- https://github.com/LunaticPrakash/yoga-api
- https://github.com/Stuwert/yoga-builder
- https://bioportal.bioontology.org/ontologies/YOGA
- https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3797727/ (Cramer 2013)
- https://www.aoa.org/news/clinical-eye-care/head-down-yoga-poses-increase-eye-pressure-in-glaucoma-patients
- https://doaj.org/article/57ffb954dd79459f8d16b69cfc30a350 (IJO 2023 meta-analysis)
- https://www.sanskrit.nic.in/uploads/CYP_2022/English.pdf (AYUSH CYP)
- https://www.gktoday.in/ayush-ministry-launches-namaste-yoga-mobile-app/
- https://www.wikidata.org/wiki/Q466797
- https://ceur-ws.org/Vol-1644/paper31.pdf (JSONB vs Mongo)
- https://www.acefitness.org/continuing-education/prosource/september-2016/6034/the-art-of-intelligently-sequencing-yoga-classes/
