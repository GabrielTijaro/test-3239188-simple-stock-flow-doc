# Domain — Simple Stock Flow

> Derived from `spec/data-model.md` (*the model*). `R-nn` = numbered rule in `05-architecture` §6; `S-nn` = assumption. The model is in English for code and columns [Art. XI]; here the business terms are in Spanish (translated to English for this context) and the technical term in parentheses.

## 1. Glossary

| Term | Definition | Where it lives | Source |
|---|---|---|---|
| Product | Catalog item: name, price, stock, category and optional image, and nothing else | `Product` · `product` | [§1], [DP-03] |
| Category | Product classification. Fixed set of five, seeded, without maintenance | `Category` · `category` | [§1], [D-10] |
| Price | Current value of the product; strictly positive | `Money` · `product.price` | [§1] |
| Stock | Available units; never negative | `product.stock` | [§1] |
| Image | **Opaque key** of the external binary; absent = `NULL` | `product.image_key` | [§1], [D-08] |
| Logical deletion | Removal from the catalog without deleting the row | `product.deleted_at` | [§2.2], [D-03] |
| Sale | Consummated and **immutable** commercial event: who, when and what | `Sale` · `sale` | [§1] |
| Sale line | Item row: product, quantity and **frozen** price; does not exist outside its sale | `SaleItem` · `sale_item` | [§1] |
| Quantity | Units of a line; strictly positive | `Quantity` | [§1] |
| Frozen | Copy of the value at the time of the sale; it does not follow the catalog and is not denormalization | `product_name`, `unit_price`, `category_name` | [§1] |
| Total / Subtotal | Sum of subtotals / price × quantity. **They are calculated, not stored** | `Sale.Total`, `SaleItem.Subtotal` | [§1], [Art. VII] |
| User | Internal operator who authenticates and registers sales. **There is no client entity** | `User` · `user` | [§1] |
| Role | `admin` or `seller` (closed set of two) | `user.role` | [§1], [§2.5] |
| Password hash | Irreversible footprint; the domain never sees the password | `user.password_hash` | [§1], [D-09] |
| Date range | Report window; the end cannot be earlier than the start | application value object | [§1] |
| Sales report | Aggregation by product over a range; **is not persisted** | read model | [§1], [D-06] |

## 2. Context map

The model describes **a single system** with a single schema [§3]. Three areas are identified within a single context (S-02):

| Area | Aggregates | Relates to | How |
|---|---|---|---|
| Catalog | `Product`, `Category` (reference) | Sales | A line references a product by identity (FK-3) and `Sale.AddItem` calls `Product.Withdraw` [§5], [§2.3] |
| Sales | `Sale` → `SaleItem` | Catalog, Identity | Withdraws stock; attributes the sale to a user (`sold_by`; FK-4 pending T-12) [§2.3], [§5] |
| Identity | `User` | Sales | Authorship of the sale [§5] |
| Report | read model | Sales | Aggregates over `sale` and `sale_item`; without its own table [§1, D-06] |

## 3. Entities, aggregates and invariants

| Entity | Role in the model | Identity | Attributes | Invariants |
|---|---|---|---|---|
| `Category` | Reference: **not a root**, no lifecycle, read only | `id` | `name` | R-23 |
| `Product` | Root (catalog) | `id` | `name`, `price`, `stock`, `category_id`, `image_key`, `deleted_at` | R-01…R-07 |
| `Sale` | Root (sales) | `id` | `sold_at`, `sold_by` (→ `sold_by_username` in T-12), `sold_by_user_id` (pending T-12) | R-08…R-12 |
| `SaleItem` | Internal to `Sale` | `id` | `product_id`, `product_name`, `quantity`, `unit_price`, `sale_id`, `category_name` | R-13…R-17 |
| `User` | Root (identity) | `id` | `username`, `password_hash`, `role` | R-18…R-22 |

Sources: [§2.1–§2.5], [§3]. Model discrepancies regarding columns (`category_name`, count 21/22): **DM-1…DM-3** in `05-architecture` §10.3.

