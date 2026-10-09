Architecture — Simple Stock Flow
Reconstructed from spec/data-model.md (hereafter, the model). First pass (challenge order 1); the closing review (order 6) is in §10.
Citation convention: [§2.3] = section of the model · [FK-3], [Q9], [D-04], [T-20], [ADR-002], [DP-03], [Art. X] = identifiers defined by the model and not redefined here.
Assumption (S-nn) = not stated in the model; the complete list is in §9.
1. How the Model Was Read
Every rule carries one of three labels: database-enforced, domain-only, or pending (T-xx) [“How to read this document”]. These labels are preserved as-is: a “domain-only” rule is not presented as guaranteed.
If the document contradicts the database engine, the engine takes precedence [header; Art. X]. The engine is not accessible from this repository, so when the model contradicts itself, the most recently dated statement (§13, 2026-09-20) takes precedence, and the contradiction is recorded (DM-n, §10.3).
Nothing the model does not state is asserted as fact; it is marked as an assumption.
2. Architectural Style
Decision	What the Model Says	Source
Hexagonal architecture (ports and adapters)	“By hexagonal design”; hash port; report-reading port; persistence adapter is where the mapping lives; paths src/domain/ and src/adapters/outbound/persistence/Configurations/	[§2.5], [§1 Report, D-06], [§0], [§12]
Rich domain model with aggregates (tactical DDD)	Three aggregate roots and one reference entity	[§2.1–§2.5]
One system, one database, one schema	“The sales schema groups the entire system” → modular monolith (S-02)	[§3]
Persistence	PostgreSQL 16, EF Core (shadow properties, global filter, migrations), C#	[header], [§2.2], [§3.2], [§0]
Image binaries stored outside the database	External storage; the database stores only an opaque key	[§1 Image, D-08], [§7.1]
Migrations exclusively own the schema	Nothing else writes DDL	[§3.2, ADR-001]
3. Component View
flowchart LR
  subgraph ENT[Inbound Adapters]
    API[HTTP API with token authentication]
  end
  subgraph APP[Application]
    UC[Use Cases]
    DR[DateRange - Value Object]
  end
  subgraph DOM[Domain - src/domain]
    P[Product]
    S[Sale with SaleItem]
    U[User]
    C[Category - Read Only]
  end
  subgraph SAL[Outbound Adapters]
    REPO[EF Core Persistence - PostgreSQL sales Schema]
    RPT[Report Reader]
    HASH[Password Hashing]
    IMG[Image Storage]
  end
  API --> UC
  UC --> DR
  UC --> DOM
  DOM -- repository ports --> REPO
  UC -- read port --> RPT
  UC -- hash port --> HASH
  UC -- image port --> IMG


Code dependencies always point toward the domain; adapters implement the ports.

