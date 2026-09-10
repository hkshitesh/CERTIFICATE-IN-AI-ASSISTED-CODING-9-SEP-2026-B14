# ExpenseFlow Architecture

PoC expense submission and approval API. One user journey: submit an expense,
convert it to base currency (INR), approve or reject it.

## 1. Schema — `expenses` table (SQLite)

```sql
CREATE TABLE expenses (
    id                 INTEGER PRIMARY KEY AUTOINCREMENT,
    description        TEXT    NOT NULL,
    amount_minor       INTEGER NOT NULL CHECK (amount_minor > 0),
    currency           TEXT    NOT NULL CHECK (length(currency) = 3),
    fx_rate_micros     INTEGER NOT NULL CHECK (fx_rate_micros > 0),
    amount_base_minor  INTEGER NOT NULL CHECK (amount_base_minor > 0),
    status             TEXT    NOT NULL DEFAULT 'pending'
                               CHECK (status IN ('pending', 'approved', 'rejected')),
    submitted_by       TEXT    NOT NULL,
    created_at         TEXT    NOT NULL DEFAULT (CURRENT_TIMESTAMP),
    decided_at         TEXT
);
```

| Column | Type | Why |
|---|---|---|
| `id` | `INTEGER PK AUTOINCREMENT` | Stable identifier for lookups and the approve/reject actions. |
| `description` | `TEXT NOT NULL` | What the expense is for; required for any reviewable claim. |
| `amount_minor` | `INTEGER NOT NULL` | Original submitted amount in minor units (paise/cents). Integer per CLAUDE.md's money rule — never float. |
| `currency` | `TEXT(3) NOT NULL` | ISO 4217 code of the currency submitted in. Needed to know what to convert from. |
| `fx_rate_micros` | `INTEGER NOT NULL` | The currency→INR rate used at submission time, stored as an integer scaled by 1,000,000 (e.g. rate 0.011234 → 11234). Storing the rate itself as an integer (not a float) keeps the exact multiplier reproducible and auditable — if the external FX rate later changes, historical rows don't silently drift. |
| `amount_base_minor` | `INTEGER NOT NULL` | The amount converted to INR paise, computed once at submission and persisted. Never recomputed on read, so a later rate change can't alter an already-submitted expense's value. |
| `status` | `TEXT NOT NULL, CHECK enum` | Workflow state: `pending` → `approved`/`rejected`. `CHECK` constraint keeps bad states out at the DB layer, not just the API layer. |
| `submitted_by` | `TEXT NOT NULL` | Who submitted the expense — minimal audit trail; the brief has no user/auth model, so this is a plain string, not a foreign key. |
| `created_at` | `TEXT NOT NULL` | ISO-8601 submission timestamp. |
| `decided_at` | `TEXT NULL` | ISO-8601 timestamp of approval/rejection; `NULL` while pending. |

## 2. Endpoints

| Method | Path | Request body | Response |
|---|---|---|---|
| `POST` | `/expenses` | `{description: str, amount_minor: int (>0), currency: str (3-letter ISO), submitted_by: str}` | `201` → full expense object (`id`, `status="pending"`, `amount_base_minor`, `fx_rate_micros`, `created_at`, ...). `502` if the FX call fails. |
| `GET` | `/expenses` | — (optional `?status=pending\|approved\|rejected` query filter) | `200` → list of expense objects |
| `GET` | `/expenses/{expense_id}` | — | `200` → expense object, `404` if not found |
| `POST` | `/expenses/{expense_id}/approve` | — | `200` → updated expense object; `404` if not found; `409` if not currently `pending` |
| `POST` | `/expenses/{expense_id}/reject` | — | `200` → updated expense object; `404` if not found; `409` if not currently `pending` |

No endpoints beyond these five — matches the single journey in the brief (submit → convert → approve/reject) plus the read endpoints needed to view state before deciding.

## 3. File layout

Exactly the five files CLAUDE.md prescribes — no new modules added, so the FX call/conversion logic lives inside `routes.py` rather than a separate `fx.py`.

- **`app/main.py`** — FastAPI app instance; loads `.env` via `python-dotenv`; calls `Base.metadata.create_all()` on startup; includes the router from `routes.py`.
- **`app/db.py`** — SQLAlchemy engine (`sqlite:///expenseflow.db`, path overridable via `DATABASE_URL` env var), `SessionLocal` sessionmaker, declarative `Base`, and a `get_db()` dependency generator.
- **`app/models.py`** — the `Expense` ORM class mapped to the `expenses` table, with the columns and `CHECK` constraints above.
- **`app/schemas.py`** — pydantic v2 models: `ExpenseCreate` (request body for `POST /expenses`), `ExpenseRead` (response model, `from_attributes=True`), `ExpenseStatus` (str enum: `pending`/`approved`/`rejected`).
- **`app/routes.py`** — the `APIRouter` with all five endpoints; also owns the FX client call (`httpx`, reading `FX_API_URL` / `FX_API_KEY` from env — provider left pluggable, not tied to a specific vendor) and the amount-conversion helper. Each endpoint takes `db: Session = Depends(get_db)`.

## 4. Edge cases

**1. External FX API failure or timeout.**
The `httpx` call is wrapped with an explicit timeout. On timeout, non-200, or malformed response, `POST /expenses` fails fast with `502`/`503` *before* any DB write — there is no partial or pending row without a resolved rate. Nothing is committed unless the conversion succeeded.

**2. Rounding / precision drift.**
The FX rate is stored as an integer (`fx_rate_micros`, rate × 1,000,000) instead of a float, so the exact multiplier used is preserved for audit rather than approximated. `amount_base_minor` is computed with integer arithmetic and a fixed rounding rule (round-half-up to the nearest paisa) once, at submission time, and never recalculated — so a later change in the market rate can't retroactively change the value of an already-submitted expense. Submissions already in INR skip the external call entirely (rate is implicitly 1,000,000).

**3. Double-decision races / invalid state transitions.**
Approve and reject are only valid from `pending`. Implemented as a single conditional update — `UPDATE expenses SET status=..., decided_at=... WHERE id=:id AND status='pending'` — rather than read-then-write, so two concurrent approve/reject calls on the same expense can't both succeed. Whichever request loses the race affects 0 rows and gets `409 Conflict`; the DB-level `CHECK` on `status` is a second backstop against an invalid state ever being written.

---
*Two open decisions were resolved by default while drafting this (not explicitly confirmed): FX provider is left pluggable via env vars rather than pinned to a vendor, and FX logic stays inline in `routes.py` to honor CLAUDE.md's exact five-file layout. Flag if either should change.*
