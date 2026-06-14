# Split Expenser — Enterprise Shared Expense Management & CSV Anomaly Ingestion SaaS

![Split Expenser Banner](assets/split_expenser_banner.png)

Split Expenser is a premium multi-tenant SaaS expense-splitting and debt-reconciliation platform designed for roommates, group trips, and joint business costs. It features a staged **CSV Ingestion Anomaly Resolution Wizard**, a **Date-Aware Historical Roster Engine**, and a **Greedy Min-Flow Debt-Simplification Network Solver**.

---

## 🚀 Key Platform Features

- **Staged CSV Ingestion Review Queue**: Raw CSV files are placed in an intermediate database review queue (`import_records` and `anomaly_records`) instead of directly contaminating the group ledger. The client corrects anomalies in the UI before committing data.
- **Dynamic Anomaly Engine**: Scans every uploaded row against 10 distinct mathematical, syntactic, and relational anomaly rules.
- **Interactive UI Correction Wizard**: Color-coded row warning states (amber for warnings, red for critical errors, green for valid rows) with on-the-spot drop-down select boxes for mapping unresolved or missing payers.
- **Historical active timelines**: Member records contain `joined_at` and `left_at` limits. The balance engine automatically checks transaction dates against these intervals, so users are only split into expenses logged while they were active group members.
- **Auto-Provisioning & Retroactive Joining**: Unregistered users found in the CSV are automatically provisioned with mock profiles (e.g. `dev@split.local`) and retroactively added to the group roster so transaction calculations compile smoothly.
- **Greedy Min-Flow Network Solver**: Resolves outstanding group obligations, reducing cash transfer transactions from $O(N^2)$ to $O(N)$ matching routes.
- **PostgreSQL Relational Schema**: Enforces strict referential constraints, cascading delete rules on groups, and delete restriction rules on active users to ensure complete data integrity.

---

## 🛠️ Technology Stack

### Backend
- **Core Framework**: FastAPI (Python 3.11+)
- **ORM**: SQLAlchemy (Declarative base mapping with transaction scopes)
- **Database**: PostgreSQL (for multi-user concurrency and transactional safety)
- **Authentication**: JWT token authorization (HS256) & BCrypt password hashing
- **Test Harness**: Pytest suite (10 unit & integration test cases)

### Frontend
- **Framework**: React 18 (Vite SPA)
- **Styling**: Vanilla CSS (Modern glassmorphic palette, custom scrollbars, animations)
- **Icons**: Lucide React

---

## 🧬 Project Architecture & Layout

```
Split Expenser/
├── backend/
│   ├── app/
│   │   ├── core/
│   │   │   ├── config.py         # App environment variables (JWT keys, DB URLs)
│   │   │   ├── database.py       # SessionLocal factory & Engine configuration
│   │   │   └── security.py       # Password hashing & JWT helper utilities
│   │   ├── models/               # Declarative SQLAlchemy ORM Models
│   │   │   ├── user.py           # User schema
│   │   │   ├── group.py          # Group & GroupMembership schemas (timeline bounds)
│   │   │   ├── expense.py        # Expense & ExpenseSplit schemas
│   │   │   ├── settlement.py     # Direct payment schema
│   │   │   └── imports.py        # ImportJob, ImportRecord, & AnomalyRecord staging schemas
│   │   ├── schemas/              # Pydantic validation structures
│   │   │   ├── auth.py, group.py, expense.py, settlement.py, imports.py
│   │   ├── services/
│   │   │   ├── balance_engine.py # Net balance calculator & min-flow graph optimizer
│   │   │   └── anomalies.py      # Core anomaly rules engine
│   │   ├── routers/              # Controllers/End-point routers
│   │   │   ├── auth.py           # Authentication services
│   │   │   ├── groups.py         # Group CRUD & member roster management
│   │   │   ├── expenses.py       # Direct expense creations
│   │   │   ├── settlements.py    # Direct settlement recordings
│   │   │   ├── balances.py       # Group balance reports & simplified routes
│   │   │   └── imports.py        # CSV uploading, anomaly scans, & resolution approvals
│   │   └── main.py               # FastAPI entrypoint, CORS setup, and route aggregation
│   ├── tests/                    # Backend automated tests
│   │   ├── conftest.py           # Testing clients, mock databases, and fixture scopes
│   │   ├── test_anomalies.py     # Verification of anomaly engine rules
│   │   ├── test_balances.py      # Verification of timeline calculations
│   │   └── test_imports.py       # CSV upload & wizard correction integration tests
│   └── seed.py                   # Seeding script for roommate timelines
├── frontend/                     # React + Vite Client Application
│   ├── src/
│   │   ├── App.jsx               # Single Page Application core (Ledger, dashboard, wizards)
│   │   ├── index.css             # Premium CSS theme (Harmonious glass, HSL variables)
│   │   └── main.jsx              # React DOM mounting
└── assets/
    └── split_expenser_banner.png # Generated app header artwork
```

