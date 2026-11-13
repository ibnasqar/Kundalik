# Kundalik

from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, Field
from typing import Optional
from datetime import date, datetime
import sqlite3
from contextlib import contextmanager

DB = "rivojlanish.db"

app = FastAPI(
    title="Rivojlanish API",
    description="Shaxsiy rivojlanish va moliya boshqaruvi",
    version="1.0.0"
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


@contextmanager
def database():
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()


def rows(sql, params=()):
    with database() as db:
        return [dict(x) for x in db.execute(sql, params).fetchall()]


def row(sql, params=()):
    result = rows(sql, params)
    return result[0] if result else None


def run(sql, params=()):
    with database() as db:
        cur = db.execute(sql, params)
        return cur.lastrowid


def required_record(table, record_id):
    result = row(
        f"SELECT * FROM {table} WHERE id=?",
        (record_id,)
    )
    if not result:
        raise HTTPException(404, "Ma'lumot topilmadi")
    return result


@app.on_event("startup")
def initialize_database():
    with database() as db:
        db.executescript("""
        CREATE TABLE IF NOT EXISTS habits (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            description TEXT DEFAULT '',
            importance TEXT DEFAULT 'O‘rta',
            priority INTEGER DEFAULT 10,
            active INTEGER DEFAULT 1,
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS habit_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            habit_id INTEGER NOT NULL,
            log_date TEXT NOT NULL,
            completed INTEGER DEFAULT 1,
            UNIQUE(habit_id, log_date),
            FOREIGN KEY(habit_id)
                REFERENCES habits(id) ON DELETE CASCADE
        );

        CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            kind TEXT NOT NULL,
            UNIQUE(name, kind)
        );

        CREATE TABLE IF NOT EXISTS transactions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            kind TEXT NOT NULL,
            category_id INTEGER,
            amount REAL NOT NULL,
            tx_date TEXT NOT NULL,
            description TEXT DEFAULT '',
            FOREIGN KEY(category_id)
                REFERENCES categories(id) ON DELETE SET NULL
        );

        CREATE TABLE IF NOT EXISTS budgets (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            category_id INTEGER NOT NULL,
            period TEXT NOT NULL,
            amount REAL NOT NULL,
            UNIQUE(category_id, period),
            FOREIGN KEY(category_id)
                REFERENCES categories(id) ON DELETE CASCADE
        );

        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT DEFAULT '',
            due_date TEXT,
            priority TEXT DEFAULT 'O‘rta',
            status TEXT DEFAULT 'Rejada',
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS goals (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            description TEXT DEFAULT '',
            target_date TEXT,
            progress INTEGER DEFAULT 0,
            status TEXT DEFAULT 'Faol',
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        );
        """)

        default_categories = [
            ("Oziq-ovqat", "Xarajat"),
            ("Transport", "Xarajat"),
            ("Uy-joy", "Xarajat"),
            ("Ta'lim", "Xarajat"),
            ("Salomatlik", "Xarajat"),
            ("Ko'ngilochar", "Xarajat"),
            ("Boshqa", "Xarajat"),
            ("Ish haqi", "Daromad"),
            ("Qo'shimcha daromad", "Daromad"),
            ("Investitsiya", "Daromad"),
        ]

        db.executemany(
            "INSERT OR IGNORE INTO categories(name,kind) VALUES(?,?)",
            default_categories
        )


# =========================
# MODELLAR
# =========================

class Habit(BaseModel):
    name: str
    description: str = ""
    importance: str = "O‘rta"
    priority: int = 10
    active: bool = True


class HabitLog(BaseModel):
    log_date: date = Field(default_factory=date.today)
    completed: bool = True


class Category(BaseModel):
    name: str
    kind: str


class Transaction(BaseModel):
    kind: str
    category_id: Optional[int] = None
    amount: float = Field(gt=0)
    tx_date: date = Field(default_factory=date.today)
    description: str = ""


class Budget(BaseModel):
    category_id: int
    period: str
    amount: float = Field(gt=0)


class Task(BaseModel):
    title: str
    description: str = ""
    due_date: Optional[date] = None
    priority: str = "O‘rta"
    status: str = "Rejada"


class Goal(BaseModel):
    name: str
    description: str = ""
    target_date: Optional[date] = None
    progress: int = Field(default=0, ge=0, le=100)
    status: str = "Faol"


# =========================
# BOSH SAHIFA
# =========================

@app.get("/")
def home():
    return {
        "application": "Rivojlanish",
        "message": "Har kuni bir qadam",
        "version": "1.0.0",
        "api_docs": "/docs"
    }


@app.get("/api/dashboard")
def dashboard():
    today = date.today().isoformat()

    habits = row(
        "SELECT COUNT(*) AS total FROM habits WHERE active=1"
    )["total"]

    completed = row("""
        SELECT COUNT(*) AS total
        FROM habit_logs
        WHERE log_date=? AND completed=1
    """, (today,))["total"]

    tasks = row("""
        SELECT COUNT(*) AS total
        FROM tasks WHERE status!='Bajarildi'
    """)["total"]

    finance = row("""
        SELECT
        SUM(CASE WHEN kind='Daromad' THEN amount ELSE 0 END) income,
        SUM(CASE WHEN kind='Xarajat' THEN amount ELSE 0 END) expense
        FROM transactions
        WHERE tx_date=?
    """, (today,))

    return {
        "date": today,
        "habits_total": habits,
        "habits_completed": completed,
        "tasks_pending": tasks,
        "today_income": finance["income"] or 0,
        "today_expense": finance["expense"] or 0,
        "today_balance":
            (finance["income"] or 0) - (finance["expense"] or 0)
    }


# =========================
# ODATLAR
# =========================

@app.get("/api/habits")
def get_habits():
    return rows("""
        SELECT * FROM habits
        ORDER BY priority ASC, id DESC
    """)


@app.post("/api/habits")
def create_habit(data: Habit):
    if not data.name.strip():
        raise HTTPException(400, "Odat nomi kiritilmagan")

    habit_id = run("""
        INSERT INTO habits
        (name, description, importance, priority, active)
        VALUES (?, ?, ?, ?, ?)
    """, (
        data.name.strip(),
        data.description,
        data.importance,
        data.priority,
        int(data.active)
    ))

    return required_record("habits", habit_id)


@app.put("/api/habits/{habit_id}")
def update_habit(habit_id: int, data: Habit):
    required_record("habits", habit_id)

    with database() as db:
        db.execute("""
            UPDATE habits
            SET name=?, description=?, importance=?,
                priority=?, active=?
            WHERE id=?
        """, (
            data.name,
            data.description,
            data.importance,
            data.priority,
            int(data.active),
            habit_id
        ))

    return required_record("habits", habit_id)


@app.delete("/api/habits/{habit_id}")
def delete_habit(habit_id: int):
    required_record("habits", habit_id)
    run("DELETE FROM habits WHERE id=?", (habit_id,))
    return {"message": "Odat o'chirildi"}


@app.post("/api/habits/{habit_id}/logs")
def log_habit(habit_id: int, data: HabitLog):
    required_record("habits", habit_id)

    with database() as db:
        db.execute("""
            INSERT INTO habit_logs(habit_id, log_date, completed)
            VALUES (?, ?, ?)
            ON CONFLICT(habit_id, log_date)
            DO UPDATE SET completed=excluded.completed
        """, (
            habit_id,
            data.log_date.isoformat(),
            int(data.completed)
        ))

    return {"message": "Odat holati saqlandi"}


@app.get("/api/habits/{habit_id}/logs")
def get_habit_logs(habit_id: int):
    required_record("habits", habit_id)
    return rows("""
        SELECT * FROM habit_logs
        WHERE habit_id=?
        ORDER BY log_date DESC
    """, (habit_id,))


# =========================
# MOLIYAVIY KATEGORIYALAR
# =========================

@app.get("/api/categories")
def get_categories(kind: Optional[str] = None):
    if kind:
        return rows(
            "SELECT * FROM categories WHERE kind=? ORDER BY name",
            (kind,)
        )
    return rows("SELECT * FROM categories ORDER BY kind, name")


@app.post("/api/categories")
def create_category(data: Category):
    if data.kind not in ["Daromad", "Xarajat"]:
        raise HTTPException(400, "Kategoriya turi noto'g'ri")

    try:
        category_id = run(
            "INSERT INTO categories(name,kind) VALUES(?,?)",
            (data.name.strip(), data.kind)
        )
    except sqlite3.IntegrityError:
        raise HTTPException(409, "Bu kategoriya mavjud")

    return required_record("categories", category_id)


@app.put("/api/categories/{category_id}")
def update_category(category_id: int, data: Category):
    required_record("categories", category_id)

    with database() as db:
        db.execute("""
            UPDATE categories SET name=?, kind=? WHERE id=?
        """, (data.name.strip(), data.kind, category_id))

    return required_record("categories", category_id)


@app.delete("/api/categories/{category_id}")
def delete_category(category_id: int):
    required_record("categories", category_id)
    run("DELETE FROM categories WHERE id=?", (category_id,))
    return {"message": "Kategoriya o'chirildi"}


# =========================
# KIRIM VA CHIQIMLAR
# =========================

@app.get("/api/transactions")
def get_transactions(
    kind: Optional[str] = None,
    start: Optional[date] = None,
    end: Optional[date] = None
):
    sql = """
        SELECT t.*, c.name AS category
        FROM transactions t
        LEFT JOIN categories c ON c.id=t.category_id
        WHERE 1=1
    """
    params = []

    if kind:
        sql += " AND t.kind=?"
        params.append(kind)

    if start:
        sql += " AND t.tx_date>=?"
        params.append(start.isoformat())

    if end:
        sql += " AND t.tx_date<=?"
        params.append(end.isoformat())

    sql += " ORDER BY t.tx_date DESC, t.id DESC"
    return rows(sql, tuple(params))


@app.post("/api/transactions")
def create_transaction(data: Transaction):
    if data.kind not in ["Daromad", "Xarajat"]:
        raise HTTPException(400, "Operatsiya turi noto'g'ri")

    if data.category_id is not None:
        category = required_record("categories", data.category_id)
        if category["kind"] != data.kind:
            raise HTTPException(
                400, "Kategoriya operatsiya turiga mos emas"
            )

    transaction_id = run("""
        INSERT INTO transactions
        (kind, category_id, amount, tx_date, description)
        VALUES (?, ?, ?, ?, ?)
    """, (
        data.kind,
        data.category_id,
        data.amount,
        data.tx_date.isoformat(),
        data.description
    ))

    return row(
        "SELECT * FROM transactions WHERE id=?",
        (transaction_id,)
    )


@app.put("/api/transactions/{transaction_id}")
def update_transaction(
    transaction_id: int,
    data: Transaction
):
    required_record("transactions", transaction_id)

    if data.kind not in ["Daromad", "Xarajat"]:
        raise HTTPException(400, "Operatsiya turi noto'g'ri")

    if data.category_id is not None:
        category = required_record("categories", data.category_id)
        if category["kind"] != data.kind:
            raise HTTPException(400, "Kategoriya turi mos emas")

    with database() as db:
        db.execute("""
            UPDATE transactions
            SET kind=?, category_id=?, amount=?,
                tx_date=?, description=?
            WHERE id=?
        """, (
            data.kind,
            data.category_id,
            data.amount,
            data.tx_date.isoformat(),
            data.description,
            transaction_id
        ))

    return required_record("transactions", transaction_id)


@app.delete("/api/transactions/{transaction_id}")
def delete_transaction(transaction_id: int):
    required_record("transactions", transaction_id)
    run("DELETE FROM transactions WHERE id=?", (transaction_id,))
    return {"message": "Operatsiya o'chirildi"}


# =========================
# XARAJAT LIMITLARI
# =========================

@app.get("/api/budgets")
def get_budgets():
    return rows("""
        SELECT b.*, c.name AS category
        FROM budgets b
        JOIN categories c ON c.id=b.category_id
        ORDER BY c.name
    """)


@app.post("/api/budgets")
def create_budget(data: Budget):
    category = required_record("categories", data.category_id)

    if category["kind"] != "Xarajat":
        raise HTTPException(400, "Limit faqat xarajat kategoriyasiga qo'yiladi")

    if data.period not in ["Kunlik", "Oylik"]:
        raise HTTPException(400, "Davr noto'g'ri")

    with database() as db:
        db.execute("""
            INSERT INTO budgets(category_id, period, amount)
            VALUES (?, ?, ?)
            ON CONFLICT(category_id, period)
            DO UPDATE SET amount=excluded.amount
        """, (data.category_id, data.period, data.amount))

    return {"message": "Limit saqlandi"}


@app.get("/api/budgets/status")
def budget_status():
    today = date.today()
    result = []

    for budget in get_budgets():
        if budget["period"] == "Kunlik":
            period_start = today.isoformat()
        else:
            period_start = today.replace(day=1).isoformat()

        spent = row("""
            SELECT COALESCE(SUM(amount),0) AS total
            FROM transactions
            WHERE kind='Xarajat'
            AND category_id=?
            AND tx_date>=?
            AND tx_date<=?
        """, (
            budget["category_id"],
            period_start,
            today.isoformat()
        ))["total"]

        result.append({
            **budget,
            "spent": spent,
            "remaining": budget["amount"] - spent,
            "exceeded": spent > budget["amount"]
        })

    return result


@app.delete("/api/budgets/{budget_id}")
def delete_budget(budget_id: int):
    required_record("budgets", budget_id)
    run("DELETE FROM budgets WHERE id=?", (budget_id,))
    return {"message": "Limit o'chirildi"}


# =========================
# VAZIFALAR
# =========================

@app.get("/api/tasks")
def get_tasks(status: Optional[str] = None):
    if status:
        return rows("""
            SELECT * FROM tasks WHERE status=?
            ORDER BY due_date
        """, (status,))

    return rows("SELECT * FROM tasks ORDER BY due_date, id DESC")


@app.post("/api/tasks")
def create_task(data: Task):
    if not data.title.strip():
        raise HTTPException(400, "Vazifa nomi kerak")

    task_id = run("""
        INSERT INTO tasks
        (title, description, due_date, priority, status)
        VALUES (?, ?, ?, ?, ?)
    """, (
        data.title,
        data.description,
        data.due_date.isoformat() if data.due_date else None,
        data.priority,
        data.status
    ))

    return required_record("tasks", task_id)


@app.put("/api/tasks/{task_id}")
def update_task(task_id: int, data: Task):
    required_record("tasks", task_id)

    with database() as db:
        db.execute("""
            UPDATE tasks
            SET title=?, description=?, due_date=?,
                priority=?, status=?
            WHERE id=?
        """, (
            data.title,
            data.description,
            data.due_date.isoformat() if data.due_date else None,
            data.priority,
            data.status,
            task_id
        ))

    return required_record("tasks", task_id)


@app.delete("/api/tasks/{task_id}")
def delete_task(task_id: int):
    required_record("tasks", task_id)
    run("DELETE FROM tasks WHERE id=?", (task_id,))
    return {"message": "Vazifa o'chirildi"}


# =========================
# MAQSADLAR
# =========================

@app.get("/api/goals")
def get_goals():
    return rows("SELECT * FROM goals ORDER BY target_date")


@app.post("/api/goals")
def create_goal(data: Goal):
    goal_id = run("""
        INSERT INTO goals
        (name, description, target_date, progress, status)
        VALUES (?, ?, ?, ?, ?)
    """, (
        data.name,
        data.description,
        data.target_date.isoformat() if data.target_date else None,
        data.progress,
        data.status
    ))

    return required_record("goals", goal_id)


@app.put("/api/goals/{goal_id}")
def update_goal(goal_id: int, data: Goal):
    required_record("goals", goal_id)

    with database() as db:
        db.execute("""
            UPDATE goals
            SET name=?, description=?, target_date=?,
                progress=?, status=?
            WHERE id=?
        """, (
            data.name,
            data.description,
            data.target_date.isoformat() if data.target_date else None,
            data.progress,
            data.status,
            goal_id
        ))

    return required_record("goals", goal_id)


@app.delete("/api/goals/{goal_id}")
def delete_goal(goal_id: int):
    required_record("goals", goal_id)
    run("DELETE FROM goals WHERE id=?", (goal_id,))
    return {"message": "Maqsad o'chirildi"}


# =========================
# HISOBOTLAR
# =========================

@app.get("/api/reports/finance")
def finance_report(
    start: Optional[date] = None,
    end: Optional[date] = None
):
    sql = """
        SELECT kind, SUM(amount) AS total
        FROM transactions WHERE 1=1
    """
    params = []

    if start:
        sql += " AND tx_date>=?"
        params.append(start.isoformat())

    if end:
        sql += " AND tx_date<=?"
        params.append(end.isoformat())

    sql += " GROUP BY kind"

    totals = rows(sql, tuple(params))

    income = next(
        (x["total"] for x in totals if x["kind"] == "Daromad"), 0
    ) or 0

    expense = next(
        (x["total"] for x in totals if x["kind"] == "Xarajat"), 0
    ) or 0

    categories = rows("""
        SELECT c.name AS category, SUM(t.amount) AS total
        FROM transactions t
        LEFT JOIN categories c ON c.id=t.category_id
        WHERE t.kind='Xarajat'
        GROUP BY t.category_id
        ORDER BY total DESC
    """)

    return {
        "income": income,
        "expense": expense,
        "balance": income - expense,
        "expenses_by_category": categories
    }


@app.get("/api/reports/habits")
def habits_report():
    return rows("""
        SELECT
            h.id,
            h.name,
            COUNT(
                CASE WHEN l.completed=1 THEN 1 END
            ) AS completed_days
        FROM habits h
        LEFT JOIN habit_logs l ON l.habit_id=h.id
        GROUP BY h.id
        ORDER BY completed_days DESC
    """)


@app.get("/api/health")
def health():
    return {
        "status": "ok",
        "database": "connected",
        "time": datetime.now().isoformat()
    }
