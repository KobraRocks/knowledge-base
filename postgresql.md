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