---

## ⚙️ Core Engines & Logical Design

### 1. Staged CSV Ingestion Architecture

```mermaid
flowchart TD
    A["👤 Select CSV File"] --> B["📤 POST /api/groups/{id}/imports/upload"]
    B --> C["📦 Staging Area<br/>Create ImportRecord"]
    C --> D["🔍 Scan: Anomaly Engine<br/>(anomalies.py)"]
    
    D --> E{"Anomalies Found?"}
    E -->|"Yes (Severity: WARNING/CRITICAL)"| F["⚠️ Status: PENDING_REVIEW<br/>Return Anomaly Metadata"]
    E -->|"No (Valid Rows)"| G["✅ Status: VALIDATED<br/>Return Ready State"]
    
    F --> H["🖥️ UI Wizard Review Pane"]
    H --> I{"User Actions"}
    I -->|"Map Payer Dropdown"| J["💾 Save resolved paid_by_user_id"]
    I -->|"Toggle Skip Row"| K["❌ Mark Status: SKIPPED"]
    
    J --> L["📤 POST /api/imports/{id}/approve"]
    K --> L
    G --> L
    
    L --> M{"Validate Mappings"}
    M -->|"Missing Payer / Receiver"| N["🛑 HTTP 400 Bad Request"]
    M -->|"Valid"| O["💸 Commit transactions to ledger"]
```

---

### 2. Active Timeline Roster Logic

Memberships track transaction liability limits using date ranges:
- **Calculation Window**: A member is only included in an expense's split calculations if:
  $$\text{Expense Date} \ge \text{Membership joined\_at} \quad \text{AND} \quad (\text{Membership left\_at} \text{ is NULL} \quad \text{OR} \quad \text{Expense Date} < \text{Membership left\_at})$$
- **Mathematical Net Balance Equation**: The net balance $B(i)$ for user $i$ in group $G$ combines direct expenses, split shares, and direct payments:

$$B(i) = \sum_{e \in E_G, P(e)=i} A(e) - \sum_{e \in E_G} O(e, i) + \sum_{s \in S_G, F(s)=i} Set(s) - \sum_{s \in S_G, T(s)=i} Set(s)$$

Where:
- $E_G$: Expenses logged in group $G$.
- $P(e) = i$: Indicates user $i$ is the payer of expense $e$.
- $A(e)$: Total monetary value of expense $e$.
- $O(e, i)$: Split obligation share calculated in cents for user $i$.
- $S_G$: Reconciling direct settlements.
- $F(s) = i$: User $i$ is the sender of settlement $s$.
- $T(s) = i$: User $i$ is the receiver of settlement $s$.
- $Set(s)$: Value of settlement $s$.

---

### 3. Greedy Min-Flow Debt Simplification Algorithm

