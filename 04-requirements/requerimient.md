# Requirements — Simple Stock Flow

> Derived from `spec/data-model.md` (*the model*). Each criterion cites its source; `S-nn` = assumption (list in `05-architecture` §9); `R-nn` = numbered rule in `05-architecture` §6.
> The model does **not** define the error format nor the API contract [§12]: the criteria describe behavior, not format (S-11).

## 1. Actors

| Actor | What it is | Source |
|---|---|---|
| `admin` | Internal operator with `admin` role | [§1 Role], [§2.5] |
| `seller` | Internal operator with `seller` role | [§1 Role], [§2.5] |
| *(does not exist)* | **Client or buyer:** is not a user nor a system entity | [§1 User] |

**Permission matrix (S-03).** The model only sets row 2: the registration of sellers is done by an `admin` and a `seller` receives 403 [§11 H-3, §13 D-3]. The rest is assumed.

| Operation | admin | seller |
|---|---|---|
| Login | yes | yes |
| Register sellers | yes | **no** [§13 D-3] |
| Create, modify, restock and deactivate products | yes | no |
| Search products and consult categories | yes | yes |
| Register, consult and list sales | yes | yes |
| Sales report | yes | no |

## 2. User stories

### HU-01 · Login
**As an** operator (`admin` or `seller`) **I want to** authenticate myself **so that** I can operate with my role. · Source: [Q10], [R-19], [R-20], [R-22]
- **Given** an existing user, **when** they send their username and correct password, **then** they log in with their role [§1, §2.5].
- **Given** a name with uppercase letters or spaces ("Ana "), **when** they log in, **then** they are searched by the normalized value, by exact match [§2.5, Q10].
- **Given** an incorrect password or non-existent user, **then** it is rejected without revealing which one failed (S-05).
- The verification is done by the hash port; the hash does **not** appear in responses, logs, or errors [§2.5, §7].

### HU-02 · Register sellers
**As an** `admin` **I want to** create `seller` users **so that** they can register sales. · Source: [§11 H-3, DP-04], [§9.2], [§13 D-3], [R-18…R-21], [R-31]
- **Given** an authenticated `admin`, **when** they create a `seller` user, **then** it is saved with a normalized name and a hash produced by the hash port [R-19, R-22].
- **Given** an already existing name (normalized), **then** it is rejected [R-18, `IX_user_username`].
- **Given** an authenticated `seller`, **when** they attempt registration, **then** they receive 403; **without a token**, 401 [§13 D-3].
- **Given** any attempt to assign the `admin` role at runtime, **then** it is rejected: no one grants it; the initial `admin` is provisioned by the deployment from the environment [DP-04, §9.2].
- **Given** a role other than `admin`/`seller`, **then** it is rejected [R-21].

### HU-03 · Create product
**As an** `admin` **I want to** register a product **so that** it can be offered in the catalog. · Source: [§2.2], [§1, DP-03], [R-01…R-03], [R-05], [R-06], [R-26]
- **Given** a name, price > 0, stock ≥ 0 and an existing category, **then** it is created as active.
- **Given** an empty name or one with only spaces, **then** it is rejected; the name is saved trimmed [R-01].
- **Given** a price ≤ 0 or negative stock, **then** it is rejected [R-02, R-03]. Note: until T-20, a price ≤ 0 is only prevented by the domain [§2.2].
- **Given** a missing or non-existent category, **then** it is rejected [R-05, FK-1].
- **Given** a price with more than two decimals, **then** it is rounded to 2 with `AwayFromZero` [R-26].
- The product has a **name, price, stock, category and optional image, and nothing else**: no description or SKU [DP-03, §1]. Without an image, it is saved as `NULL`, never an empty string [R-06].

### HU-04 · Modify product
**As an** `admin` **I want to** rename, change the price or category, and manage the image **so that** the catalog is kept up to date. · Source: [§2.2], [§1 Frozen name], [§7.1], [R-15], [R-16], [R-30]
- **Given** a new price > 0, **then** it is applied; already registered sales **keep** their price at the time [R-15].
- **Given** a rename or recategorization, **then** sales and reports for closed periods do not change [§1, R-16].
- **Given** an image replacement, **then** the key is updated, the transaction is committed, and **then** the previous binary is deleted [R-30]. An orphan binary due to failure is accepted [§11 H-2].
- **Given** two concurrent modifications of the same product, **then** one fails (S-14) [D-04].

