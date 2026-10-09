# Database Storage: SQL (PostgreSQL) & NoSQL (MongoDB)

Up to this point, we have stored data either in memory (variables, lists, dictionaries) or in flat files on disk (text files, JSON). While files work well for simple configurations and export reports, production applications face real-world challenges:

* **Massive Scale:** Searching through a 10 GB JSON file in memory will crash your program.
* **Concurrent Access:** Multiple users writing to a text file simultaneously causes data corruption.
* **Complex Relationships & Queries:** Filtering, joining, and aggregating nested data with standard loops is slow and error-prone.

**Databases** solve these problems by providing fast indexing, concurrent read/write protection (ACID transactions), and optimized query engines.

---

## 1. SQL vs. NoSQL: How Python Sees Them

| Feature | Relational (SQL) - PostgreSQL | Document (NoSQL) - MongoDB |
| :--- | :--- | :--- |
| **Data Model** | Tables, Rows, and Columns | Collections and BSON Documents |
| **Schema** | Rigid, predefined schema | Flexible, dynamic schema |
| **Python Mapping** | Rows map to **Tuples** or **NamedTuples** | Documents map directly to **Dictionaries** |
| **Best Used For** | Structured data, financial records, relationships | Hierarchical data, real-time analytics, rapid prototyping |
| **Standard Driver** | `psycopg` (or `psycopg2`) | `pymongo` |

---

## 2. Relational Storage: PostgreSQL with Python

PostgreSQL is an enterprise-grade, open-source relational database. Data is stored in strict **tables** where each row has fixed columns defined by data types (e.g., `VARCHAR`, `INTEGER`, `BOOLEAN`).

### Installation

```bash
pip install psycopg[binary]
```

### Python Implementation: CRUD Operations

Here is how you connect, create tables, insert data safely, and query results:

```python
import psycopg

# 1. Establish connection to the PostgreSQL database
# In production, load credentials from environment variables!
CONNECTION_URI = "postgresql://postgres:secretpassword@localhost:5432/training_db"

with psycopg.connect(CONNECTION_URI) as conn:
    # 2. Open a cursor to execute SQL statements
    with conn.cursor() as cur:
        # Create a table
        cur.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                username VARCHAR(50) UNIQUE NOT NULL,
                email VARCHAR(100) NOT NULL,
                age INT,
                is_active BOOLEAN DEFAULT TRUE
            );
        """)

        # 3. Parameterized INSERT (Prevents SQL Injection!)
        new_user = ("alice_dev", "alice@example.com", 28)
        cur.execute(
            """
            INSERT INTO users (username, email, age)
            VALUES (%s, %s, %s)
            ON CONFLICT (username) DO NOTHING;
            """,
            new_user
        )

        # 4. Bulk INSERT using a list of tuples
        user_batch = [
            ("bob_admin", "bob@example.com", 34),
            ("carol_analyst", "carol@example.com", 25)
        ]
        cur.executemany(
            """
            INSERT INTO users (username, email, age)
            VALUES (%s, %s, %s)
            ON CONFLICT (username) DO NOTHING;
            """,
            user_batch
        )

        # 5. Query data (SELECT)
        cur.execute("SELECT id, username, email, age FROM users WHERE age >= %s;", (25,))
        
        # fetchall() returns rows as Python tuples: (id, username, email, age)
        rows = cur.fetchall()
        for row in rows:
            print(f"ID: {row[0]} | User: {row[1]} | Email: {row[2]} | Age: {row[3]}")

    # Changes are automatically committed when the context manager exits cleanly!
    conn.commit()
```

**Never Format Raw Strings Into SQL Queries:**

```python
# ❌ CRITICAL SECURITY VULNERABILITY (SQL Injection):
cur.execute(f"SELECT * FROM users WHERE username = '{user_input}';")

# ✅ SECURE: Always use parameterized queries
cur.execute("SELECT * FROM users WHERE username = %s;", (user_input,))
```

---

## 3. Document Storage: MongoDB with Python

MongoDB is a document-oriented NoSQL database. Instead of rows and tables, it stores data in **BSON** (Binary JSON) documents organized inside **collections**.

Because documents are JSON-like structures, they map **1:1 to Python dictionaries**—including nested dictionaries and lists!

### Installation

```bash
pip install pymongo
```

### Python Implementation: CRUD Operations

```python
from pymongo import MongoClient
from datetime import datetime

# 1. Connect to the local or cloud (Atlas) MongoDB instance
client = MongoClient("mongodb://localhost:27017/")

# 2. Select database and collection (created automatically on first write)
db = client["company_directory"]
employees = db["employees"]

# 3. INSERT: Pass native Python dictionaries directly
new_employee = {
    "name": "Sarah Connor",
    "role": "Systems Engineer",
    "skills": ["Linux", "Python", "Docker"],
    "details": {
        "office": "Building A",
        "remote": True
    },
    "joined_at": datetime.now()
}

result = employees.insert_one(new_employee)
print(f"Inserted document ID: {result.inserted_id}")

# 4. BULK INSERT: Pass a list of dictionaries
team = [
    {"name": "John Doe", "role": "Backend Developer", "skills": ["Python", "PostgreSQL"], "experience_years": 4},
    {"name": "Jane Smith", "role": "Data Scientist", "skills": ["Python", "Pandas", "PyTorch"], "experience_years": 6}
]
employees.insert_many(team)

# 5. QUERY: Search using dictionary filters
# Find all employees who have "Python" in their skills list and experience >= 4 years
query = {
    "skills": "Python",
    "experience_years": {"$gte": 4}
}
matching_employees = employees.find(query)

for emp in matching_employees:
    print(f"- {emp['name']} ({emp['role']})")

# 6. UPDATE: Modify documents using atomic operators like $set
employees.update_one(
    {"name": "Sarah Connor"},
    {"$set": {"role": "Lead Systems Architect"}}
)

# 7. DELETE: Remove documents matching a filter
employees.delete_one({"name": "John Doe"})

# Close the client connection
client.close()
```

---

## 4. When to Use What: Architecture Decision Guide

```mermaid
flowchart TD
    Start["Need Persistent Storage?"] --> Scale{"Data Scale & Concurrency"}
    Scale -->|Single user, simple config, small logs| Files["Flat Files\n(JSON / CSV / SQLite)"]
    Scale -->|Multi-user, high volume, mission critical| Type{"Data Structure"}
    Type -->|Strict schemas, relational joins, financial data| PG["PostgreSQL\n(Relational SQL)"]
    Type -->|Flexible schema, nested documents, rapid iteration| Mongo["MongoDB\n(Document NoSQL)"]
```

* **Use Flat Files (JSON / CSV) when:** The data is small, read infrequently, or used solely for export/import between tools.
* **Use PostgreSQL when:** Data integrity, relational joins (e.g., Orders $\rightarrow$ Customers $\rightarrow$ Products), and strict ACID compliance are required.
* **Use MongoDB when:** Data structures vary between records, data is naturally nested (hierarchical documents), or requirements change rapidly during development.
