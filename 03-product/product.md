# Product — Simple Stock Flow

> Derived from `spec/data-model.md` (*the model*). The model **does not state** the business problem: it is inferred from what its design prevents. What is inferred is marked `S-nn` (see `05-architecture` §9).

## 1. The problem

**Who has it.** The internal operators who manage the catalog and register sales (`admin` and `seller`). The buyer **is not** a user of the system [§1 User].

**What business.** The five seeded categories —General, Tools, Electricity, Plumbing, Paints— suggest a supplies or hardware store (S-01) [§9.1].

**How it is solved today.** The model does not describe the previous process: **nothing is asserted** about it.

**The pain, read from what the design prevents:**

| Failure that the system prevents | Decision that prevents it | Source |
|---|---|---|
| Negative stock or overselling with simultaneous operators | `CHECK stock >= 0` + optimistic concurrency | [§2.2], [§3 `xmin`], [ADR-002], [D-04] |
| Past reports that change when renaming or repricing a product | Sale line with **frozen** name, price and category | [§1 Frozen name], [§2.4], [ADR-004], [§11.1] |
| Sales pointing to a deleted product | Logical deletion + FK-3 `RESTRICT` | [§2.2], [§5], [§7.1] |
| Loose or duplicated lines | `sale_id NOT NULL`, unique `(sale_id, product_id)`, FK-2 | [§2.4], [§4] |
| Exposed user passwords | Hash port, never indexed nor logged | [§2.5], [§7] |
| Inflated scope without requirement | Closed attributes, without client, without extra currency, without audit | [§1, DP-03], [§8], [D-05] |

**Why it is worth solving.** An invariant that only lives in C# protects the application, not the data; the model exists to say **where each rule lives** and not to promise what is not applied [«How to read this document»].

## 2. Users

| User | Need | Source |
|---|---|---|
| `admin` | Maintain catalog and stock, register sellers, view the report (S-03) | [§1 Role], [§11 H-3] |
| `seller` | Search products and register sales (S-03) | [§1 User] |

## 3. Vision

**For** internal operators of a business with a catalog and inventory (S-01), **who** need to sell without unbalancing the stock or rewriting their history,
**Simple Stock Flow** is a **single-currency** inventory and sales system
**that** maintains a catalog with non-negative stock, registers immutable sales with frozen price, name and category, and delivers aggregated reports by product over a date range.
**Unlike** a system whose past sales read the live catalog [§1 Frozen name],
**our product** guarantees that **what is sold is not rewritten** and that each rule declares where it lives.

## 4. Product principles

1. **What is sold is not rewritten.** Sales are neither edited nor deleted, and the report for a closed range does not change [§2.3, §7.1, §11.1].
2. **A rule lives where it can be guaranteed, and what is not guaranteed is declared.** The engine / domain only / pending marks are part of the product [«How to read this document»].
3. **Deliberate minimum scope.** Product with five attributes, five fixed categories, no client, no extra currency, no audit [DP-03, D-10, §1, D-05, §8].
4. **Small privacy surface.** The operator is recorded, not the buyer [§7].

## 5. Success indicators (derived, S-15)

The model does not define business metrics; these indicators are **verified against the invariants**:

| Indicator | How it is checked | Source |
|---|---|---|
| Zero products with negative stock | `SELECT` on `product.stock < 0` returns no rows | [§2.2] |
| Report for a closed range identical before and after renaming a product | Execute it twice | [§1], [§11.1] |
| Zero sale lines without a sale | `sale_item.sale_id` is `NOT NULL` | [§4], [§13 D-2] |

## 6. Already planned direction

The model does not give a product horizon, but it does have pending tasks that describe where it is being refined:

| Task | What it contributes | Source |
|---|---|---|
| T-12 | The sale goes on to reference the user by foreign key (`sold_by_user_id`, FK-4) | [§3], [§5] |
| T-13 | Indexes that support search and reporting | [§6.2] |
| T-20 | "Domain only" rules drop to the engine | [§4] |
| T-05 | Explicit currency guard when adding lines | [§2.3] |
| T-11 | Frozen category in the sale line | [§2.4] |

## 7. Outside the vision

Clients or buyers as an entity; multi-currency; extra product attributes (description, SKU); category maintenance; `created_at`/`updated_at` audit; report by seller; editing or deleting sales. Sources: [§1], [D-05], [DP-03], [D-10], [§8], [DP-02], [§2.3].

## 8. Open questions

- **CA-06.1:** "one row per product" versus the decision in §11.1 — pending owner's decision [§11.1].
- **H-2:** orphan image binaries without a cleanup process — it is operational [§11].
- **User registration:** the `admin` role is no longer an open question: no one grants it at runtime [DP-04, §11 H-3].
