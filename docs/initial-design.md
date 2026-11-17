# BudgetApp: Initial Design

## Summary
BudgetApp is a private budgeting app for a household. Users import transactions from bank and credit card CSV exports (Wells Fargo, Chase, Capital One), the app categorizes each one (gas, groceries, restaurants and so on), and a dashboard compares spending to the budget set for each category. Each month it produces a report on how the household did and which budgets need adjusting.

**Goals**
- Replace manual spreadsheet tracking for two people sharing finances.
- Categorize automatically using rules, then embedding similarity, then an LLM fallback, and learn from corrections.
- Keep financial data safe: runs locally, no bank credentials stored, no real data in the repo.
- Support many households, each with only its own data (Postgres row-level security).

## Initial ERD
```mermaid
erDiagram
    households ||--o{ household_members : has
    users ||--o{ household_members : joins
    households ||--o{ accounts : owns
    households ||--o{ categories : defines
    households ||--o{ category_rules : defines
    households ||--o{ budgets : sets
    accounts ||--o{ import_batches : receives
    accounts ||--o{ transactions : contains
    import_batches ||--o{ transactions : created
    categories ||--o{ transactions : classifies
    categories ||--o{ budgets : limits
    categories ||--o{ category_rules : targets
    households ||--o{ monthly_reports : receives

    users { uuid id PK
            text email
            text password_hash }
    households { uuid id PK
                 text name }
    household_members { uuid household_id FK
                        uuid user_id FK
                        text role }
    accounts { uuid id PK
               uuid household_id FK
               text institution
               text type }
    import_batches { uuid id PK
                     uuid account_id FK
                     timestamptz imported_at }
    transactions { uuid id PK
                   uuid account_id FK
                   uuid category_id FK
                   date posted_on
                   numeric amount
                   text raw_description
                   text merchant_norm
                   text dedupe_hash
                   vector embedding }
    categories { uuid id PK
                 uuid household_id FK
                 text name }
    category_rules { uuid id PK
                     uuid household_id FK
                     uuid category_id FK
                     text pattern }
    budgets { uuid household_id FK
              uuid category_id FK
              date month
              numeric amount }
    monthly_reports { uuid id PK
                      uuid household_id FK
                      date month
                      jsonb summary }
```

## Rough system design
```mermaid
flowchart LR
    CSV[Bank CSV exports] --> IMP[Importers<br/>Wells Fargo / Chase / Capital One]
    PLAID[(Plaid, later)] -.-> IMP
    IMP --> API[FastAPI backend<br/>auth + households]
    API --> CAT[Categorizer]
    CAT --> R[1. Rules]
    CAT --> V[2. Embeddings<br/>pgvector]
    CAT --> L[3. Claude fallback]
    API <--> DB[(Postgres + pgvector<br/>row-level security)]
    UI[React dashboard] <--> API
    API --> REP[Monthly report]
```
Everything runs in Docker Compose on one local machine. It is reachable only on localhost, or over a private VPN such as Tailscale for the second user.

## Initial daily goals
| Date | Goal |
|---|---|
| Nov 18-19 | Finish this design, pitch, make repo public |
| Nov 20-22 | Docker Compose, schema, auth, households, RLS, seed data |
| Nov 23-26 | CSV importers, dedupe, transactions page |
| Nov 27-29 | Categorizer, budgets, dashboard. Working demo |
| Nov 30 | Present in class |
| Dec 1-5 | Feedback, monthly report, tests |
| Dec 6-10 | Demo GIF, final report, submit |