4. Aggregates
Aggregate	Root	Inside	Value Objects	Protects	Source
Catalog	Product	—	Money, image key	R-01…R-07	[§2.2]
Sales	Sale	SaleItem (internal constructor: only Sale.AddItem creates it)	Money, Quantity	R-08…R-17	[§2.3], [§2.4]
Identity	User	—	Role (Roles)	R-18…R-22	[§2.5]
Reference	Category: not a root, no lifecycle, read-only repository	—	—	R-23	[§2.1]
Value objects do not have their own tables: they live in their owner's row [§2, D-07].
Aggregates reference one another by aggregate root identity: product→category, sale_item→product, sale→user [§5, cardinalities].
There are no report, audit, or counter tables [§2].
5. Ports
Port	Direction	What It Provides	Pattern	Source
Product repository	Outbound	Search (partial text, category, active only, sorted by name, paginated and with count); by ID; by batch of active IDs (read before writing stock); save with concurrency token	Q1, Q2, Q3	[§6.1], [D-04]
Category repository	Outbound, read-only	List sorted by name; retrieve by ID. No method creates, renames, or deletes	Q4, Q5	[§2.1], [§6.1]
Sale repository	Outbound	Sale with its line items; sales within a date range (paginated, with count); save a confirmed sale. No editing or deletion	Q6, Q7	[§6.1], [§2.3]
Unpaginated sales-by-range query	—	Remove from the port: it has no consumer	Q8	[§6.1]
User repository	Outbound	Exact username lookup (on every login)	Q10	[§6.1]
Report-reading port	Outbound	Product-level aggregation over a date range, computed by the database engine, returning a read model	Q9	[§1], [D-06], [§6.1]
Hash port	Outbound	Generate and verify the hash; the only place that sees the plaintext password	—	[§2.5], [§9.2], [D-09]
Image storage (S-10)	Outbound	Store and delete binary files using an opaque key	—	[§1], [§7.1], [D-08]
HTTP API	Inbound	Token authentication and role-based authorization; the API contract is not included in the model (S-11)	—	[§13 D-3], [§3], [§12]
6. Where Each Rule Lives
R	Rule	Where It Lives	Label	Source
R-01	Product name is required, non-empty, and trimmed	Product.Rename; the engine only requires NOT NULL	Domain-only	[§2.2]
R-02	price > 0	Product.ChangePrice (Money allows 0)	Domain-only → T-20	[§2.2], [§4]
R-03	stock >= 0 after any operation	Product.Withdraw/Restock + ck_product_stock_non_negative	Database-enforced	[§2.2], [§4]
R-04	Withdrawing more stock than available fails	Product.Withdraw (process rule, not a CHECK)	Domain-only	[§2.2]
R-05	Category is required and must exist	FK_product_category_category_id (FK-1, RESTRICT)	Database-enforced	[§2.2], [§5]
R-06	No image = NULL, never an empty string	Product.AttachImage	Domain-only	[§2.2]
R-07	Soft deletion, never physical deletion	deleted_at + global filter	Database-enforced (T-09; see DM-4)	[§2.2], [D-03], [§13 D-1]
R-08	A sale records who performed it	Sale constructor; NOT NULL	Domain-only	[§2.3]
R-09	At least one line item is required for confirmation	Sale.EnsureConfirmable (would require a deferred trigger)	Domain-only	[§2.3]
R-10	A product cannot appear more than once in a sale	Sale.AddItem + unique index (sale_id, product_id)	Domain and database (T-20; DM-5)	[§2.3], [§4]
R-11	Deducting stock and adding the line item are one operation	Sale.AddItem calls Product.Withdraw	Domain-only	[§2.3]
R-12	Sales are immutable	No edit or delete port exists	Domain-only (by absence)	[§2.3], [§7.1]
R-13	A line item requires a product	NOT NULL + FK-3 (RESTRICT)	Database-enforced (T-20; DM-5)	[§2.4], [§5]
R-14	quantity > 0	Quantity constructor	Domain-only → T-20	[§2.4]
R-15	Line-item name and price are frozen	Sale.AddItem copies them from Product	Domain-only	[§2.4], [§1]
R-16	Category name is frozen, with deliberately no FK	sale_item.category_name NOT NULL	Database-enforced (T-11; DM-3)	[§2.4], [D-06], [ADR-004]
R-17	A line item cannot exist outside its sale	FK-2 (CASCADE) + sale_id NOT NULL	Database-enforced (T-20; DM-5)	[§2.4], [§5]
R-18	Username is required and unique	Constructor + IX_user_username	Database-enforced (uniqueness)	[§2.5]
R-19	Username is lowercase and trimmed	User.NormalizeUsername	Domain-only → T-20	[§2.5]
R-20	Hash is required and non-empty	User constructor; NOT NULL	Domain-only	[§2.5]
R-21	role must be in ('admin','seller')	Roles.IsValid	Domain-only → T-20	[§2.5], [§9.2]
R-22	The domain never sees the plaintext password	Hash port	By hexagonal design	[§2.5], [D-09]
R-23	Category name is required, non-empty, and unique	Category.Rename (domain-only → T-20); IX_category_name (database)	Mixed	[§2.1]
R-24	Total and subtotal are calculated, not stored	Sale.Total, SaleItem.Subtotal; no column	Domain	[§1], [Art. VII]
R-25	Single currency; no currency column	No table has one; the guard in Sale.AddItem is pending (T-05)	Design	[§3], [§2.3], [D-05]
R-26	Amount rounded to 2 decimal places using AwayFromZero	Money and numeric(18,2), which must change together	Domain + database	[§2.2]
R-27	Date range: end cannot precede start	Application-layer value object, no table	Application	[§1]
R-28	No column uses DEFAULT	Values are supplied by the domain	Database (by absence)	[§3]
R-29	Timestamps use timestamptz; server runs in UTC	Column type	Database	[§3]
R-30	When deleting a binary: clear image_key, commit, and then delete the binary; no atomicity	Application + external storage	Procedure	[§7.1]
R-31	The initial admin is created at startup using environment credentials; nobody grants admin at runtime	Application startup + API	Outside the schema	[§9.2], [§11 H-3], [D-10], [DP-04]
R-32	The report groups by the frozen category value	Read-port query	Query	[§11.1], [ADR-004]
R-33	The report is not broken down by seller	Read port	Decision	[DP-02], [§7.1]

