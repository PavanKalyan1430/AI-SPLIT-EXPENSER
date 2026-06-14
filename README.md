# Split Expenser

Split Expenser is a premium Shared Expense Management Application built with a **FastAPI** backend, a local **PostgreSQL** database, and a **Vite + React** frontend. It features an interactive **staged CSV anomaly validator wizard**, timezone-aware date roster tracking, and a **greedy min-flow debt simplification engine** (Debt Simplification Pathway).

---

## 🛠️ Technology Stack
- **Backend**: Python 3.11+, FastAPI, SQLAlchemy ORM, Uvicorn, PostgreSQL
- **Frontend**: React, Vite, CSS (vanilla with modern glassmorphism styling)
- **Database**: PostgreSQL (relational integrity with cascaded deletes and RESTRICT constraints)
- **AI Tooling**: Google Gemini / Antigravity Pair-Programming Agent

---

## 🚀 Setup & Installation

### Prerequisites
- Python 3.11+ installed
- Node.js (v18+) installed
- PostgreSQL server running locally (e.g., via pgAdmin)

### 1. Database Setup
Create a PostgreSQL database named `split_expenser` with the following credentials (configured in `.env` files):
- **User**: `postgres`
- **Password**: `pavan`
- **Host**: `localhost`
- **Port**: `5432`
- **Database Name**: `split_expenser`

---

### 2. Backend Setup
1. Open a terminal and navigate to the backend directory:
   ```bash
   cd backend
   ```
2. Install the required dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Initialize the database schema and seed the default group, members, and historical roster timelines:
   ```bash
   py seed.py
   ```
4. Run the FastAPI development server:
   ```bash
   py -m uvicorn app.main:app --host 127.0.0.1 --port 8000
   ```
   *The API will be available at [http://127.0.0.1:8000](http://127.0.0.1:8000). You can inspect Swagger Docs at `/docs`.*

---

### 3. Frontend Setup
1. Open a new terminal and navigate to the frontend directory:
   ```bash
   cd frontend
   ```
2. Install the frontend dependencies:
   ```bash
   npm install
   ```
3. Run the React + Vite development server:
   ```bash
   npm run dev
   ```
   *The frontend app will be available at [http://localhost:5173](http://localhost:5173).*

---

## 🧪 Running Automated Tests
The backend features a comprehensive test suite (10 integration and unit tests) checking anomaly rules, net balance calculations, and imports:
1. Navigate to the backend directory:
   ```bash
   cd backend
   ```
2. Execute pytest:
   ```bash
   py -m pytest
   ```

---

## 👥 Seed Roommate Timelines
By running `seed.py`, the following historical timeline setup is loaded into the **Roommates** group:
- **Jan 1, 2026**: Aisha, Rohan, and Priya join the group.
- **March 29, 2026**: Meera leaves the group (inactive for subsequent transactions).
- **April 5, 2026**: Sam joins the group.
- **Dev & Kabir**: Created dynamically and retroactively joined during the CSV import workflow.
