---
title: Getting Started
description: Run the Umucyo backend and both web apps on your computer.
---

<!-- informational-website-link -->
[![Visit the Sulwe informational website](assets/design/get.png)](https://sulwe-informational-website.vercel.app/)

[Visit the Sulwe informational website](https://sulwe-informational-website.vercel.app/)

# Getting Started

<div class="at-a-glance" markdown>

**In short:** start the backend first on port 8000, then each web app with `npm run dev`. Local work uses mock payments, so nothing is ever sent to M-Pesa.

**You need three terminals, four tools and about twenty minutes.** When you finish this page you will have a working cooperative, an administrator account, and a delivery you created yourself.

</div>

## How the three pieces fit together

Umucyo is three separate programs that talk to one database. Nothing works until the backend is running, because both web apps ask it for everything.

```mermaid
flowchart LR
    A["sulwe-dashboard<br/>port 3000"] -->|HTTP + token| B["sulwe-backend<br/>port 8000"]
    C["sulwe-frontend<br/>port 3001"] -->|HTTP + token| B
    B --> D[("PostgreSQL")]
    B --> E["MTN MoMo<br/>(mocked locally)"]
    B --> F["SMS provider<br/>(mocked locally)"]
```

| Piece | Who uses it | Runs on |
| --- | --- | --- |
| Backend | Nothing directly — it is the API | `8000` |
| Dashboard | Cooperative staff and administrators | `3000` |
| Agent app | Agents who register farmers and record deliveries | `3001` when running together |

The two web apps have no database of their own. If the backend is down, both apps load and then show errors. That is normal and it is the first thing to check when something looks broken.

## Get the code

Umucyo is split into three repositories:

| Repository | What it holds |
| --- | --- |
| [sulwe-backend](https://github.com/akirachix/sulwe-backend) | FastAPI backend |
| [sulwe-dashboard](https://github.com/akirachix/sulwe-dashboard) | Admin dashboard |
| [sulwe-frontend](https://github.com/akirachix/sulwe-frontend) | Agent app |

```bash
git clone https://github.com/akirachix/sulwe-backend.git
git clone https://github.com/akirachix/sulwe-dashboard.git
git clone https://github.com/akirachix/sulwe-frontend.git
```

Keep all three folders next to each other so you can move between them easily:

```bash
mkdir umucyo && cd umucyo
# run the three git clone commands above inside this folder
ls
# sulwe-backend  sulwe-dashboard  sulwe-frontend
```

## What you need

- Python 3.11 and [uv](https://docs.astral.sh/uv/)
- PostgreSQL
- Node.js and npm
- Git

### Install the tools

Check what you already have before installing anything:

```bash
python3 --version     # want 3.11.x
uv --version
psql --version
node --version        # 18 or newer
npm --version
```

Anything missing:

**macOS**

```bash
brew install python@3.11 uv postgresql@16 node
brew services start postgresql@16
```

**Ubuntu / WSL**

```bash
sudo apt update
sudo apt install python3.11 python3.11-venv postgresql nodejs npm
curl -LsSf https://astral.sh/uv/install.sh | sh
sudo service postgresql start
```

**Windows**

Install Python 3.11 and Node from their websites, PostgreSQL from the
installer, then uv from PowerShell:

```powershell
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

On Windows, run the rest of this page in **WSL** or Git Bash. Every command below is a shell command.

### Create the database

The backend expects a PostgreSQL database to exist before it starts. It does not create one for you.

```bash
# create a local role and database
createuser -s umucyo --pwprompt
createdb -O umucyo umucyo
```

Confirm you can connect:

```bash
psql -U umucyo -d umucyo -c "SELECT current_database();"
```

## 1. Backend (`sulwe-backend`)

1. Install the packages.
    ```bash
    uv sync
    ```
2. Create a `.env` file with the settings listed in [Configuration](configuration.md). Keep `DARAJA_SUBMISSION_MODE=mock`.
3. Create or update the database.
    ```bash
    uv run alembic heads          # must print exactly one head
    uv run alembic upgrade head
    ```
4. Start the server.
    ```bash
    uv run fastapi dev
    ```
5. Open the API documentation at `http://127.0.0.1:8000/docs`.

### Before step 2: the settings that must be right

A fresh `.env` copied from `.env.example` fails in one of two ways — a wrong database address, or a blank secret. The minimum to get running:

| Setting | Local value | Why |
| --- | --- | --- |
| `DATABASE_URL` | `postgresql://umucyo:umucyo@localhost:5432/umucyo` | Must point at the database you created above |
| `SECRET_KEY` | any long random string | Signs tokens; the backend refuses to start if it is empty |
| `JWT_SECRET_KEY` | any long random string | Signs agent-app tokens |
| `DARAJA_SUBMISSION_MODE` | `mock` | Keeps payments local |
| `ALLOW_SANDBOX_SUBMISSIONS` | `false` | Second guard against real payouts |

Generate a secret rather than inventing one:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(48))"
```

Full list: [Configuration](configuration.md).

!!! warning "Back up before changing the database"
    Before `alembic upgrade head` on a database that holds data, take a backup:
    `pg_dump -h localhost -U <db user> -Fc -d <database> -f backup.dump`

### Step 3: one head, then upgrade

```bash
uv run alembic heads
```

- **Exactly one line** — continue.
- **Two or more lines** — stop. Two developers created migrations from the same starting point and the schema is now ambiguous. Fix it before upgrading, as described in [Database](database.md).

Then:

```bash
uv run alembic current    # what is applied now
uv run alembic upgrade head
uv run alembic current    # should now match head
```

### Step 4: what a good start looks like

```bash
uv run fastapi dev
```

You should see the server bind to `127.0.0.1:8000` and mention reload. Then, in another terminal:

```bash
curl -s http://127.0.0.1:8000/api/v1/health
```

A healthy response returns `200` with a status body. If instead you get `connection refused`, the server is not running. If you get a `500`, read the terminal — almost always `DATABASE_URL`.

!!! tip "The API documentation is the fastest way to check a change"
    Open `http://127.0.0.1:8000/docs`. Every route is listed with its request shape. Sign in with the same token the apps use and you can call anything by hand, without touching a browser.

## 2. Admin dashboard (`sulwe-dashboard`)

1. Install the packages.
    ```bash
    npm install
    ```
2. Create `.env.local` with the backend address (see [Configuration](configuration.md#admin-dashboard)).
3. Start it and open `http://localhost:3000`.
    ```bash
    npm run dev
    ```

The one variable that matters on day one:

```bash
NEXT_PUBLIC_API_URL=http://127.0.0.1:8000
```

!!! warning "`NEXT_PUBLIC_` values are baked in at build time"
    Changing `.env.local` while `npm run dev` is running is not enough for production builds — rebuild. In development the dev server picks it up on restart. This is the single most common "why is it calling the wrong backend" cause.

## 3. Agent app (`sulwe-frontend`)

Follow the same steps as the dashboard in the `sulwe-frontend` folder. If both apps run at the same time, start the second one on another port: `npm run dev -- -p 3001`.

```bash
cd ../sulwe-frontend
npm install
# .env.local with the same NEXT_PUBLIC_API_URL
npm run dev -- -p 3001
```

The agent app is the one to open on a phone or in a narrow browser window. On a real phone, bind to all interfaces so the device can reach your laptop:

```bash
npm run dev -- -p 3001 --hostname 0.0.0.0
```

Then open `http://<your-laptop-ip>:3001` on the phone, on the same network.

## 4. First login

1. Create a cooperative and its first administrator (see [User registration](features/registration.md)).
2. Log in with your email and password.
3. Enter the 6-digit code sent to your email.

### Where the first administrator comes from

There is no public sign-up — that is deliberate. Someone has to seed the first cooperative and administrator, either:

- through a seed script in the backend, or
- through the Django-style admin or a one-off script run once on your machine.

Once that account exists, every other account is created from inside the dashboard. If you are unsure whether a seed exists in your checkout, look for a script under the backend's scripts or seed folder, or ask the team rather than creating a user by hand in SQL — a hand-written row usually gets the password hash wrong.

!!! tip "Local email codes"
    In local development the one-time code is either printed in the backend terminal or written to the mail log, depending on your `.env`. If no email arrives, look at the backend output first, not your inbox.

## Running everything at once

Three terminals, started in this order:

| Terminal | Folder | Command | Address |
| --- | --- | --- | --- |
| 1 | `sulwe-backend` | `uv run fastapi dev` | `http://127.0.0.1:8000` |
| 2 | `sulwe-dashboard` | `npm run dev` | `http://localhost:3000` |
| 3 | `sulwe-frontend` | `npm run dev -- -p 3001` | `http://localhost:3001` |

Start the backend first. The web apps retry, but you will see avoidable errors if you open them before the API is up.

## Check your setup in five minutes

Work down this list once. If all nine pass, your environment is sound and any later problem is a code problem, not a setup problem.

1. `uv run alembic current` matches `uv run alembic heads`.
2. `http://127.0.0.1:8000/docs` loads.
3. The health endpoint returns `200`.
4. The dashboard loads at `3000` and the sign-in form renders.
5. Signing in with the seeded administrator sends a code.
6. The code arrives (backend terminal or mail log) and logging in succeeds.
7. A collection point can be created and appears in the list.
8. The agent app loads at `3001` and shows its task list.
9. A delivery saved twice with the same idempotency key produces **one** record, not two.

## Every day after this

You do not repeat the setup. The short loop is:

```bash
# terminal 1
cd sulwe-backend && uv run fastapi dev

# terminal 2
cd sulwe-dashboard && npm run dev

# terminal 3
cd sulwe-frontend && npm run dev -- -p 3001
```

After pulling new code, three commands catch most surprises:

```bash
git pull
uv sync                              # backend only: new packages
uv run alembic upgrade head          # backend only: new migrations
npm install                          # each web app: new packages
```

## When it does not work

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `connection refused` on `8000` | Backend not started | Start terminal 1 |
| `500` on every route | Bad `DATABASE_URL` | Check the backend terminal for the psycopg error |
| `password authentication failed` | Wrong role or password in the URL | Re-check the `createdb` / `createuser` step |
| `Target database is not up to date` | Migrations behind | `uv run alembic upgrade head` |
| `Multiple head revisions` | Two migrations branched | Fix the heads before upgrading |
| Login succeeds, every page shows `401` | Token rejected — usually `SECRET_KEY` changed | Sign in again; clear stored tokens |
| Dashboard calls the wrong host | `NEXT_PUBLIC_API_URL` wrong, or built earlier | Fix `.env.local`, restart or rebuild |
| App loads but lists are empty | Backend reachable, request failed | Open the browser network tab before assuming no data |
| `Module not found` after a pull | Packages out of date | `uv sync` and `npm install` |
| Port already in use | Another copy still running | Stop it, or use `-p 3002` |

!!! tip "Restart after changing settings"
    The backend reads `.env` and the web apps read `.env.local` only when they start. Restart them after every change.

## Where to go next

| If you want to… | Read |
| --- | --- |
| Understand how the pieces are wired | [Architecture](architecture.md) |
| Know every setting and its default | [Configuration](configuration.md) |
| Work on the API | [Backend](backend.md) and [API](api.md) |
| Change the schema | [Database](database.md) |
| Ship something | [Deployment](deployment.md) |
| Use the dashboard or agent app | [Dashboard](frontend-dashboard.md), [PWA](frontend-PWA.md) |
