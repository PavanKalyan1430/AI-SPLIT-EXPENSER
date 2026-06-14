# Split Expenser — Enterprise Shared Expense Management & CSV Anomaly Resolution SaaS
<img width="897" height="262" alt="Split Expenser Banner" src="https://github.com/user-attachments/assets/1223d84e-eef9-4905-8e21-ae81424af730" />

Split Expenser is a high-performance, multi-tenant expense-splitting and debt-reconciliation platform designed for flatmates, travel groups, and shared business cost centers. It combines an interactive **staging-area CSV anomaly validator wizard**, timezone-aware **active timeline rosters**, and a **greedy min-flow debt-simplification network engine** (Debt Simplification Pathway).

---

## 🚀 Key Features

- **Staged CSV Ingestion Pipeline**: Messy CSV files are not directly written to the ledger. Instead, they are parsed into isolated staging tables (`import_records` and `anomaly_records`), scanned for data anomalies, and returned to a React review wizard.
- **Dynamic Anomaly Resolution Engine**: Color-coded row warning states (MISSING_PAYER, UNRESOLVED_PAYER, FOREIGN_CURRENCY, CSV_DUPLICATE_WARNING, DB_DUPLICATE_WARNING, PERCENTAGE_SUM_MISMATCH, NEGATIVE_AMOUNT_REFUND, INVALID_DATE_FORMAT, FUTURE_DATE, DATE_AMBIGUITY_WARNING, INACTIVE_PAYER) with on-the-spot mapping select dropdowns.
- **Interactive UI Resolution Wizard**: Built with vanilla CSS glassmorphism, dynamic fade animations, loading skeletons, and inline error-correction controls that let the user fix data discrepancies without leaving the application dashboard.
- **Timezone-Aware Active Timeline Rosters**: Memberships track `joined_at` and `left_at` offsets. When splitting expenses, the balance engine automatically respects these boundaries so members are only charged for transactions occurring while they were active group members.
- **Greedy Min-Flow Debt Simplification**: A network-flow balance engine that reduces the total transaction count from $O(N^2)$ to $O(N)$ using sorting and greedy matching algorithms.
- **Robust PostgreSQL Referential Schema**: Leverages strict foreign key constraints, `ON DELETE CASCADE` triggers on groups, and `ON DELETE RESTRICT` rules to prevent deleting active transaction-linked users.

---

## 🛠️ Tech Stack

### Backend
- **Framework**: FastAPI (Python 3.11+)
- **Database**: PostgreSQL with SQLAlchemy ORM (Transactional commits and cascading triggers)
- **Security**: BCrypt password hashing & JWT bearer token authentication
- **Tests**: Pytest suite (10 unit & integration test cases)

### Frontend
- **Framework**: React 18 (Vite boilerplate)
- **Styling**: Vanilla CSS (Harmonious HSL palettes, dark-mode styling, glassmorphism, micro-animations)
- **State Management**: React State & Context API
- **Icons**: Lucide React

---

## 🧬 Project Directory Structure

