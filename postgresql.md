# ✅ Manual PostgreSQL Setup Tutorial (non-root service)

This step-by-step guide explains how to manually set up and start PostgreSQL as a non-root user on Linux, including how to make it boot-persistent.

---

## 🧱 Prerequisites

* PostgreSQL binaries installed (`initdb`, `pg_ctl`, etc.)
* A `postgres` system user
* Root access (`sudo`)

---

## 🪜 Step-by-Step Setup

---

### **1. Create the data directory**

```bash
sudo mkdir -p /var/lib/postgres/data
sudo chown -R postgres:postgres /var/lib/postgres/data
sudo chmod 700 /var/lib/postgres/data
```

---

### **2. Initialize the PostgreSQL cluster**

```bash
sudo -u postgres initdb --locale=C.UTF-8 --encoding=UTF8 -D /var/lib/postgres/data --data-checksums
```

This sets up the system catalogs, configs, and directory structure.

---

### **3. Create the runtime socket/lock directory**

PostgreSQL requires this runtime folder to create its Unix socket and lock files:

```bash
sudo mkdir -p /run/postgresql
sudo chown postgres:postgres /run/postgresql
sudo chmod 775 /run/postgresql
```

---

### **4. Start PostgreSQL manually**

```bash
sudo -u postgres pg_ctl -D /var/lib/postgres/data -l /var/lib/postgres/logfile start
```

---

## ✅ PostgreSQL is now running!

You can now:

* Connect using `psql`:

  ```bash
  sudo -u postgres psql
  ```
* Stop the server manually:

  ```bash
  sudo -u postgres pg_ctl -D /var/lib/postgres/data stop
  ```

---

## 🔁 Bonus: Make it boot-persistent

Since `/run` is a **temporary filesystem (tmpfs)**, the `/run/postgresql` directory will **disappear at every reboot**, and PostgreSQL will fail to start unless it's recreated.

### ✅ Option 1: Use `systemd-tmpfiles` (recommended)

Create a config file to ensure the directory is auto-created at boot.

#### 1. Create the tmpfiles config:

```bash
sudo nano /etc/tmpfiles.d/postgresql.conf
```

#### 2. Add this content:

```
d /run/postgresql 0775 postgres postgres -
```

* `d`: create directory
* `0775`: permission
* `postgres postgres`: owner
* `-`: no timeout

#### 3. Reload `systemd-tmpfiles` and apply:

```bash
sudo systemd-tmpfiles --create
```

✅ From now on, `/run/postgresql` will be auto-created at boot with the right permissions.

---

### ✅ Option 2: Add a line in `rc.local` (fallback for non-systemd systems)

If your system uses `rc.local`, add:

```bash
mkdir -p /run/postgresql
chown postgres:postgres /run/postgresql
chmod 775 /run/postgresql
```

before your `pg_ctl start` command.

---

## 🧾 Final Summary of Commands

```bash
# Create data directory
sudo mkdir -p /var/lib/postgres/data
sudo chown -R postgres:postgres /var/lib/postgres/data
sudo chmod 700 /var/lib/postgres/data

# Initialize the DB
sudo -u postgres initdb --locale=C.UTF-8 --encoding=UTF8 -D /var/lib/postgres/data --data-checksums

# Create runtime directory
sudo mkdir -p /run/postgresql
sudo chown postgres:postgres /run/postgresql
sudo chmod 775 /run/postgresql

# Start the server
sudo -u postgres pg_ctl -D /var/lib/postgres/data -l /var/lib/postgres/logfile start

# Make /run/postgresql persistent across reboots
echo 'd /run/postgresql 0775 postgres postgres -' | sudo tee /etc/tmpfiles.d/postgresql.conf
sudo systemd-tmpfiles --create
```

To enable **remote access through localhost** for PostgreSQL so tools like **n8n** (running locally) can connect to it, follow these steps:

---

## ✅ Goal

Allow local apps (like n8n) to connect to PostgreSQL via:

```
host: localhost
port: 5432
```

---

## 🪜 Steps to Enable Access

### **1. Edit `postgresql.conf` to listen on localhost**

Find your config file — usually in the data directory:

```bash
sudo nano /var/lib/postgres/data/postgresql.conf
```

🔍 Find the line:

```conf
#listen_addresses = 'localhost'
```

✅ **Uncomment and ensure it says**:

```conf
listen_addresses = 'localhost'
```

> This makes PostgreSQL listen on the local network interface (`127.0.0.1`).

Save and exit (`Ctrl+O`, `Enter`, then `Ctrl+X`).

---

### **2. Edit `pg_hba.conf` to allow local authenticated TCP connections** (optional)

Still in the same directory, open:

```bash
sudo nano /var/lib/postgres/data/pg_hba.conf
```

🔽 Add this line **at the top** or near other `host` entries:

```conf
host    all             all             127.0.0.1/32            md5
```

This allows any local app (via `127.0.0.1`) to connect to any DB with password auth.

🔐 Optionally, for IPv6 localhost:

```conf
host    all             all             ::1/128                 md5
```

Save and exit.

---

### **3. Restart PostgreSQL to apply changes**

```bash
sudo -u postgres pg_ctl -D /var/lib/postgres/data restart
```

Or if it fails, use:

```bash
sudo -u postgres pg_ctl -D /var/lib/postgres/data stop
sudo -u postgres pg_ctl -D /var/lib/postgres/data -l /var/lib/postgres/logfile start
```

---

### **4. Create a PostgreSQL user and database (if needed)**

If n8n needs its own database:

```bash
sudo -u postgres psql
```

Inside `psql`, run:

```sql
CREATE USER n8nuser WITH PASSWORD 'n8npassword';
CREATE DATABASE n8ndb OWNER n8nuser;
```

Grant all privileges if needed:

```sql
GRANT ALL PRIVILEGES ON DATABASE n8ndb TO n8nuser;
\q
```

---

### ✅ n8n `.env` configuration (example)

In your n8n `.env` or configuration:

```env
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=localhost
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8ndb
DB_POSTGRESDB_USER=n8nuser
DB_POSTGRESDB_PASSWORD=n8npassword
```

---

## 🧪 Test the connection

You can test the connection from another terminal with:

```bash
psql -h localhost -U n8nuser -d n8ndb
```

If it connects, you're good to go 🚀