Split Expenser simplifies network obligations to minimize total transaction count. It runs a greedy matching algorithm:
1. **Balance Segregation**: Split group members into Debtors (net balance $< 0$) and Creditors (net balance $> 0$).
2. **Sort**: Sort debtors ascending (largest debt first) and creditors descending (largest credit first).
3. **Match**: Reconcile top debtor $D$ and top creditor $C$:
   - $TransferAmount = \min(|Balance_D|, Balance_C)$
   - $D$ pays $C$ the $TransferAmount$.
   - Update balances. Remove members whose balance reaches zero.
4. **Repeat** until all debts are fully cleared.

---

## 🔍 Detailed Ingestion Anomaly Rules

The engine runs each row through the following checks:
- **`MISSING_PAYER`** (Critical): The `paid_by` column is empty.
- **`UNRESOLVED_PAYER`** (Critical): Payer's name is not found in the group roster.
- **`INACTIVE_PAYER`** (Warning): The payer was inactive on the transaction date.
- **`CSV_DUPLICATE_WARNING`** (Warning): The row is identical to another row in the same CSV upload.
- **`DB_DUPLICATE_WARNING`** (Warning): The row matches an expense already logged in the database.
- **`NEGATIVE_AMOUNT_REFUND`** (Warning): The amount is negative, indicating a group refund.
- **`INVALID_DATE_FORMAT`** (Critical): The transaction date format is unrecognized.
- **`FUTURE_DATE`** (Warning): The transaction date is in the future.
- **`DATE_AMBIGUITY_WARNING`** (Info): The date is ambiguous (e.g. `04-05-2026` could be Day/Month or Month/Day).
- **`PERCENTAGE_SUM_MISMATCH`** (Critical): Split percentages do not sum to 100%.
- **`EXACT_SPLIT_SUM_MISMATCH`** (Critical): Exact split amounts do not sum to the total expense value.

---

## 🗄️ Database Table Structures

```
  +-------------------------------------------------------------+
  |                          users                              |
  +-------------------------------------------------------------+
  | id (UUID, PK) | name | email (UK) | password_hash | created |
  +------------------+------------------+-----------------------+
                     |                  |
                     | 1                | 1
                     |                  |
                    inf                inf
  +-----------------------------------+ +------------------------------------------+
  |           group_memberships       | |                expenses                  |
  +-----------------------------------+ +------------------------------------------+
  | id (UUID, PK)                     | | id (UUID, PK)                            |
  | group_id (UUID, FK -> groups)     | | group_id (UUID, FK -> groups)            |
  | user_id (UUID, FK -> users)       | | paid_by_user_id (UUID, FK -> users)      |
  | joined_at (TIMESTAMP)             | | amount (INTEGER, cents)                  |
  | left_at (TIMESTAMP, NULLABLE)     | | description | split_type | currency      |
  +-----------------------------------+ | expense_date | created_at                |
                                        +-------------------+----------------------+
                                                            |
                                                            | 1
                                                            |
                                                           inf
                                        +------------------------------------------+
                                        |             expense_splits               |
                                        +------------------------------------------+
                                        | id (UUID, PK)                            |
                                        | expense_id (UUID, FK -> expenses)        |
                                        | user_id (UUID, FK -> users)              |
                                        | share_value (DECIMAL)                    |
                                        | calculated_amount (INTEGER, cents)       |
                                        +------------------------------------------+
```

---

## 🌐 API Route Specifications