### HU-05 · Restock
**As an** `admin` **I want to** add units to the stock **so that** new inventory is reflected. · Source: [§2.2 `Product.Restock`], [R-03]
- **Given** a quantity > 0 (S-07), **then** the stock increases by that amount.
- The stock never becomes negative after the operation [R-03].
- **Given** concurrent restocking, **then** none are lost: the losing one fails and retries (S-14) [D-04].

### HU-06 · Deactivate a product
**As an** `admin` **I want to** remove a product from the catalog **so that** it is no longer offered without losing its history. · Source: [R-07], [§7.1], [§5 FK-3, cardinalities]
- **Given** an active product, **when** it is deactivated, **then** it is marked with `deleted_at` and stops appearing in searches and active batches [R-07, Q1, Q3].
- Its row, its sales, and the report that counts it are preserved [§7.1].
- It is never physically deleted; a manual `DELETE` with associated sales **fails** [§5 FK-3].
- Its image binary **is deleted** [§7.1].
- **Given** a deactivated product, **then** it cannot be sold [§5].
- Reactivation does not exist (S-08).

### HU-07 · Search products
**As an** operator **I want to** search by partial text and category **so that** I can find the product to sell or edit. · Source: [Q1]
- **Given** a partial text and an optional category, **then** it returns **only active ones**, ordered by name, **paginated and with a total** [Q1].
- A deactivated product does not appear [Q1].

### HU-08 · Consult categories
**As an** operator **I want to** see the categories **so that** I can classify and filter. · Source: [Q4], [Q5], [§2.1], [§9.1], [D-10]
- It returns the **five** seeded categories, ordered by name [Q4, §9.1].
- One can be obtained by identifier [Q5].
- There is no operation to create, rename, or delete categories [§2.1, D-10].

### HU-09 · Register a sale
**As an** operator **I want to** register a sale of one or more products **so that** there is a record of what was sold and to deduct stock. · Source: [§2.3], [§2.4], [R-08…R-17], [R-24], [Q3]
- **Given** at least one line (active product and quantity > 0), **when** they confirm, **then** it is registered with `sold_at` and the user who performs it [R-08, §3].
- Each line **freezes** the name, unit price, and category name [R-15, R-16].
- Each line withdraws stock in the same operation [R-11].
- **Given** insufficient stock in any line, **then** the operation fails and the sale is not registered [R-04, S-04, S-12].
- **Given** the same product twice, **then** it is rejected [R-10]. With 0 or negative quantity, it is rejected [R-14]. Without lines, it is not confirmed [R-09]. With a deactivated product, it is rejected [§5].
- Total and subtotals **are calculated**, not stored [R-24].
- **Given** two operators competing for the last unit, **then** exactly one gets it and the stock does not become negative [RNF-01, D-04].
- Once registered, the sale **is neither edited nor deleted** [R-12].

### HU-10 · Consult a sale
**As an** operator **I want to** see a sale with its lines **so that** I can verify what was sold. · Source: [Q6], [R-15], [R-24], [§7]
- **Given** an existing id, **then** it returns the header and the lines with the **frozen** values and the calculated total [Q6].
- If the product changed later, the sale shows the values of that moment [§1].
- Who performed it is personal data and has restricted access [§7].

### HU-11 · List sales by range
**As an** operator **I want to** list sales between two dates **so that** I can review the activity. · Source: [Q7], [Q8], [R-27]
- **Given** a valid range, **then** it returns sales **paginated, newest first, with a total** [Q7].
- **Given** an end date before the start date, **then** it is rejected [R-27].
- The unpaginated query (Q8) is not exposed: it has no consumer [§6.1].