```
Split Expenser/
├── backend/
│   ├── app/
│   │   ├── core/
│   │   │   ├── config.py         # Environment configuration (JWT secrets, PostgreSQL connection)
│   │   │   ├── database.py       # SQLAlchemy engine & session factory
│   │   │   └── security.py       # Password hashing & JWT generation helpers
│   │   ├── models/               # SQLAlchemy ORM Models
│   │   │   ├── user.py           # User definitions and hashed credentials
│   │   │   ├── group.py          # Groups & GroupMembership (timeline limits)
│   │   │   ├── expense.py        # Expense & ExpenseSplit details
│   │   │   ├── settlement.py     # Reconciliation transactions
│   │   │   └── imports.py        # ImportJob, ImportRecord, & AnomalyRecord staging models
│   │   ├── schemas/              # Pydantic schemas (Request / Response validation)
│   │   │   ├── auth.py, group.py, expense.py, settlement.py, imports.py
│   │   ├── services/
│   │   │   ├── balance_engine.py # Net balances and min-flow debt simplifier
│   │   │   └── anomalies.py      # Core rules and anomaly scan engine
│   │   ├── routers/              # API Endpoints
│   │   │   ├── auth.py           # Login / Registration
│   │   │   ├── groups.py         # Group CRUD & membership roster invites
│   │   │   ├── expenses.py       # Direct expense logging & split distributions
│   │   │   ├── settlements.py    # Direct payments logging
│   │   │   ├── balances.py       # simplified settlement paths & net balances
│   │   │   └── imports.py        # CSV upload staging, anomaly reports, & resolution approvals
│   │   └── main.py               # Uvicorn entrypoint and CORS middleware configuration
│   ├── tests/                    # Backend Integration & Unit Test Suite
│   │   ├── conftest.py           # DB session mocks and Client test configurations
│   │   ├── test_anomalies.py     # Anomaly validation rule assertions
│   │   ├── test_balances.py      # Balance calculation tests (timeline aware)
│   │   └── test_imports.py       # Staged upload and resolution approval tests
│   └── seed.py                   # Seeding script for historical roommate timelines
├── frontend/                     # React 18 + Vite Dashboard SPA
│   ├── src/
│   │   ├── App.jsx               # Main SPA Layout, wizards, and modals
│   │   ├── index.css             # Vanilla CSS design system, typography, and scrollbars
│   │   └── main.jsx              # React DOM mounting
└── README.md                     # Setup instructions & project reference guide
```

---

## 🧠 Core Workflows & Engines

### 1. Staged CSV Import Pipeline

Split Expenser separates the ingestion phase into a staged preview layer. This prevents corrupt data (like missing emails, negative amount typos, or unrecognized names) from reaching your database ledger.

```mermaid
flowchart TD
    A["👤 Upload CSV File"] --> B["⚙️ Ingestion Router<br/>(imports.py)"]
    B --> C["🗄️ Save Staged Raw Data<br/>(ImportRecord)"]
    C --> D["🔍 Run Anomaly Engine<br/>(anomalies.py)"]
    D --> E{"Anomalies Found?"}
    E -->|"Yes"| F["⚠️ Status: PENDING_REVIEW<br/>Return Anomaly Log"]
    E -->|"No"| G["✅ Status: VALIDATED<br/>Return Success Code"]
    
    F --> H["🖥️ React Resolution Wizard<br/>Display Color-Coded Warnings"]
    H --> I{"User Actions"}
    I -->|"Map Payer to Member"| J["💾 Save Correction Mapping"]
    I -->|"Skip Row"| K["❌ Mark Status: SKIPPED"]
    
    J --> L["🚀 Approve Resolution<br/>(/imports/{id}/approve)"]
    K --> L
    G --> L
    
    L --> M{"Check Validation Constraints"}
    M -->|"Fails (e.g. Empty Payer/Receiver)"| N["🛑 Raise HTTP 400 Bad Request"]
    M -->|"Passes"| O["💸 Commit to Ledger<br/>Generate Expenses & Settlements"]
    
    O --> P["📊 Update Net Balances &<br/>Simplify Pathways"]
```

---

### 2. Active Timeline Roster Engine

Group memberships in Split Expenser are not binary; they are **historical intervals**. The `GroupMembership` table tracks `joined_at` and `left_at` timestamps.

#### Balance Obligation Formula:
The net balance $B(i)$ for user $i$ in group $G$ is computed as:

$$B(i) = \sum_{e \in E_G, P(e)=i} A(e) - \sum_{e \in E_G} O(e, i) + \sum_{s \in S_G, F(s)=i} Set(s) - \sum_{s \in S_G, T(s)=i} Set(s)$$

Where:
- $E_G$ represents all expenses logged in group $G$.
- $P(e) = i$ matches if user $i$ paid for expense $e$.
- $A(e)$ is the total amount of expense $e$.
- $O(e, i)$ is the obligation (split share) owed by user $i$ for expense $e$.
- $S_G$ represents all recorded direct settlements.
- $F(s) = i$ matches if user $i$ sent settlement $s$.
- $T(s) = i$ matches if user $i$ received settlement $s$.
- $Set(s)$ is the monetary value of settlement $s$.