| Method | Endpoint | Description | Request Body / Query | Response Structure |
| :--- | :--- | :--- | :--- | :--- |
| **POST** | `/api/auth/register` | Register new user. | `{"email", "password", "name"}` | `{"id", "email", "name"}` |
| **POST** | `/api/auth/login` | Log in and get JWT. | `{"email", "password"}` | `{"access_token", "token_type"}` |
| **GET** | `/api/auth/me` | Fetch logged-in user profile.| *Bearer Token* | `{"id", "email", "name"}` |
| **POST** | `/api/groups` | Create new expense group. | `{"name", "description"}` | `{"id", "name", "created_at"}` |
| **POST** | `/api/groups/{id}/members` | Add user to group roster. | `{"email"}` | `{"detail": "User added"}` |
| **DELETE** | `/api/groups/{id}/members/{uid}` | Set left_at date for member. | *Path parameters* | `{"detail": "Member deactivated"}` |
| **POST** | `/api/groups/{id}/expenses` | Log new group expense. | `{"amount", "desc", "splits": []}`| `{"id", "amount", "splits": []}` |
| **POST** | `/api/groups/{id}/settlements`| Log direct payment. | `{"from_user_id", "to_user_id"}` | `{"id", "amount", "settle_date"}` |
| **GET** | `/api/groups/{id}/balances` | Get group balance sheets. | *Path parameters* | `{"net_balances", "simplified"}` |
| **POST** | `/api/groups/{id}/imports/upload` | Stage CSV for review. | *Multipart file upload* | `{"id", "status", "records": []}` |
| **POST** | `/api/imports/{id}/approve` | Confirm and import CSV records. | `{"resolutions": []}` | `{"imported_expenses", "settle"}` |

---

## 🚀 Setup & Execution Guide

### 1. Environment Setup
Configure your local environment details (located in `backend/.env`):
```ini
DATABASE_URL=postgresql://postgres:pavan@localhost:5432/split_expenser
SECRET_KEY=yoursecretjwtkeyhere
ALGORITHM=HS256
```

### 2. Run Backend API
```bash
cd backend
pip install -r requirements.txt
py seed.py    # Builds tables and configures active timelines
py -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

### 3. Run Frontend SPA
```bash
cd frontend
npm install
npm run dev
```

### 4. Run Test Suite
```bash
cd backend
py -m pytest
```

---

## 🧠 System Design Interview Q&A Preparation

### Q1: Why use integer cents instead of float decimal columns for balances?
- **Float Rounding Errors**: Floating-point numbers represent fractions using binary bases, introducing representation errors (e.g. `0.1 + 0.2` becomes `0.30000000000000004`). In a finance ledger, these fractions accumulate over millions of splits.
- **Integer Cents Solution**: Storing money as integer cents (e.g. 15.50 INR is stored as `1550`) removes rounding errors. We distribute remainder cents sequentially among participants during equal splits (e.g. splitting 100 cents among 3 people distributes 34, 33, and 33 cents), ensuring the sum of shares always matches the total expense amount.

### Q2: How does the greedy min-flow algorithm simplify transactions?
- **Greedy Min-Flow**: It matches the user who owes the most (largest debtor) with the user who is owed the most (largest creditor). This guarantees that the total number of transactions is minimized (at most $N-1$ transactions, where $N$ is the number of members).
- **Trade-off**: Min-Flow reduces transaction counts but does not guarantee the minimum total sum of money moved. For consumer applications (like Splitwise), reducing transaction count is the optimal user experience.

### Q3: How did you solve the timezone conflict between SQLite and PostgreSQL?
- **Behavior Mismatch**: PostgreSQL natively stores and returns timezone-aware datetimes (`datetime.datetime(..., tzinfo=datetime.timezone.utc)`). SQLite stores datetimes as text strings, returning timezone-naive datetime objects. Direct comparisons during unit tests triggered `TypeError: can't compare offset-naive and offset-aware datetimes`.
- **Solution**: We created a utility comparison helper that inspects `tzinfo` and replaces/strips offsets during roster checks and test assertions, ensuring consistent behavior across both engines:
  ```python
  exp_date_naive = expense_date.replace(tzinfo=None) if expense_date.tzinfo else expense_date
  joined_naive = joined.replace(tzinfo=None) if joined.tzinfo else joined
  ```

### Q4: Why isolate imports in a staged staging area instead of directly writing to the ledger?
- **Database Contamination**: Importing directly is atomic. If a file has one unrecognized name or typo on row 20, the database transaction must roll back, losing all corrections. If we don't roll back, the database becomes contaminated.
- **Staging Solution**: Staging saves raw CSV columns in temporary `ImportRecord` rows. This allows the frontend to retrieve the records, run calculations, and allow the user to resolve anomalies (e.g., mapping unrecognized payers) before committing the clean data to the ledger.
