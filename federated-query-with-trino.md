# **Federated Querying with Trino (SQL Server \+ PostgreSQL)**

This guide demonstrates how to use **Trino** (formerly PrestoSQL) to run a single SQL query that joins data across two different databases: **Microsoft SQL Server** (storing User/Investor data) and **PostgreSQL** (storing Stock Orders).  
We will conclude with a **Python** script to execute this federated query programmatically.

## **Architecture Overview**

* **Trino Server**: Acts as the query engine.  
* **Catalog A (SQL Server)**: Contains the investors table.  
* **Catalog B (PostgreSQL)**: Contains the stock\_orders table.  
* **Python Client**: Submits the query to Trino.

---

## **1\. Trino Configuration**

To make Trino "see" your databases, you must configure **Catalogs**. These are simple properties files stored in /etc/trino/catalog/ inside the Trino container.

### **A. SQL Server Catalog (sqlserver.properties)**

*Assuming SQL Server is accessible at sqlserver-host.*

Properties

connector.name=sqlserver  
connection-url=jdbc:sqlserver://sqlserver-host:1433;databaseName=FinancialDB;encrypt=false  
connection-user=sa  
connection-password=StrongPassword123\!

### **B. PostgreSQL Catalog (postgres.properties)**

*Assuming Postgres is accessible at postgres-host.*

Properties

connector.name=postgresql  
connection-url=jdbc:postgresql://postgres-host:5432/orders\_db  
connection-user=postgres  
connection-password=secret

## **2\. The Hypothetical Data Model**

SQL Server Table: FinancialDB.dbo.investors  
| id | full\_name | region |  
| :--- | :--- | :--- |  
| 101 | Alice Carter | EU |  
| 102 | Bob Smith | US |  
PostgreSQL Table: orders\_db.public.stock\_orders  
| order\_id | investor\_id | symbol | quantity |  
| :--- | :--- | :--- | :--- |  
| 5001 | 101 | AAPL | 10 |  
| 5002 | 101 | MSFT | 5 |

## **3\. The Python Solution**

We will use the official trino python client.

### **Prerequisites**

Bash

pip install trino

### **query\_investor.py**

Python

import trino  
from trino.dbapi import connect

def get\_investor\_portfolio(investor\_id):  
    """  
    Queries Trino to join SQL Server investors with Postgres orders.  
    """  
      
    \# 1\. Establish Connection to Trino  
    conn \= connect(  
        host='localhost',      \# Trino Coordinator Host  
        port=8080,             \# Default Trino Port  
        user='admin',          \# User (can be arbitrary for default setup)  
        catalog='system',      \# Default catalog to land in  
        schema='runtime'  
    )  
      
    cur \= conn.cursor()

    \# 2\. Define the Federated Query  
    \# Note the fully qualified names: catalog.schema.table  
    sql \= """  
    SELECT   
        i.full\_name,  
        i.region,  
        s.symbol,  
        s.quantity,  
        (s.quantity \* 150.00) as estimated\_value \-- Arbitrary price for demo  
    FROM   
        sqlserver.dbo.investors AS i  
    JOIN   
        postgres.public.stock\_orders AS s  
    ON   
        i.id \= s.investor\_id  
    WHERE   
        i.id \= ?  
    """

    print(f"Fetching portfolio for Investor ID: {investor\_id}...")  
      
    \# 3\. Execute with parameter binding (safe against injection)  
    cur.execute(sql, (investor\_id,))  
      
    \# 4\. Process Results  
    rows \= cur.fetchall()  
      
    if not rows:  
        print("No orders found for this investor.")  
        return

    print(f"\\n{'Name':\<15} | {'Region':\<10} | {'Symbol':\<10} | {'Qty':\<5} | {'Value'}")  
    print("-" \* 60\)  
      
    for row in rows:  
        name, region, symbol, qty, val \= row  
        print(f"{name:\<15} | {region:\<10} | {symbol:\<10} | {qty:\<5} | ${val:,.2f}")

if \_\_name\_\_ \== "\_\_main\_\_":  
    \# Example: Query for Investor 101  
    get\_investor\_portfolio(101)

---

## **4\. How to Run (Docker Quickstart)**

If you don't have Trino running yet, you can spin it up instantly with Docker, mounting your catalog files.  
**Structure:**

Plaintext

.  
├── docker-compose.yml  
└── catalogs/  
    ├── sqlserver.properties  
    └── postgres.properties

**docker-compose.yml:**

YAML

services:  
  trino:  
    image: trinodb/trino:latest  
    container\_name: trino-coordinator  
    ports:  
      \- "8080:8080"  
    volumes:  
      \# Mount your local catalog configs into the container  
      \- ./catalogs:/etc/trino/catalog

**Run command:**

Bash

docker-compose up \-d  
python query\_investor.py

---

### **💡 Key Takeaway**

The magic lies in the **FROM** clause. Trino abstracts the physical location of the data, allowing you to treat distinct databases as if they were just schemas in the same engine:  
sqlserver.dbo.investors ↔ postgres.public.stock\_orders  
---

### **Instructions to Export**

You can now click the **Share/Export** icon below this message and select **"Export to Docs"** to save this specific version.