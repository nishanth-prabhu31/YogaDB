# Relational Mapping and 3NF Normalization

| | |
|---|---|
| **Last updated** | 2026-11-09 |
| **Status** | Draft for team review at Gate A |
| **Inputs** | [er/er-conceptual.png](er/er-conceptual.png) · [data-spec.md](data-spec.md) |

## 1. ER to relational mapping

Underlined keys are written here as **PK**. In SQL the surrogate key is simply `id`. The ER diagrams label it `practice_id`, `category_id` and so on, so each attribute's name is clear on the page.

| Mapping rule | Applied to | Result |
|---|---|---|
| Strong entity → relation | PRACTICE, CATEGORY, BODY_PART, BENEFIT, CONDITION, SOURCE, MEDIA, USER | 8 relations (`USER` becomes `app_user`, because `user` is a reserved word) |
| Weak entity → relation with owner's key | None. Every entity has its own key. | 0 |
| Binary 1:N → FK on the N side | PRACTICE *illustrated by* MEDIA | `media.practice_id` |
| Recursive 1:N → self-referencing FK | *variation of* | `practice.parent_id`, NULL for a base practice |
| Multivalued attribute → new relation | PRACTICE.alias | `alias` |
| Binary M:N → junction relation, PK is both FKs | *belongs to*, *works on*, *gives*, *has*, *favourites* | `practice_category`, `practice_body_part`, `practice_benefit`, `user_condition`, `favourite` |
| Ternary with a "1" side → PK is the two "many" keys, the "1" side is a plain FK | *contra*, *cat rule* (SOURCE is the 1 side) | `contra`, `category_contra`, with `source_id` outside the PK |

**Total: 8 + 1 + 5 + 2 = 16 relations**, which matches [data-spec.md](data-spec.md).

**Relational schema** (`*` marks a foreign key):
```
practice(id PK, slug, name_en, name_sa, name_kn, type, parent_id*, difficulty, description, created_at, updated_at)
alias(id PK, practice_id*, alias_text, norm_key)
category(id PK, name, cat_group)
practice_category(practice_id* PK, category_id* PK)
body_part(id PK, name)
practice_body_part(practice_id* PK, body_part_id* PK, action)
benefit(id PK, name)
practice_benefit(practice_id* PK, benefit_id* PK)
condition(id PK, name, cond_type)
source(id PK, title, author, year, url, src_type, licence)
contra(practice_id* PK, condition_id* PK, severity, note, evidence, source_id*)
category_contra(category_id* PK, condition_id* PK, severity, note, evidence, source_id*)
media(id PK, practice_id*, url, alt_text, author, licence, source_url, is_primary)
app_user(id PK, name, created_at)
user_condition(user_id* PK, condition_id* PK, start_date, end_date)
favourite(user_id* PK, practice_id* PK, added_at)
```

## 2. Normalization walkthrough

### UNF: one messy sheet
Imagine one row per pose, as a spreadsheet would hold it. Values in `{ }` repeat inside one cell.

| pose_slug | name_en | parent_slug | parent_name | difficulty | benefits | categories (name, group) | rules (condition, cond_type, severity, source_title, source_year) |
|---|---|---|---|---|---|---|---|
| eka-pada-adho-mukha-svanasana | Three-Legged Dog | adho-mukha-svanasana | Downward-Facing Dog | 2 | {flexibility, strength} | {(inversion, position), (standing, position)} | {(glaucoma, chronic, caution, IOP Rise…, 2015), …} |

**Problems:** you can't query or constrain a value inside a list. Changing a source's year means editing every cell that mentions it.

### UNF → 1NF: remove repeating groups
Every cell must hold one value. Each repeating group moves to its own relation, along with the key of the row it came from.

```
P1(pose_slug PK, name_en, parent_slug, parent_name, difficulty)
P_BENEFIT(pose_slug PK, benefit PK)
P_CATEGORY(pose_slug PK, category PK, cat_group)
P_RULE(pose_slug PK, condition PK, cond_type, severity, source_title, source_year, source_url)
```

*Why not one flat table with every combination?* A pose with 3 benefits, 2 categories and 4 rules would need 3 × 2 × 4 = 24 rows, even though the three lists have nothing to do with each other.

### 1NF → 2NF: remove partial dependencies
A non-key column must depend on the **whole** key. This only matters for composite keys.

