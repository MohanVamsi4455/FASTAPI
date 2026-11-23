# VM Setup & Deployment Commands — Explained

Step-by-step walkthrough of `VM_CMDS`, in the order the commands were run.

> **⚠️ Security note:** the original commands include a plaintext password
> (`Password=Mohan@123`). Rotate that password and avoid keeping real
> credentials in any file in the repo.

## 1. Connect to the EC2 instance

Connect as the default `ubuntu` user, using the PEM key for auth.

```bash
ssh -i "D:\Aws_PEM_KEYS\FASTAPI.pem" ubuntu@13.203.66.100
```

## 2. Root-level setup

Become root so the rest of the setup commands don't need `sudo` repeated.

```bash
sudo -i
```

- `adduser mohan` — create a new non-root user that will own and run the application.
- `sudo apt update && sudo apt upgrade -y` — refresh the package index and upgrade all installed packages.
- `sudo apt update` — refresh the package index again (e.g. after the upgrade, to pick up any new listings).
- `python3 --version` — confirm which Python 3 version ships with the OS.
- `sudo apt install python3-pip` — install pip (Python's package manager) for Python 3.
- `sudo apt install -y python3.12-venv` — install the venv module for Python 3.12 (needed to create isolated virtual environments).
- `sudo apt install postgresql postgresql-contrib -y` — install the PostgreSQL server and its extra contrib utilities.

```bash
adduser mohan
sudo apt update && sudo apt upgrade -y
sudo apt update
python3 --version
sudo apt install python3-pip
sudo apt install -y python3.12-venv
sudo apt install postgresql postgresql-contrib -y
```

## 3. Configure PostgreSQL

Configure PostgreSQL so it can be reached from outside localhost, and verify the install.

- `psql --version` — check the installed PostgreSQL client version.
- `sudo cat /etc/passwd` — inspect system users (sanity check, e.g. confirm the `postgres` OS user exists).
- `sudo -u postgres psql` — log into PostgreSQL as the `postgres` superuser (via OS-level "peer" auth).
- `ls /etc/postgresql/` — list the PostgreSQL config directories to find the right version folder.
- `cd /etc/postgresql/16/main` — move into the config folder for PostgreSQL 16.
- Edit `postgresql.conf`: set `listen_addresses='*'` so Postgres accepts connections on all network interfaces, not just localhost.
- Edit `pg_hba.conf`: change the auth method from `peer` to `md5` (password-based) and widen the allowed hosts, so remote/password logins are permitted.
- `systemctl restart postgresql` — restart PostgreSQL so the config changes take effect.
- `psql -U postgres` — log in again to confirm the new auth settings work.

```bash
psql --version
sudo cat /etc/passwd
sudo -u postgres psql
ls /etc/postgresql/
cd /etc/postgresql/16/main
sudo vi postgresql.conf       # add listen_addresses='*'
sudo vi pg_hba.conf           # change "peer" to "md5", allow all hosts
systemctl restart postgresql
psql -U postgres
```

## 4. Switch to the app user

The application will run under the `mohan` account from here on.

```bash
ssh mohan@13.203.66.100
```

- `mkdir mohan` — create a working folder (redundant with the home directory, kept for reference).
- `cd ~/app` — move into the app directory.
- `rm -rf venv` — remove any existing virtual environment so it can be rebuilt cleanly.
- `python3.12 -m venv venv` — create a fresh virtual environment using Python 3.12.
- `source venv/bin/activate` — activate the virtual environment.
- `python --version` — confirm the Python version being used inside the venv.
- `deactivate` — leave the virtual environment.

```bash
mkdir mohan
cd ~/app
rm -rf venv
python3.12 -m venv venv
source venv/bin/activate
python --version
deactivate
```

## 5. Pull the application source

- `mkdir src && cd src` — create a folder to hold the source code.
- `git clone https://github.com/Mohanvamsi4405/FASTAPI.git` — clone the FastAPI project repo.
- `cd ..` — go back up a level.
- `source venv/bin/activate` — re-activate the virtual environment to install dependencies into it.
- `pip install -r requirements.txt` — install all Python dependencies. If `psycopg2` fails to build, install the Postgres dev headers first: `sudo apt install -y libpq-dev`.

```bash
mkdir src
cd src
git clone https://github.com/Mohanvamsi4405/FASTAPI.git
cd ..
source venv/bin/activate
pip install -r requirements.txt
# if psycopg2 build fails:
sudo apt install -y libpq-dev
```

## 6. Set up environment variables

Set up the environment variables the app needs (DB credentials, secrets, etc.).

- `export MY_NAME="Mohan"` / `printenv` / `unset MY_NAME` / `printenv` — example of setting a one-off shell variable and confirming it appears/disappears from the env.
- `cd ~` — go to the home directory.
- `vi ~/.env` / `cat ~/.env` — create/edit the real `.env` file with the app's actual config values.
- `set -o allexport; source /home/mohan/.env; set +o allexport` — load every variable from `.env` into the current shell session (one-time, for this session).
- `ls -la` — confirm `.env` and `.profile` exist.
- Edit `.profile`: append the same `source .env` line so the env vars load automatically on every future login.

```bash
export MY_NAME="Mohan"
printenv
unset MY_NAME
printenv

cd ~
vi ~/.env
cat ~/.env

set -o allexport; source /home/mohan/.env; set +o allexport

ls -la
vi .profile   # append: set -o allexport; source /home/mohan/.env; set +o allexport
```

## 7. Apply database migrations

- `cd app/src/` — move into the cloned project's source folder (where `alembic.ini` lives).
- `alembic upgrade head` — run all pending Alembic migrations to bring the database schema up to date.

```bash
cd app/src/
alembic upgrade head
```

## 8. Run the app

First a quick manual check, then the real production-style server.

- `uvicorn app.main:app` — quick sanity-check run (single process, foreground, default settings).
- `pip install gunicorn` — install the process manager that will supervise multiple uvicorn worker processes. If it fails, install uvicorn's extra speed-up libs instead: `pip install uvtools httptools`.
- `gunicorn -w 4 -k uvicorn.workers.UvicornWorker app.main:app --bind 0.0.0.0:8000` — start the app for real: gunicorn manages 4 worker processes, each running the `UvicornWorker` class (so each worker can handle async FastAPI code), listening on all interfaces, port 8000.

```bash
uvicorn app.main:app

pip install gunicorn
# if it fails:
pip install uvtools httptools

gunicorn -w 4 -k uvicorn.workers.UvicornWorker app.main:app --bind 0.0.0.0:8000
```

## 9. Verify the server is running

- `ps -aef | grep -i gunicorn` — list running processes and filter for "gunicorn" to confirm the master + worker processes started.

```bash
ps -aef | grep -i gunicorn
```