Declared technical debt: the five value-related “domain-only” rules (R-02, R-14, R-19, R-21, and the non-empty requirement in R-23) are to be enforced by the database in T-20 [§4]. The model's criterion: if a constraint is violated, something wrote outside the adapter [ADR-002, §2].

7. Consistency and Concurrency
Optimistic concurrency: PostgreSQL's xmin as a concurrency token, exposed as a shadow property (T-10) [§3, D-04].
Last line of defense: ck_product_stock_non_negative [§2.2, ADR-002].
Contention point: Q3, the product read that precedes the stock write [§6.1].
Operation across two aggregates: Sale.AddItem modifies Product and Sale as “one operation” [R-11]. The model does not specify how this is implemented: a database transaction is assumed (S-04), and in the event of a conflict, the losing write is assumed to fail (S-14).
Binaries: storage does not participate in the database transaction, so atomicity is not guaranteed [R-30].
8. Persistence, Security, and Privacy
sales schema, five singular table names: category, product, sale, sale_item, user. The table names are singular, not the schema name. user does not need quoting because it is schema-qualified [§0].
Naming convention: EF-generated names retain their style (PK_, IX_, FK_); manually written constraints (CHECKs) use snake_case: ck_{table}_{rule} [§3.1].
Applied migrations: four, from InitialSchema to RenameTablesToSingular [§3.2].
Seeding: the five categories are inserted by the initial migration with fixed IDs. The initial administrator is not seeded through SQL [§9.1–§9.2].
Indexes: access patterns and their status (present / missing) are documented in [§6.2]. Three are missing (T-13): the partial (category_id, name) index, the trigram index, and the unique composite index with INCLUDE. pg_trgm is installed by the migration that creates the index, never by db/init/ [§6.2].
Attribute-by-attribute privacy: username is personal data; password_hash is a secret (never in logs, responses, or errors, and never indexed); there is no end-customer data [§7].
9. Assumptions
S	Assumption	Why It Is an Assumption
S-01	Hardware store / supplies business	Inferred only from the seeded categories [§9.1]
S-02	Modular monolith: one API and one database	The model only states that the schema groups the system [§3]
S-03	Role-based permission matrix (see 04-requirements)	The model only specifies seller creation [§11 H-3, §13 D-3]
S-04	Recording a sale is a database transaction	The model says “one operation” [§2.3]
S-05	Invalid credentials produce a generic message	The model does not define the message
S-06	Domain events are derived from operations	The model does not define events or persist them [§2, §8]
S-07	Restocking requires quantity > 0	Positivity is only specified for sale line items [§1]
S-08	No product reactivation, role changes, or user editing	The model does not define these operations
S-09	No numeric performance thresholds	The model prioritizes by frequency without specifying figures [§6.1]
S-10	An image-storage port exists	The model refers to “external storage” [§1, §7.1]
S-11	The API contract is outside the model	It resides in api-contract.md, which was not provided [§12]
S-12	A sale is constructed and confirmed before being persisted	Interpretation of “to be confirmable” [§2.3]
S-13	A report for a range with no sales returns an empty result	The model does not define this
S-14	A concurrency conflict causes the losing write to fail, and the client retries	The model does not define the response
S-15	Success indicators are derived from invariants	The model does not define business metrics
10. Closing Review (Order 6): Verification Against the Model and 01–04
10.1 Is Every Section of the Model Covered?
Model	Where It Is Covered
§0 Naming	This document, §8
§1 Glossary	02-domain §1
§2 Entities and invariants	This document, §§4 and 6; 02-domain §3
§3 Physical model	This document, §8
§4 Constraints	This document, §6
§5 Foreign keys	This document, §6; 02-domain §5
§6 Access patterns and indexes	This document, §§5 and 8; 04 HU-07…HU-12, RNF-08
§7 Privacy and retention	This document, §8; 04 RNF-05, RNF-07
§8 No auditing	01-context §4 (out of scope)
§9 Seeding	This document, §8; 04 HU-02, HU-08
§10 Verification	04 RNF-09; this document, §10.5
§11 Gaps	02-domain §9; 03-product §8
§12 Sign-off and exclusions	01-context §4
§13 Technical debt	This document, §10.3
10.2 Does Every User Story Have an Aggregate and a Port?
HU	Aggregate(s)	Outbound Ports	Pattern
HU-01 Log in	User	User repository, hash	Q10
HU-02 Create seller accounts	User	User repository, hash	Q10
HU-03 Create product	Product, Category	Product repository, Category repository, images	Q5
HU-04 Modify product	Product, Category	Product repository, Category repository, images	Q2, Q5
HU-05 Restock	Product	Product repository	Q2
HU-06 Discontinue product	Product	Product repository, images	Q2
HU-07 Search products	Product	Product repository	Q1
HU-08 View categories	Category	Category repository	Q4, Q5
HU-09 Record sale	Sale, Product, User	Sale repository, Product repository	Q3
HU-10 View sale	Sale	Sale repository	Q6
HU-11 List sales	Sale	Sale repository	Q7
HU-12 Report	Read model	Report-reading port	Q9

