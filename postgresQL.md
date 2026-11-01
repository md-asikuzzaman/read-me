# PostgreSQL & Docker Quick Reference

A practical PostgreSQL command cheat sheet for managing databases, users, authentication, and tables inside a Docker container.

## 1. Access PostgreSQL in Docker

**Method 1: Connect directly to PostgreSQL**

```bash
docker exec -it postgres_db psql -U postgres
```

This opens the PostgreSQL CLI directly without opening the container's Bash shell.

**Method 2: Access the Docker container's Bash shell first**

```bash
# Step 1: Enter the container
docker exec -it postgres_db bash

# Step 2: Connect as the postgres user
psql -U postgres

# Step 3: Connect using another PostgreSQL user and database
psql -U limon -d author
```

Exit commands:

```sql
\q
```

Exits PostgreSQL.

```bash
exit
```

Exits the Docker container's Bash shell.

## 2. PostgreSQL Useful Commands

Run these commands inside the PostgreSQL CLI (`psql`).

| Command | Description |
|---|---|
| `\l` | List all databases |
| `\du` | List all users and roles |
| `\c db_name` | Connect to a database |
| `\dt` | List all tables in the current schema |
| `\d table_name` | Show table structure |
| `\q` | Exit PostgreSQL |

**Clear the terminal:**

```bash
CTRL + L
```

## 3. Check PostgreSQL Password

To check the PostgreSQL password environment variable configured in the Docker container:

```bash
docker exec postgres_db printenv POSTGRES_PASSWORD
```

**Important:** This displays the container's environment variable, not necessarily the current password stored in PostgreSQL. If the database password was changed later, the environment variable may not match it.

## 4. Create a PostgreSQL User and Database

Connect to PostgreSQL as an administrator:

```bash
docker exec -it postgres_db psql -U postgres
```

Create a user with a password:

```sql
CREATE USER asik WITH PASSWORD 'strong_password';
```

Create a database and assign ownership to that user:

```sql
CREATE DATABASE ecom_db OWNER asik;
```

Connect using the new user:

```bash
psql -U asik -d ecom_db -W
```

`-W` forces PostgreSQL to ask for the password.

**Useful SQL commands:**

```sql
-- Create a user
CREATE USER username WITH PASSWORD 'your_password';

-- Create a database
CREATE DATABASE database_name OWNER username;

-- Change a user's password
ALTER USER username WITH PASSWORD 'new_password';

-- Drop a user (only if it owns no dependent objects)
DROP USER username;
```

**Note:** If a user owns a database or other objects, you may need to transfer or remove those dependencies before dropping the user.

## 5. Disable Trust Authentication

PostgreSQL's `trust` authentication allows connections without checking a password. For better security, use password-based authentication such as `scram-sha-256`.

### Step 1: Enter the Docker container

```bash
docker exec -it postgres_db bash
```

### Step 2: Back up the authentication configuration

For this PostgreSQL 18 container:

```bash
cp /var/lib/postgresql/18/docker/pg_hba.conf \
   /var/lib/postgresql/18/docker/pg_hba.conf.backup
```

### Step 3: Inspect the configuration

```bash
cat /var/lib/postgresql/18/docker/pg_hba.conf
```

Find the relevant `local` and loopback `host` rules that use `trust`.

For example:

```conf
local   all   all                  trust
host    all   all   127.0.0.1      trust
host    all   all   ::1            trust
```

Change the applicable rules to:

```conf
local   all   all                  scram-sha-256
host    all   all   127.0.0.1      scram-sha-256
host    all   all   ::1            scram-sha-256
```

**Important:** Do not blindly replace every occurrence of `trust`. Review the rules first, including replication rules, and change only the authentication rules you intend to secure.

### Step 4: Reload PostgreSQL configuration

After saving the changes, reload the configuration:

```bash
docker exec -it postgres_db psql -U postgres \
  -c "SELECT pg_reload_conf();"
```

This command requires a working PostgreSQL administrator login. If password authentication has already been enabled, you must provide valid credentials.

### Step 5: Test password authentication

```bash
docker exec -it postgres_db \
  psql -U asik -d ecom_db -W
```

Enter the user's actual PostgreSQL password when prompted.

**Important:** Make sure the user already has a valid password before switching its authentication rule to `scram-sha-256`. Otherwise, you may lock yourself out. Keep the backup so you can restore the previous configuration if necessary.

## 6. Delete All Data from a Table

There are two common ways to remove all rows from a table.

### Method 1: DELETE

```sql
DELETE FROM employe;
```

- Deletes all rows.
- Keeps the table structure.
- Does not automatically reset the identity/serial sequence.

### Method 2: TRUNCATE

```sql
TRUNCATE TABLE employe;
```

- Removes all rows quickly.
- Keeps the table structure.
- Does not automatically reset the identity/serial sequence.

### Method 3: TRUNCATE and Reset ID

```sql
TRUNCATE TABLE employe RESTART IDENTITY;
```

- Removes all rows.
- Keeps the table structure.
- Resets associated identity/serial sequences.

**Recommended for practice:** 

```sql
TRUNCATE TABLE employe RESTART IDENTITY;
```

**Warning:** These commands permanently remove table data unless you are able to roll back the transaction. Always check the table name before executing them.

## 7. Quick Command Summary

| Task | Command |
|---|---|
| Enter Docker Bash | `docker exec -it postgres_db bash` |
| Open PostgreSQL | `psql -U postgres` |
| List databases | `\l` |
| List users | `\du` |
| Connect to database | `\c database_name` |
| List tables | `\dt` |
| Show table structure | `\d table_name` |
| Exit PostgreSQL | `\q` |
| Exit Docker Bash | `exit` |
| Check password environment variable | `docker exec postgres_db printenv POSTGRES_PASSWORD` |
| Create user | `CREATE USER username WITH PASSWORD 'password';` |
| Create database | `CREATE DATABASE db_name OWNER username;` |
| Delete all table rows | `DELETE FROM table_name;` |
| Truncate table | `TRUNCATE TABLE table_name;` |
| Truncate and reset ID | `TRUNCATE TABLE table_name RESTART IDENTITY;` |

---

## Notes

- PostgreSQL commands such as `CREATE USER`, `CREATE DATABASE`, and `TRUNCATE` run inside `psql`.
- Commands such as `docker exec` run in your normal terminal.
- Commands beginning with `\` are PostgreSQL `psql` meta-commands, not regular SQL statements.
- Replace `postgres_db`, `username`, `database_name`, and table names with your actual values.
- Use strong, unique passwords for real projects.

**Environment:** Docker + PostgreSQL 18
