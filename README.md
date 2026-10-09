# test-3239188-simple-stock-flow-doc

> **SDD Challenge · Ficha ADSO 3239188**
> Delivery: **today, October 8, 2026, at 10:50 p.m. (Colombia time)**. Countdown: https://claude.ai/artifact/3ja3TMwGnprBV6QCcyAruT

This challenge **is not done in the main repository of your project**. It is done in a **fork of this repository**.

## What needs to be done

The only input is the *Simple Stock Flow* data model: [`spec/data-model.md`](spec/data-model.md).
From it, the system documentation is reconstructed **backwards**, from the architecture to the context:

| Order | Folder | What is produced from the data model |
|---|---|---|
| 1 | `05-architecture/` | System style and parts implied by the model (aggregates, ports, where each rule lives) |
| 2 | `04-requirements/` | User stories and non-functional requirements necessitated by the model |
| 3 | `03-product/` | Problem it solves and product vision |
| 4 | `02-domain/` | Domain entities, rules, and events, with their glossary |
| 5 | `01-context/` | General description and scope (what is built and what is not) |
| 6 | `05-architecture/` (closing) | Return to the architecture and verify that it matches everything above and the model |

The `06-data/` folder **is not written**: it is the model that is provided to you.

## Rules

1. **Fork** this repository to your account or your team's account.
2. Work in your fork. One folder per document, with the names from the table above.
3. Every statement must be traceable to the data model (cite the section, for example "§2.3" or "FK-2").
   If something does not come from the model, mark it as an **assumption**.
4. What counts is the **last commit before 10:50 p.m.** Anything that arrives after will not be reviewed.
5. Performance is evaluated with SDD: how the specification is read, interpreted, and applied. Not the volume of text.

## What week each team is in

This was calculated by comparing each `-docs` repository with the governance template. The weeks are those from
`00-sdd-guide.md`: week 1 context and domain (01-02), week 2 product and requirements (03-04),
weeks 2-3 architecture and data (05-06), weeks 3-4 detailed design (07 onwards).

| Team (project) | Current week | What they already have |
|---|---|---|
| lexia | 3-4 | 01 to 06 and 07-api |
| fixgo | 2-3 | 01 to 06 |
| smart-technical-service-to-professional | 2-3 | 01 to 06 |
| belleza-ya | 2-3 | 01 to 06 |
| distrilink | 2 | 01 to 04; missing 05 |
| interemprendedores | 2 | 01 to 04; missing 05 |
| construction-project-management-system | 1 | only 01-context |
| residential-complex | not started | no changes to the template |
| huila-travel-expedition | not started | no changes to the template |
| huila-travel-services | not started | no changes to the template |
| fastbill-manager | not started | no changes to the template |

This is an estimate based on modified files in `main`. If your team worked on another branch, let us know.

## Note on the data model

`spec/data-model.md` links to other documents from the original spec (`constitution.md`, `plan.md`, `adr/`, etc.).
**They are not provided**: those links will not open. Everything you need is in the model.