*Note: If an expense $e$ occurs at time $T$, and user $i$ was not an active member on $T$ (i.e. $T < joined\_at$ or $T \ge left\_at$), then $O(e, i) = 0$.*

---

### 3. Greedy Min-Flow Debt Simplification Algorithm

To minimize transaction friction, net balances are resolved using a **Greedy Min-Flow Solver** ($O(N \log N)$ sorting complexity). Rather than making everyone settle individually, the platform matches largest debtors with largest creditors.

#### Algorithm Steps:
1. Divide users into two sets: **Debtors** (net balance $< 0$) and **Creditors** (net balance $> 0$).
2. Sort Debtors ascending (largest debt first) and Creditors descending (largest credit first).
3. Greedily match the top debtor $D$ and top creditor $C$:
   - $TransferAmount = \min(|Balance_D|, Balance_C)$
   - Log transaction: $D$ pays $C$ the $TransferAmount$.
   - Adjust both balances: $Balance_D \leftarrow Balance_D + TransferAmount$, $Balance_C \leftarrow Balance_C - TransferAmount$.
   - Remove any member whose balance becomes $0$.
4. Repeat until all balances are fully reconciled.

#### Python Implementation:
```python
def simplify_debts(balances: Dict[UUID, int]) -> List[Dict]:
    debtors = [[bal, uid] for uid, bal in balances.items() if bal < 0]
    creditors = [[bal, uid] for uid, bal in balances.items() if bal > 0]

    # Sort largest amounts to the front
    debtors.sort(key=lambda x: x[0])
    creditors.sort(key=lambda x: x[0], reverse=True)

    transactions = []
    d_idx = c_idx = 0

    while d_idx < len(debtors) and c_idx < len(creditors):
        debtor_bal, debtor_id = debtors[d_idx]
        creditor_bal, creditor_id = creditors[c_idx]

        amount = min(-debtor_bal, creditor_bal)
        if amount > 0:
            transactions.append({
                "from_user_id": debtor_id,
                "to_user_id": creditor_id,
                "amount": amount
            })
            debtors[d_idx][0] += amount
            creditors[c_idx][0] -= amount

        if abs(debtors[d_idx][0]) < 1:
            d_idx += 1
        if creditors[c_idx][0] < 1:
            c_idx += 1

    return transactions
```

---

## 🗄️ Relational Constraints & Database Schema

The database architecture employs **cascading constraints** on group models to ensure deletions leave no orphaned records:
- **`ON DELETE CASCADE`** on `group_id` fields ensures that deleting a group automatically purges memberships, expenses, splits, and settlements.
- **`ON DELETE RESTRICT`** on `from_user_id`, `to_user_id`, and `paid_by_user_id` ensures that active users with transactions cannot be deleted.

---

## 🌐 API Endpoint Catalog

| Method | Endpoint | Description | Payload/Query | Response Shape |
| :--- | :--- | :--- | :--- | :--- |
| **POST** | `/api/auth/register` | Register new user account. | `email, password, name` | `{id, email, name}` |
| **POST** | `/api/auth/login` | Log in and retrieve JWT. | `email, password` | `{access_token, token_type}` |
| **GET** | `/api/auth/me` | Fetch active user profile details. | *Bearer Token* | `{id, email, name}` |
| **POST** | `/api/groups` | Create new shared expense group. | `name, description` | `{id, name, description, created_at}` |
| **POST** | `/api/groups/{id}/members` | Invite member to group. | `email` | `{detail: "User added"}` |
| **DELETE** | `/api/groups/{id}/members/{u_id}` | Set left_at date for group member. | *Path parameters* | `{detail: "Member deactivated"}` |
| **POST** | `/api/groups/{id}/expenses` | Log an expense & custom splits. | `amount, description, splits[]` | `{id, description, amount, splits[]}` |
| **POST** | `/api/groups/{id}/settlements` | Log a direct payment settlement. | `from_user_id, to_user_id, amount` | `{id, amount, settlement_date}` |
| **GET** | `/api/groups/{id}/balances` | Get net balances & simplified paths. | *Path parameters* | `{net_balances, simplified_settlements}` |
| **POST** | `/api/groups/{id}/imports/upload` | Stage CSV file for analysis. | *Multipart file upload* | `{id, status, records: [raw_data, anomalies]}` |
| **POST** | `/api/imports/{id}/approve` | Confirm and ingest CSV records. | `resolutions: [record_id, action, corrected_data]` | `{imported_expenses, imported_settlements}` |

