# LetterRoll — Django 6.1 / Python 3.14

Self-hosted Gmail newsletter rollup using Django 6.1 for the dashboard and access control, with a separate async worker for real-time Gmail IMAP ingestion and the daily HTML LetterRoll.

## Architecture

- **Django web service**: LetterRoll dashboard, login/session authentication, CSRF protection, sender controls, Google OAuth flow, and Django admin.
- **Worker service**: Gmail IMAP/XOAUTH2 ingestion via IMAP IDLE plus the daily LetterRoll scheduler.
- **SQLite**: the only persistent database. WAL mode, a 30-second busy timeout, and a single worker keep the small self-hosted deployment simple and reliable.
- **Gmail OAuth token**: stored in `/data/google-token.json` with mode `0600`.

Django 6.1's built-in authentication system uses sessions and exposes `request.user`; the dashboard is protected with `login_required`. The Django admin is available to staff users.

## First run

1. Copy `.env.example` to `.env`.
2. Set a long random `DJANGO_SECRET_KEY`.
3. Set `DJANGO_ALLOWED_HOSTS` to the hostname(s) you will actually use.
4. Configure Google OAuth as described below.
5. Start the stack:

```bash
docker compose up -d --build
```

6. Create the first dashboard user:

```bash
docker compose exec web python manage.py createsuperuser
```

7. Open `http://YOUR-HOST:8080/` and sign in.
8. From the dashboard, choose **Connect Gmail**.

For a homelab, Django's normal username/password authentication is intentionally separate from Google's Gmail OAuth credential. The Django login controls who can access the dashboard; the Google credential controls access to the Gmail mailbox.

## Google OAuth setup

1. Create/select a Google Cloud project.
2. Configure the OAuth consent screen.
3. Create an OAuth 2.0 Client ID with application type **Web application**.
4. Add the exact value of `OAUTH_REDIRECT_URI` to the client's Authorized redirect URIs.
5. Put the client ID and secret in `.env`.
6. Set `GOOGLE_EMAIL` to the Gmail account being authorized.
7. Sign in to the LetterRoll dashboard.
8. Click **Connect Gmail** and complete Google's consent screen.

The app requests the Gmail `https://mail.google.com/` scope because the worker reads mail and modifies Gmail labels through IMAP, while the same OAuth credential is used for SMTP delivery.

For a public/reverse-proxied deployment, use an HTTPS redirect URI and set `DJANGO_SECURE_COOKIES=true`. Also include the public hostname in `DJANGO_ALLOWED_HOSTS`.

## Django access control

The dashboard requires an authenticated Django user. Sender changes, manual rollups, and Gmail OAuth are also behind authentication. Staff users can access `/admin/` to manage users and inspect sender/message records.

If this is a single-user homelab, one superuser is enough. If multiple people need access, create separate Django users rather than sharing a password.

## Behavior

- **Included**: label `LetterRoll/Included`; removed from Inbox when `ARCHIVE_NEWSLETTERS=true`; included in the next LetterRoll.
- **Excluded**: label `LetterRoll/Excluded`; removed from Inbox when archiving is enabled; not included in LetterRoll.
- **Review**: label `LetterRoll/Review`; stays in Inbox.
- **Normal**: not treated as a newsletter; stays in Inbox.

New senders are classified using newsletter signals such as `List-Unsubscribe`, `List-ID`, `Precedence`, and unsubscribe/preferences text. Dashboard decisions override the automatic classification.

## Important notes

This is a clean Django rewrite, so the previous database schema is not automatically migrated. The existing Google OAuth token in `/data/google-token.json` can remain in place. The production database is SQLite at `/data/letterroll.sqlite3`; no PostgreSQL or Redis service is required. The web and worker containers share the same `/data` volume.

SQLite backups should use the included management command so the live database is copied consistently:

```bash
docker compose exec web python manage.py backup_db
```

Backups are written under `/data/backups/` by default. Treat the backup and OAuth token as sensitive files.

For production Internet exposure, put Django behind HTTPS/reverse proxy, use a strong secret key, restrict `ALLOWED_HOSTS`, enable secure cookies, and do not use Django's development server as the public-facing server. The production compose file uses Gunicorn; do not expose Django's development server to the network.

## Dependency management

LetterRoll manages Python dependencies in three layers with `pip-tools`:

- `requirements.in` — runtime dependencies; compiled to `requirements.txt`.
- `requirements-test.in` — test dependencies layered on `requirements.txt`; compiled to `requirements-test.txt`.
- `requirements-dev.in` — development/formatting tools layered on `requirements-test.txt`; compiled to `requirements-dev.txt`.

The lock files are checked into version control. Use the Invoke tasks to regenerate the lock files and synchronize a clean development environment. Production images install only `requirements.txt`.

## Code formatting

Black and isort are development dependencies. Their project rules live in `pyproject.toml`: Black uses an 88-character line length and Python 3.14 target, while isort uses the Black profile and recognizes `accounts`, `letterroll`, and `letterroll_project` as first-party packages.

Run both before committing:

```powershell
inv format
```

## Testing

Install the development dependencies and run the complete suite:

```powershell
python -m pip install --upgrade pip-tools invoke
inv deps
inv sync
python manage.py migrate
inv test
```

Available Invoke tasks:

- `inv deps` — compile the runtime, test, and development lock files.
- `inv deps-upgrade` — upgrade dependency pins and recompile all lock files.
- `inv sync` — synchronize the current environment from `requirements-dev.txt`.
- `inv format` — run Black and isort.
- `inv format-check` — verify formatting without changing files.
- `inv test` — run the test suite.

Invoke works from PowerShell and other Windows shells as well as from Unix-like environments.

The suite includes unit tests for email parsing, classification/scoring, HTML sanitization, OAuth configuration, and model behavior; integration tests for Django authentication/CSRF, sender management, admin access, IMAP message ingestion/idempotency, LetterRoll delivery/marking, and deployment configuration.

## Traefik

See `deploy/traefik/` for a Docker Compose example. It uses Traefik's Docker provider with `exposedbydefault=false`, routes the Django web container by hostname, redirects HTTP to HTTPS, and uses a Let's Encrypt HTTP-01 resolver.

## Django version

LetterRoll targets **Django 6.1.1** and requires Python 3.14 or newer; the container uses the official Python 3.14 slim image. Django 6.1 also requires SQLite 3.37 or newer.

## User model

LetterRoll uses a project-owned `accounts.User` model derived from Django's `AbstractUser`. The model currently keeps Django's standard username-based authentication while making email addresses unique. `AUTH_USER_MODEL` is set from the start so application code uses `get_user_model()` rather than depending on Django's built-in user model.
