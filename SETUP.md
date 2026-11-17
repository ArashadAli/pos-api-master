# Open POS API - Setup Guide

This document explains how to set up, configure, and run the Open POS API project locally from scratch on a Windows machine with Python 3.11 and PostgreSQL.

---

## 1. Prerequisites

Before starting, make sure you have:
- **Python 3.11+** installed (check with `python --version`)
- **PostgreSQL** installed and running (check with pgAdmin or psql)
- **Git** installed

---

## 2. Step-by-Step Installation

### Step 1: Navigate to the Project Directory
Open your terminal and make sure you are in the directory containing `manager.py`:
```powershell
cd open-pos-api-master
```

### Step 2: Create and Activate Virtual Environment
Create an isolated virtual environment so project dependencies don't mix with your global Python:
```powershell
python -m venv venv
.\venv\Scripts\Activate.ps1
```
*(If you see an execution policy error in PowerShell, run `Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned` first).*

You should now see `(venv)` at the beginning of your prompt.

### Step 3: Install Dependencies
Install all required packages from `requirements.txt`:
```powershell
pip install -r requirements.txt
```

### Step 4: Configure Environment Variables
Copy `.env.example` to create your own `.env` file:
```powershell
copy .env.example .env
```

### Step 5: Create the PostgreSQL Database
Open pgAdmin or `psql` and create the database:
```sql
CREATE DATABASE "open-pos-DB";
```

### Step 6: Initialize Database & Create Admin User
Run the database migrations and create your initial admin account:
```powershell
# 1. Run migrations to create all database tables
flask db upgrade

# 2. (Or if setting up fresh without migrations)
flask create-db

# 3. Create the initial admin user with full permissions
flask create-admin -e admin@pos.com -p "admin123" -n "Admin User" -m "9999999999"
```

### Step 7: Run the Application
Start the development server:
```powershell
flask run
```
The server will start at: `http://127.0.0.1:5000/`

---

## 4. Problems Faced & How They Were Fixed

Since this project was originally built around 2017 with Python 3.6, several updates were required to run it smoothly on modern Python 3.11:

### 1. Inactive Git Protocol in `requirements.txt`
* **Issue:** `requirements.txt` had `-e git://github.com/...`, which failed because GitHub retired the unencrypted `git://` protocol.
* **Fix:** Changed the protocol to `git+https://`.

### 2. C-Build & MSVC Compilation Errors
* **Issue:** The original requirements contained frozen versions of older libraries like `cffi==1.9.1`, `pycrypto`, and junk libraries (`jupyter`, `boto3`). Windows failed to compile them without Visual C++ Build Tools.
* **Fix:** Cleaned `requirements.txt` to include only active dependencies with compatible version ranges so pip installs pre-built binary wheels directly. Replaced legacy `pygresql` with `psycopg2-binary`.

### 3. `ImportError: cannot import name 'soft_unicode' from 'markupsafe'`
* **Issue:** `MarkupSafe` 3.x removed `soft_unicode`, which older `Jinja2` 2.x expects.
* **Fix:** Pinned `MarkupSafe==2.0.1`.

### 4. `TypeError: Type Type cannot be instantiated`
* **Issue:** In `src/utils/resource.py` (line 185), type hints used parentheses `Type(db.Model)` which modern Python 3.11 strictly disallows.
* **Fix:** Updated to square brackets: `Type[db.Model]`.

### 5. `inspect.getargspec` Removed in Python 3.11
* **Issue:** Legacy `Flask-Script` crashed with `AttributeError: module 'inspect' has no attribute 'getargspec'`.
* **Fix:** Modernized `manager.py` away from the deprecated `Flask-Script` library and switched completely to native **Flask CLI** with Click commands (`flask run`, `flask db`, `flask routes`, `flask create-admin`).

### 6. Missing PostgreSQL UUID Function (`uuid_generate_v4() does not exist`)
* **Issue:** PostgreSQL tables use UUID primary keys by default, but the `uuid-ossp` extension wasn't enabled.
* **Fix:** Ran `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";` in the database.

### 7. User Registration IntegrityError & Missing Email Validator
* **Issue:** Submitting user registration threw errors because `name` and `mobile_number` were `NOT NULL` in the DB but missing from the default form, and `WTForms` needed `email_validator`.
* **Fix:** Installed `email-validator` and extended the registration form in `src/utils/security.py` to include `name` and `mobile_number`.

### 8. Admin Panel Form Crash (`'tuple' object has no attribute 'items'`)
* **Issue:** When creating a user in `/admin/user/new/`, `WTForms 3.x` broke compatibility with `Flask-Admin 1.5.8`'s field flags.
* **Fix:** Pinned `WTForms==2.3.3` (`WTForms<3.0`).

---

## 5. Verification & Acceptance Criteria

1. **API Login Check:**
   - Open browser and visit: `http://127.0.0.1:5000/api/v1/login/`
   - Form loads successfully with HTTP 200.

2. **Admin Panel Access:**
   - Log in using your created admin credentials (`admin@pos.com` / `admin123`).
   - Navigate to `http://127.0.0.1:5000/admin/`.
   - Verify that you can view and manage Products, Orders, Users, and Retail Shops.