Result: Q1–Q7, Q9, and Q10 have consumers. Q8 does not, intentionally [§6.1]. No user story requests anything excluded by the model.

10.3 Model Discrepancies (The Database Engine Takes Precedence; Verify Against §10 of the Model)
#	Contradiction	How It Is Handled Here
DM-1	Column count: “22” in §§3, 0, and 12 versus “21” in §10.1, the §8 anchor, and the xmin note	21 is the literal output from 2026-09-19. 22 equals 21 + deleted_at (T-09, §13 D-1). 22 is used as the current state
DM-2	product.category_name appears in the product table in §3, but DP-03 and §1 limit Product to name, price, stock, category, and image, while §§2.4, 5, and 11.1 place it in sale_item	It is treated as a sale_item column. This is considered a placement error in the model
DM-3	sale_item.category_name: “database-enforced (T-11)” in §2.4, but “pending (T-11)” in the sale_item table in §3	It is documented as database-enforced [§2.4], and the discrepancy is flagged. Verify using the query in §10.1
DM-4	deleted_at: database-enforced in §§2.2, 3, and 13 D-1, but “pending T-09” in §§6.2, 6.3, and 7.1	§13 (2026-09-20) takes precedence: database-enforced
DM-5	FK-3, sale_id NOT NULL, and the composite unique constraint are database-enforced in §§2.3, 2.4, 4 (table rows), 5, and 13 D-2; but §4's heading says “eight constraints,” §5 says “two currently exist,” §6.2 says “missing (T-13),” and the outputs in §10 omit them. Additionally, §6.2 assigns ownership of the composite constraint to T-13, while §4 assigns it to T-20	§13 D-2 takes precedence: database-enforced. The §10 outputs predate 2026-09-20 and must be rerun
DM-6	spec.md CA-06.1 (“one row per product”) contradicts the decision in §11.1	Pending owner decision [§11.1]. HU-12 is written according to the decision in §11.1
10.4 Technical Debt and Open Items Carried Forward by the Architecture
Open: T-05 (currency guard), T-11 (see DM-3), T-12 (sold_by_user_id and FK-4), T-13 (indexes), and T-20 (partial: the CHECK constraints for R-02, R-14, R-19, R-21, and R-23 are missing).
Gaps with an owner: H-2 (orphaned image binary) [§11].
Measured defect, not fixed: A-7/DP-01, the tie-breaking rule for product names in the report [§11.1].
10.5 What Remains Before the Architecture Can Be Considered Complete
Rerun the three queries from §10 of the model and reconcile DM-1…DM-5.
Obtain the owner's decision on DM-6.
Confirm assumptions S-03 (permissions), S-04 (transaction), and S-14 (concurrency conflict).