---

## 🚀 Setup & Setup Instructions

### Prerequisites
1. Install Python 3.11+ and Node.js (v18+).
2. Configure local PostgreSQL server.

### 1. Database Configuration
Create a database named `split_expenser` with standard credentials (update in `backend/.env`):
```ini
DATABASE_URL=postgresql://postgres:pavan@localhost:5432/split_expenser
SECRET_KEY=yoursecretjwtkeyhere
ALGORITHM=HS256
```

### 2. Backend Installation & Dev Server
```bash
cd backend
pip install -r requirements.txt
py seed.py    # Generates database schema & historical active group members
py -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

### 3. Frontend Installation & Dev Server
```bash
cd frontend
npm install
npm run dev
```

### 4. Running Automated Tests
```bash
cd backend
py -m pytest
```

---

## 🧠 System Design & Interview Prep Q&A

### Q1: Why choose PostgreSQL over SQLite for this application?
- **Concurrency & Locking**: SQLite locks the database file on writes, causing concurrent import approvals to block or timeout. PostgreSQL supports row-level locking.
- **Relational Constraints**: PostgreSQL strictly enforces Foreign Key restrictions. SQLite allows disabling them unless configured manually on every session start.
- **Timezone Inconsistencies**: SQLite stores datetimes as text and strips timezone offsets on return. PostgreSQL natively stores timezone-aware timestamps with `DateTime(timezone=True)`, maintaining UTC offsets.

### Q2: What are the trade-offs of the Greedy Min-Flow algorithm for debt simplification?
- **Trade-off**: Min-Flow guarantees the **minimum number of cash transfers** (the size of the transactions set is at most $N-1$). However, it does not guarantee the **minimum sum of cash moved**. In rare financial ledger scenarios, we might prioritize moving fewer total rupees over reducing transaction count. For consumer apps (like Splitwise), reducing transaction count is the desired UX.

### Q3: How do you handle timezone comparison conflicts?
- **Conflict**: CSV data lists naive dates (e.g. `2026-03-29`). PostgreSQL query outputs timezone-aware timestamps (`2026-03-29 12:00:00+00:00`). Comparing these direct objects raises `TypeError`.
- **Solution**: We normalize all datetimes during comparisons by checking for `tzinfo` and replacing/stripping timezone offsets inside the active duration checking routines:
  ```python
  exp_date_naive = expense_date.replace(tzinfo=None) if expense_date.tzinfo else expense_date
  joined_naive = joined.replace(tzinfo=None) if joined.tzinfo else joined
  ```

### Q4: Why implement a Staged Import pipeline instead of importing directly?
- **Atomicity & Cleanliness**: Direct imports must run inside a single transaction database transaction. If row 34 fails verification, the previous 33 rows must rollback, causing frustration. If we don't rollback, database contamination occurs.
- **Staging Advantage**: Staging saves parsing details into the database as raw JSON text first. The client can edit row data in multiple sessions, resolve anomalies in a visual review wizard, and click approve when satisfied, ensuring only clean data is committed.

### Q5: How do you prevent rounding discrepancies when dividing odd split amounts?
- **Problem**: Splitting $100$ cents among $3$ users results in $33.333...$ cents. Simply rounding to $33$ cents yields $33 \times 3 = 99$, leaving a $1$ cent discrepancy.
- **Solution**: We divide using integer division (`amount // n`) and compute the remainder (`amount % n`). We then distribute the remainder cents one-by-one to the first $R$ users:
  ```python
  base = amount_cents // n
  remainder = amount_cents % n
  calculated_amounts = [base + (1 if i < remainder else 0) for i in range(n)]
  ```
  This guarantees that $\sum calculated\_amounts = total\_amount$ exactly.
