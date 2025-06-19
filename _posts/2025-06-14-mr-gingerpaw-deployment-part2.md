---
  title: Mr. Gingerpaw Project Log - Deployment Part 2 - Database
  description: >-
    A development log for the Mr. Gingerpaw Inventory App. This part focuses on deploying the database to Azure and verifying its connection.
  # author: pointer
  date: 2025-06-14 11:01:00 +0930
  categories: [Blogging, MrGingerpaw]
  tags: [Python, React, Deployment]
  # media_subpath: /images/2022-08-10-ipv6-version-of-frp/
  pin: false
---


## Introduction

The app has three major components: **Database**, **Backend**, and **Frontend**. All of them need to be deployed to Azure, and ideally, at minimal cost — free if possible.

For the database part, I chose **Azure Database for PostgreSQL Flexible Server**.

The following steps assume you already have an Azure account and can log in to the Azure Portal.

## Service Creation

Search for `PostgreSQL` in the Azure Portal search bar and select **Azure Database for PostgreSQL Flexible Server**, then click “Create” and fill in the required details. Below are the explanations and my choices for each field.

> When searching *PostgreSQL*, you may see both "Azure Database for PostgreSQL Server" and "Azure Database for PostgreSQL Flexible Server." Choose the Flexible Server — it’s the newer offering.
{: .prompt-tip }

### Basics

* **Project details (Subscription & Resource Group):** This organizes your services. The **Subscription** relates to your billing plan, while the **Resource Group** is a custom group for related services. I created one resource group for all three parts of my app (DB, backend, frontend).

* **Server details:**

  * **Server name:** Name of your service. Just pick something descriptive.
  * **Region:** I chose **Southeast Asia** as it’s geographically between China and Australia.
  * **PostgreSQL version:** Choose the version that matches your local development.
  * **Workload type:** I selected **Development**.
  * **Compute + Storage:** Click `Configure server` and pick **Burstable B1ms** to save costs.
  * **Availability zone:** I chose **No preference**.

* **High availability:** I selected **Disabled** — the free tier doesn't support redundancy anyway.

* **Authentication:**

  * **Authentication method:** I chose **PostgreSQL and Microsoft Entra authentication**.
  * **Microsoft Entra administrator:** Automatically set to your current account.
  * **Administrator login:** Set your admin username and password here — remember them for future connections.

### Networking

* **Network connectivity:** Choose **Public access** (no need to set up VNet for now).
* **Public access:** Enable "Allow public access to this resource through the internet using a public IP address" to simplify development and testing.
* **Firewall rules:** Enable "Allow public access from any Azure service within Azure to this server." Also, click "Add current client IP address" so your local machine can access the DB.
* **Private endpoints:** Leave empty unless using VNet (not needed here).

### Security

* **Data encryption:** I selected **Service-managed key**. You can upload your own key if you have special encryption needs, but it's not necessary here.

### Tags

* Add tags like `owner` and `project` for easier management later.

---

Once everything is set, go to **Review + Create**, double-check your config, then click **Create**.

## Database Connection

After creation, the next step is to verify the database is working. I tested this in two ways: using database tools, and through my own backend app.

First, locate your PostgreSQL server on Azure. You’ll see the **Endpoint** field:

```txt
[YourServiceName].postgres.database.azure.com
```

Use a GUI tool or IDE extension to connect:

* **pgAdmin**: A graphical tool for PostgreSQL. Make sure to enable **SSL** (`sslmode=require`) or you’ll get a `no encryption` error.
* **VSCode PostgreSQL extension**: Lightweight and useful for reading and debugging. But note:

> The VSCode PostgreSQL extension only connects to one DB at a time. In step 6 (selecting the DB), make sure to pick the exact database name or you won’t see any tables.
{: .prompt-warning }

I used pgAdmin to connect and create the database, and used VSCode only for inspection/debugging.

After confirming the DB is live, I tested connecting to it through my Python backend:

```python
DATABASE_URL = "postgresql+psycopg2://[YourAdminUserName]:[YourAdminPassword]@[YourServiceURL]:[PortNum]/[CreatedDatabaseName]?sslmode=require"

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(DATABASE_URL, echo=False)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
db = SessionLocal()

# Example usage:
# db.add(...)
# db.commit()
```

That’s it!
