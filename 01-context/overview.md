# Context — Simple Stock Flow

> Derived from `spec/data-model.md` (*the model*, verified against PostgreSQL 16.14 on 2026-09-19; see header). `S-nn` = assumption (list in `05-architecture` §9).

## 1. What it is

**Simple Stock Flow** is an internal **single-currency inventory and sales** system [D-05]. It maintains a product catalog with stock and category, registers immutable sales attributed to an operator, and produces an aggregated report by product over a date range [§1, §2].

## 2. For whom

Internal operators with `admin` or `seller` role [§1 Role]. **There is no client or buyer as a user or as an entity** [§1 User]. A supplies or hardware store is inferred (S-01) [§9.1].

## 3. What is built

| Capability | Source |
|---|---|
| Catalog: create, modify, restock and deactivate products (logical deletion) | [§2.2], [R-07] |
| Five fixed categories, read-only | [§2.1], [§9.1], [D-10] |
| Optional image per product, as an opaque key to external storage | [§1], [D-08] |
| Sales registration with frozen price, name and category lines | [§2.3], [§2.4] |
| Consult a sale and list by range | [§6.1 Q6, Q7] |
| Sales report aggregated by product, calculated in the engine | [§1], [Q9], [D-06] |
| Authentication with two roles; registration of sellers by an `admin` | [§2.5], [§11 H-3, DP-04] |

## 4. What is **not** built

| Out of scope | Why | Source |
|---|---|---|
| Client or buyer entity | The system registers the operator, not the buyer | [§1], [§7] |
| Multi-currency / currency column | Single-currency by construction | [D-05], [§3] |
| Extra product attributes (description, SKU, code) | Decided and will not be reopened | [§1], [DP-03], [§12] |
| Category maintenance (CRUD) | Read-only seed data | [§2.1], [D-10], [§4.1] |
| Editing or deleting sales | Immutable accounting record | [§2.3], [§7.1] |
| Report broken down by seller | Decided (would cross personal data) | [DP-02], [§7.1] |
| `created_at`/`updated_at` audit | Closed decision; no requirement | [§8] |
| Payments, cards, health data | Do not apply | [§7] |
| External analytics or export | Not defined: conscious omission | [§7.1] |
| Granting the `admin` role at runtime | Provisioned by the deployment | [DP-04], [§9.2] |
| API contract, business requirements and decisions D-01…D-10 | Live in documents that are not part of this model | [§12] |

## 5. Constraints and inherited decisions

PostgreSQL 16, `sales` schema, EF Core with migrations as the sole owner of the DDL, hexagon with domain in C#, binaries outside the database, code and column names in English [header], [§3.2, ADR-001], [§2.5], [D-08], [Art. XI]. Detail in `05-architecture`.

## 6. Current state (according to the model)

- **Applied migrations:** four, from `InitialSchema` to `RenameTablesToSingular` [§3.2].
- **Data as of 2026-09-19:** `category` = 5, `user` = 1 (the startup administrator) and `product`, `sale`, `sale_item` = 0 [§10.4].
- **Debt declared and settled on 2026-09-20:** D-1, D-2 and D-3 [§13].
- **Pending:** T-05, T-11, T-12, T-13 and T-20 (partial) [§4], [§6.2].
- **Internal contradictions of the model** (DM-1…DM-6): see `05-architecture` §10.3.

## 7. Limits of this documentation

- Every assertion cites the model. What does not come from it is marked as an assumption (S-01…S-15).
- If this documentation and the engine disagree, **the engine wins** [Art. X]; verification is done with the three queries from §10 of the model.
- The model links to `constitution.md`, `plan.md`, `adr/` and other documents that **are not delivered**; identifiers D-xx, T-xx, ADR-xx and DP-xx are cited as mentioned by the model, without inventing their content.

## 8. Document map

| Folder | What it contains |
|---|---|
| `01-context` | This document: what is built and what is not |
| `02-domain` | Glossary, entities, rules, events |
| `03-product` | Problem, vision, indicators |
| `04-requirements` | 12 user stories and 10 non-functional requirements |
| `05-architecture` | Hexagonal style, aggregates, ports, where each rule lives, closure |
| `06-data` | **Is not written:** it is the delivered model (`spec/data-model.md`) |