## 4. Value objects (without identity or table)

| Object | Rules | Source |
|---|---|---|
| `Money` | Rounds to 2 decimals `AwayFromZero`; rejects negatives but **allows 0** (which is why `price > 0` lives in `ChangePrice`); requires the same currency when adding | [§2.2], [§2.3] |
| `Quantity` | Strictly positive | [§1], [§2.4] |
| `DateRange` | The end cannot be earlier than the start; lives in the application layer | [§1] |
| `Roles` | Closed set `admin`, `seller` (`Roles.IsValid`) | [§2.5] |

## 5. Relationships

| Source → Destination | Cardinality | Nature | Business rule | Source |
|---|---|---|---|---|
| `category` → `product` | 1:N | aggregate crossing | Every product has **exactly one** category; there can be categories without products | [§5], FK-1 |
| `sale` → `sale_item` | 1:N | **internal** (composition) | A sale has at least one line | [§5], FK-2 |
| `sale_item` → `product` | N:1 | aggregate crossing | Points to an existing and non-deleted product when selling | [§5], FK-3 |
| `sale` → `user` | N:1 | aggregate crossing | Authorship cannot be left orphan | [§5], FK-4 (pending) |
| `sale` ↔ `product` | N:M | resolved by `sale_item`, which carries `quantity`, `unit_price`, `product_name`, `category_name` | The **only** N:M in the model | [§5] |

## 6. Lifecycles

- **Product:** active → deactivated (`deleted_at`). There is no reactivation (S-08) [§2.2]. Its stock varies with `Withdraw` and `Restock` [§2.2].
- **Sale:** built → confirmed (`EnsureConfirmable`: at least one line) → registered and immutable (S-12) [§2.3].
- **User:** created with a role; there is no deletion or role change operation (S-08) [§7.1].
- **Category:** no lifecycle; born in the initial migration [§2.1, §9.1].

## 7. Domain events

> **S-06:** the model does not define events nor persist them (there is no audit or event table [§2, §8]). The list is **derived from the operations** in §2; the report does not consume events: it is calculated in the engine [D-06].

| Event (past) | Provoked by | Data it carries | Source |
|---|---|---|---|
| `ProductCreated` | Product registration | id, name, price, stock, category | [§2.2] |
| `ProductRenamed` / `ProductPriceChanged` / `ProductRecategorized` | `Rename` / `ChangePrice` / `SetCategory` | previous and new value | [§2.2] |
| `ProductImageAttached` | `AttachImage` | opaque key or `NULL` | [§2.2], [§7.1] |
| `StockWithdrawn` | `Withdraw` (per line) | product, quantity | [§2.3] |
| `StockRestocked` | `Restock` | product, quantity | [§2.2] |
| `ProductDeactivated` | Logical deletion | product, `deleted_at` | [§2.2] |
| `SaleRegistered` | Sale confirmation | id, `sold_at`, user, frozen lines | [§2.3], [§2.4] |
| `UserRegistered` | User registration | id, normalized name, role | [§2.5], [§11 H-3] |

## 8. Policies (reactions)

| When… | then… | Source |
|---|---|---|
| a line is added to a sale | stock is withdrawn from the product in the same operation | [R-11], [§2.3] |
| a line is added | name, price and category of the product are copied | [§2.4] |
| a product is deactivated or its image is replaced | the key is nullified, committed and **then** the binary is deleted | [§7.1] |
| an attempt is made to physically delete a product with sales | the engine rejects it (FK-3) | [§5] |
| a product is recategorized | past sales keep their label; the report shows two rows | [§11.1] |

## 9. Domain questions unanswered in the model

- Reactivate a deactivated product or change a user's role: **not defined** (S-08).
- Who can restock: **not defined** (S-03).
- CA-06.1 versus §11.1: **pending owner's decision** [§11.1].
- Orphan binaries: **H-2**, without process owner [§11].