### HU-12 · Sales report by range
**As an** `admin` **I want to** have an aggregated report by product **so that** I can analyze what was sold. · Source: [Q9], [§1 Report, D-06], [§11.1], [DP-02], [R-27], [R-32], [R-33]
- **Given** a valid range, **then** it returns the aggregation by product, ordered by descending amount, **calculated in the engine and not persisted** [Q9, §1].
- It groups by **frozen** `product_id`, `product_name` and `category_name`: a product with two labels appears in **two rows** [§11.1].
- A report of a closed range **does not change** when renaming, repricing or recategorizing products later [§1, §11.1].
- It is **not** broken down by seller [DP-02].
- **Given** an end date before the start date, **then** it is rejected [R-27]. With a range without sales, it returns an empty result (S-13).
- Note: `spec.md` CA-06.1 says "one row per product"; the current wording is "one row per product **and frozen label**" — pending decision from the owner [§11.1].

## 3. Non-functional requirements

> The model does not set numerical thresholds (S-09): they are not invented here. Each requirement is formulated as a **verifiable** criterion against the model.

| ID | Requirement | Verifiable criterion | Source |
|---|---|---|---|
| RNF-01 | Stock integrity under concurrency | A manual `UPDATE` to `stock = -1` is rejected by `ck_product_stock_non_negative`; two concurrent sales for the last unit → one fails | [§2.2], [§4], [D-04], [ADR-002] |
| RNF-02 | Sale atomicity | There is no observable state where the stock decreased and the line does not exist, nor vice versa | [R-11], S-04 |
| RNF-03 | Immutable history and stable report | There is no operation to edit or delete sales; frozen values do not follow the catalog; a report of a closed range is identical before and after editing the catalog | [§2.3], [§1], [§7.1], [§11.1] |
| RNF-04 | Single currency and monetary precision | `numeric(18,2)`; `Money` rounds to 2 decimals `AwayFromZero`; no currency column; if one changes, the other changes in the same migration | [§2.2], [§3], [D-05] |
| RNF-05 | Credential security | The password is never seen in cleartext; the hash is produced by a port; never in logs, responses, projections or errors; never indexed; no history; initial credentials from the environment, not versioned | [§7], [§2.5], [§9.2], [Art. IX] |
| RNF-06 | Role-based authorization | User registration: without token → 401, `seller` → 403; the `admin` role is not granted at runtime | [§13 D-3], [§11 H-3, DP-04] |
| RNF-07 | Privacy and retention | `username` is personal data with restricted access; indefinite and never deleted sales; products with logical deletion; the image binary is indeed deleted; the report does not expose personal data | [§7], [§7.1], [DP-02] |
| RNF-08 | Performance of frequent patterns | Q1, Q3, Q7, Q9 and Q10 are of high frequency; Q9 is the most expensive: aggregation in the engine + unique composite index with `INCLUDE`; Q1: partial and trigram index. Status: **pending (T-13)**. Without figure (S-09) | [§6.1], [§6.2] |
| RNF-09 | Engine integrity and verifiability | The "domain only" rules drop to the engine (T-20); DDL only via migrations; the three queries in §10 return what the model marks promise | [§4], [§3.2, ADR-001], [§10] |
| RNF-10 | Temporal consistency | All timestamps are `timestamptz`; the server runs in UTC | [§3] |

## 4. Traceability HU → model

| HU | Tables | Pattern | Rules |
|---|---|---|---|
| HU-01 | `user` | Q10 | R-19, R-20, R-22 |
| HU-02 | `user` | Q10 | R-18…R-21, R-31 |
| HU-03 | `product`, `category` | Q5 | R-01…R-03, R-05, R-06, R-26 |
| HU-04 | `product`, `category` | Q2, Q5 | R-02, R-15, R-16, R-30 |
| HU-05 | `product` | Q2 | R-03 |
| HU-06 | `product` | Q2 | R-07 |
| HU-07 | `product` | Q1 | R-07 |
| HU-08 | `category` | Q4, Q5 | R-23 |
| HU-09 | `sale`, `sale_item`, `product` | Q3 | R-04, R-08…R-17, R-24 |
| HU-10 | `sale`, `sale_item` | Q6 | R-15, R-24 |
| HU-11 | `sale` | Q7 | R-27 |
| HU-12 | `sale`, `sale_item` | Q9 | R-27, R-32, R-33 |