| Relation | Partial dependency | Fix |
|---|---|---|
| P_CATEGORY | `category → cat_group` | Move to CATEGORY(category PK, cat_group) |
| P_RULE | `condition → cond_type` | Move to CONDITION(condition PK, cond_type) |

P1 and P_BENEFIT have no non-key column that depends on part of a composite key, so they are already in 2NF.

### 2NF → 3NF: remove transitive dependencies
A non-key column must not depend on another non-key column.

| Relation | Transitive dependency | Fix |
|---|---|---|
| P1 | `pose_slug → parent_slug → parent_name` | Remove `parent_name`. It's already stored on the parent's own row, reached through `parent_id`. |
| P_RULE | `(pose, condition) → source_title → source_year, source_url` | Move to SOURCE and keep only `source_id` in the rule. |

Adding surrogate keys, the remaining lookups (`benefit`, `body_part`), and the category-level rules, media and user data gives the 16 relations in section 1.

## 3. Functional dependencies of the final schema

A relation is in **3NF** if, for every non-trivial FD X → A, either X is a superkey or A is **prime** (part of some candidate key). It is in **BCNF** if X is always a superkey.

| Relation | Candidate keys | Non-trivial FDs | Normal form |
|---|---|---|---|
| practice | {id}, {slug} | id → all; slug → all | BCNF |
| alias | {id}, {practice_id, norm_key}, {practice_id, alias_text} | id → all; (practice_id, norm_key) → id, alias_text; (practice_id, alias_text) → id, norm_key; **alias_text → norm_key** | **3NF, not BCNF** (section 4) |
| category | {id}, {name} | id → name, cat_group; name → id, cat_group | BCNF |
| practice_category | {practice_id, category_id} | none (all-key) | BCNF |
| body_part | {id}, {name} | id ↔ name | BCNF |
| practice_body_part | {practice_id, body_part_id} | (practice_id, body_part_id) → action | BCNF |
| benefit | {id}, {name} | id ↔ name | BCNF |
| practice_benefit | {practice_id, benefit_id} | none (all-key) | BCNF |
| condition | {id}, {name} | id → name, cond_type; name → id, cond_type | BCNF |
| source | {id} | id → all | BCNF |
| contra | {practice_id, condition_id} | key → severity, note, evidence, source_id | BCNF |
| category_contra | {category_id, condition_id} | key → severity, note, evidence, source_id | BCNF |
| media | {id}, {url} | id → all; url → all | BCNF |
| app_user | {id} | id → name, created_at | BCNF |
| user_condition | {user_id, condition_id} | key → start_date, end_date | BCNF |
| favourite | {user_id, practice_id} | key → added_at | BCNF |

**The schema is in 3NF.** Fifteen relations are also in BCNF.

## 4. Design decisions that were checked

| Question | Answer |
|---|---|
| `alias.norm_key` is calculated from `alias_text`. Is that a 3NF violation? | **No, but it isn't BCNF.** `alias_text` is not a superkey, but `norm_key` is prime because it's part of the candidate key (practice_id, norm_key), so the 3NF condition holds. It's stored so that search can use an index. It is a generated column, so the update anomaly that a repeated value normally causes can't occur. |
| Does `media.url` determine `author` and `licence`? | Yes, an image's credit depends on the image. If the same URL could appear in two rows, that would be a transitive dependency. `url` is declared UNIQUE, which makes it a candidate key. A variation reuses its parent's picture through `parent_id`, without a second row. |
| Does `source_id` determine `evidence` in `contra`? | No. `evidence` is the strength of one claim, while `source.src_type` is the kind of publication. One book can carry both an expert claim and a traditional claim, so `evidence` depends on the whole (practice, condition) key. |
| Does `name_sa` determine `name_kn`? | No FD is assumed. Transliteration is only a starting point during seeding, and curators can correct each row individually. Keeping the names as columns is ADR-006. |
| Does `name_en` determine `slug`? | No FD is assumed. `slug` is an identifier the curator assigns and can change to tell apart two practices with the same English name. |
| Is `type` redundant with `parent_id`? | No. "Variation" is not a `practice_type` value (see data-spec). `type` says what kind of practice it is, and `parent_id` says whether it is a variation. |
| Does `contra` duplicate `category_contra`? | No. They store different facts at different levels, with different keys. The combined result is only calculated by a view, never stored. |

## 5. Final tables match the ER diagram
Every relation in section 1 appears exactly once in [er/er-logical.png](er/er-logical.png), and every entity and relationship in [er/er-conceptual.png](er/er-conceptual.png) has a row in the section 1 mapping table.
