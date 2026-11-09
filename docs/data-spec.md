# Data Requirement Specification: Yoga Knowledge Base

| | |
|---|---|
| **Last updated** | 2026-11-09 |
| **Status** | Draft for team sign-off at Gate A |
| **Related** | [PRD.md](PRD.md) · [ARCHITECTURE.md §6](ARCHITECTURE.md#6-data-architecture) · [normalization.md](normalization.md) · [er/](er/) |

**Conventions:** `snake_case` names, singular tables, surrogate primary key `id`, foreign keys named `<table>_id`. All foreign keys are `ON DELETE RESTRICT`. `timestamptz` values are UTC. Text is UTF-8.

---

## 1. ENUM types and allowed values

| Type | Values | Notes |
|---|---|---|
| `practice_type` | `asana`, `pranayama`, `mudra`, `bandha`, `kriya`, `meditation` | "Variation" is **not** a type. A practice is a variation when `parent_id IS NOT NULL`, so a kriya sub-type keeps the type `kriya`. |
| `cat_group` | `position`, `action`, `effect`, `style` | Groups categories. |
| `body_action` | `stretch`, `strengthen`, `stabilise` | How a practice works a body part. |
| `cond_type` | `chronic`, `injury`, `temporary` | |
| `severity` | `caution`, `modify`, `avoid` | **Declared lowest to highest.** Postgres compares ENUMs in declaration order, so `MAX(severity)` returns the worst one. |
| `evidence` | `study`, `guideline`, `expert`, `traditional` | Strength of one specific claim. |
| `source_type` | `journal`, `government`, `book`, `website`, `dataset` | Kind of publication. Different from `evidence`: one book can hold both an expert claim and a traditional claim. |
| `media_licence` | `CC0`, `CC-BY`, `CC-BY-SA` | The only licences allowed for images. |

## 2. Relational data (PostgreSQL, 16 tables)

### Core

**`practice`**: one yoga practice of any type, including variations.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 42 |
| slug | `text` | NO | UNIQUE | `adho-mukha-svanasana` |
| name_en | `text` | NO | | Downward-Facing Dog |
| name_sa | `text` | YES | IAST spelling; empty for modern poses with no Sanskrit name | Adho Mukha Śvānāsana |
| name_kn | `text` | YES | Kannada script; target ≥ 90 % filled | ಅಧೋಮುಖ ಶ್ವಾನಾಸನ |
| type | `practice_type` | NO | | `asana` |
| parent_id | `integer` | YES | FK → practice(id); CHECK `parent_id <> id` | NULL |
| difficulty | `smallint` | NO | CHECK 1–3 | 2 |
| description | `text` | NO | Written in our own words | Inverted V shape… |
| created_at | `timestamptz` | NO | DEFAULT `now()` | |
| updated_at | `timestamptz` | NO | DEFAULT `now()` | |

**`alias`**: alternate spellings used for search. This is the multivalued attribute from the ER diagram.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 7 |
| practice_id | `integer` | NO | FK → practice(id) | 42 |
| alias_text | `text` | NO | | Adho Mukha Shvanasana |
| norm_key | `text` | NO | `GENERATED ALWAYS AS (...) STORED` from `alias_text`; UNIQUE (practice_id, norm_key); indexed | `adhomukhashvanasana` |

`norm_key` is lower-case with IAST diacritics, spaces and hyphens removed, using `lower()` and `translate()`. The database calculates it, so it can never disagree with `alias_text`.

### Describing practices

**`category`**: a group such as *inversion* or *standing*.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 3 |
| name | `text` | NO | UNIQUE | inversion |
| cat_group | `cat_group` | NO | | `position` |

**`practice_category`**: links practices and categories (M:N).

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| practice_id | `integer` | NO | PK part; FK → practice(id) | 42 |
| category_id | `integer` | NO | PK part; FK → category(id) | 3 |

**`body_part`**

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 5 |
| name | `text` | NO | UNIQUE | hamstrings |

**`practice_body_part`**: links practices and body parts (M:N) and says how the part is worked.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| practice_id | `integer` | NO | PK part; FK → practice(id) | 42 |
| body_part_id | `integer` | NO | PK part; FK → body_part(id) | 5 |
| action | `body_action` | NO | | `stretch` |

**`benefit`**

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 2 |
| name | `text` | NO | UNIQUE | improves flexibility |

**`practice_benefit`**: links practices and benefits (M:N).

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| practice_id | `integer` | NO | PK part; FK → practice(id) | 42 |
| benefit_id | `integer` | NO | PK part; FK → benefit(id) | 2 |

### Safety

**`condition`**

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 1 |
| name | `text` | NO | UNIQUE | glaucoma |
| cond_type | `cond_type` | NO | | `chronic` |

**`source`**: where a safety claim comes from.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 1 |
| title | `text` | NO | | Intraocular Pressure Rise in Subjects with and without Glaucoma during Four Common Yoga Positions |
| author | `text` | YES | Empty for unsigned government documents | Jasien, Jonas, de Moraes, Ritch |
| year | `smallint` | YES | CHECK 1000–2100 (classical texts can be undated) | 2015 |
| url | `text` | YES | Empty for print-only books | https://doi.org/10.1371/journal.pone.0144505 |
| src_type | `source_type` | NO | | `journal` |
| licence | `text` | YES | Free text, describes the source and not an image | CC BY |

**`contra`**: a contraindication for one practice and one condition.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| practice_id | `integer` | NO | PK part; FK → practice(id) | 42 |
| condition_id | `integer` | NO | PK part; FK → condition(id) | 1 |
| severity | `severity` | NO | | `caution` |
| note | `text` | NO | Shown to the user with the label | Eye pressure rose in this pose in the cited study. |
| evidence | `evidence` | NO | | `study` |
| source_id | `integer` | **NO** | FK → source(id). H3 / ADR-008 | 1 |

**`category_contra`**: a contraindication for a whole category and one condition.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| category_id | `integer` | NO | PK part; FK → category(id) | 3 |
| condition_id | `integer` | NO | PK part; FK → condition(id) | 1 |
| severity | `severity` | NO | | `caution` |
| note | `text` | NO | | Head-down poses raised eye pressure in the cited study. |
| evidence | `evidence` | NO | Required by FR4 | `study` |
| source_id | `integer` | **NO** | FK → source(id). H3 / ADR-008 | 1 |

### Media

**`media`**: one licence-checked image of a practice. Stored as a URL, never as a BLOB.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 11 |
| practice_id | `integer` | NO | FK → practice(id) | 42 |
| url | `text` | NO | UNIQUE | https://upload.wikimedia.org/… |
| alt_text | `text` | NO | | A person in Downward-Facing Dog |
| author | `text` | **NO** | | (from the file page) |
| licence | `media_licence` | **NO** | | `CC-BY-SA` |
| source_url | `text` | NO | Page where the licence can be checked | https://commons.wikimedia.org/… |
| is_primary | `boolean` | NO | DEFAULT false; partial UNIQUE (practice_id) WHERE is_primary | true |

### User (one row in the MVP)

**`app_user`**

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| id | `integer` identity | NO | PK | 1 |
| name | `text` | NO | | Demo User |
| created_at | `timestamptz` | NO | DEFAULT `now()` | |

**`user_condition`**: the health conditions a user has.

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| user_id | `integer` | NO | PK part; FK → app_user(id) | 1 |
| condition_id | `integer` | NO | PK part; FK → condition(id) | 1 |
| start_date | `date` | **NO** | DEFAULT `CURRENT_DATE` | 2026-01-10 |
| end_date | `date` | YES | CHECK `end_date >= start_date` | NULL |

`start_date` is never NULL on purpose. US5 makes it optional *in the form*, and the database then fills in today. If it could be NULL, the FR6 test `start_date <= today` would not be true, and the condition would silently stop counting.

**`favourite`**

| Attribute | Type | Null | Key / constraint | Example |
|---|---|---|---|---|
| user_id | `integer` | NO | PK part; FK → app_user(id) | 1 |
| practice_id | `integer` | NO | PK part; FK → practice(id) | 42 |
| added_at | `timestamptz` | NO | DEFAULT `now()` | |

## 3. Business rules

| ID | Rule | Enforced by |
|---|---|---|
| BR1 | **Effective contraindication (FR5).** For practice P and condition C: (1) if P has its own rule, use it; otherwise (2) take the highest severity among the nearest ancestor's own rule and all category rules for the categories of P and of its ancestors; if there are none, (3) *no known rule*. (ADR-012) | View `v_effective_contra` |
| BR2 | **Active condition (FR6).** `start_date <= today` and (`end_date` is NULL or `end_date >= today`). | View `v_user_active_condition` |
| BR3 | A user's label for a practice is the worst severity across their active conditions, or *no known rule* if there is none. That label does not mean safe (ADR-013). | View `v_user_practice_safety` |
| BR4 | Every contraindication has a source. | NOT NULL + FK |
| BR5 | Every image has an author and an allowed licence. | NOT NULL + ENUM |
| BR6 | A practice has at most one primary image. | Partial unique index |
| BR7 | Every practice belongs to at least one category, so category safety rules can reach it. | A foreign key cannot express this. Checked by a query that must return 0 rows (Step 6, `check_consistency.py`). |
| BR8 | The variation tree has no loops. | CHECK stops a practice being its own parent. Longer loops: the check query in Step 6 plus the depth limit of 10 in `v_practice_lineage`. |
| BR9 | Practices are never hard-deleted, because MongoDB refers to them. | `ON DELETE RESTRICT`, H8 |
| BR10 | Every `practice_id` in MongoDB exists in PostgreSQL. | `check_consistency.py` |

**Known limit:** `user_condition` has one row per user and condition, so one condition cannot be stored twice for two separate periods. The MVP accepts this.

## 4. Document data (MongoDB, 2 collections)

**`practice_content`**: one document per practice per language. Unique index on (practice_id, lang).

| Field | Type | Required | Rule |
|---|---|---|---|
| practice_id | int | yes | Matches `practice.id` |
| lang | string | yes | Enum: `en`, `kn` |
| steps | array of string | yes | At least 1 item |
| tips | array of string | no | |
| mistakes | array of string | no | |
| breathing | string | no | |
| updated_at | date | yes | UTC |

**`practice_log`**: one document per session. Index on (user_id, date descending).

| Field | Type | Required | Rule |
|---|---|---|---|
| user_id | int | yes | Matches `app_user.id` |
| date | date | yes | UTC |
| items | array of object | yes | At least 1 item |
| items[].practice_id | int | yes | Matches `practice.id` |
| items[].hold_s | int | no | ≥ 0 |
| items[].felt | int | yes | 1–5 |
| items[].pain | bool | yes | |
| notes | string | no | |

## 5. Sample data across every table

> **This shows the data format only and is not seed data.** The study is real and was read on 2026-11-09. It measured eye pressure in four head-down poses. It does **not** assign a severity: `caution` here is an example, and the real value needs a team decision in Step 6. The variation row and Kannada names should also be checked at that point.

**`source`**

| id | title | author | year | url | src_type | licence |
|---|---|---|---|---|---|---|
| 1 | Intraocular Pressure Rise in Subjects with and without Glaucoma during Four Common Yoga Positions | Jasien, Jonas, de Moraes, Ritch | 2015 | https://doi.org/10.1371/journal.pone.0144505 | journal | CC BY |

**`practice`** (base, variation, pranayama)

| id | slug | name_en | name_sa | name_kn | type | parent_id | difficulty |
|---|---|---|---|---|---|---|---|
| 42 | adho-mukha-svanasana | Downward-Facing Dog | Adho Mukha Śvānāsana | ಅಧೋಮುಖ ಶ್ವಾನಾಸನ | asana | NULL | 2 |
| 43 | eka-pada-adho-mukha-svanasana | Three-Legged Downward Dog | Eka Pada Adho Mukha Śvānāsana | ಏಕಪಾದ ಅಧೋಮುಖ ಶ್ವಾನಾಸನ | asana | 42 | 2 |
| 90 | nadi-shodhana | Alternate Nostril Breathing | Nāḍī Śodhana | ನಾಡೀ ಶೋಧನ | pranayama | NULL | 1 |

| Table | Sample rows |
|---|---|
| `alias` | (7, 42, "Adho Mukha Shvanasana", `adhomukhashvanasana`) · (8, 90, "Anulom Vilom", `anulomvilom`) |
| `category` | (3, inversion, position) · (4, breathwork, style) |
| `practice_category` | (42, 3) · (43, 3) · (90, 4) |
| `body_part` | (5, hamstrings) |
| `practice_body_part` | (42, 5, stretch) · (43, 5, stretch) |
| `benefit` | (2, improves flexibility) |
| `practice_benefit` | (42, 2) |
| `condition` | (1, glaucoma, chronic) |
| `contra` | (42, 1, caution, "Eye pressure rose in this pose in the cited study.", study, 1) |
| `category_contra` | (3, 1, caution, "Head-down poses raised eye pressure in the cited study.", study, 1) |
| `media` | (11, 42, *image URL*, *alt text*, *author*, CC-BY-SA, *file page*, true) |
| `app_user` | (1, Demo User, *now*) |
| `user_condition` | (1, 1, 2026-01-10, NULL) |
| `favourite` | (1, 90, *now*) |

**Applying BR1 for this user (glaucoma):** practice 42 has its own rule, so it's *caution*. Practice 43 has no rule of its own, so it takes the worst of its parent's *caution* and the *caution* on the inversion category, which is *caution*. Practice 90 has no rule and neither does its category, so its label is *no known rule*.

**`practice_content`**
```json
{ "practice_id": 90, "lang": "en",
  "steps": ["Sit tall.", "Close the right nostril and breathe in through the left."],
  "tips": ["Keep the breath slow"],
  "updated_at": "2026-11-09T10:00:00Z" }
```

**`practice_log`**
```json
{ "user_id": 1, "date": "2026-11-09T06:30:00Z",
  "items": [ { "practice_id": 90, "hold_s": 300, "felt": 4, "pain": false } ],
  "notes": "Felt steady today" }
```
