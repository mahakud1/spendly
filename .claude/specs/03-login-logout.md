# Spec: Login and Logout

## Overview

Make sign-in and sign-out work. Today `GET /login` only renders a static form (whose POST has no handler) and `GET /logout` is a raw-string stub. This step adds a `POST /login` handler that verifies the email and password against the `users` table and stores the user's id in Flask's signed `session`, and turns `/logout` into a real route that clears the session and redirects. It also makes the shared navbar session-aware (Sign out when logged in, Sign in / Get started when not). It builds on Step 2 (registration creates the accounts that can now sign in) and is the prerequisite for the profile page (Step 4) and all expense routes (Steps 7-9), which need to know who the current user is.

## Depends on

- Step 1 — Database setup (`users` table, `get_db()`, `init_db()`, seeded demo user).
- Step 2 — Registration (`create_user()`, `get_user_by_email()`, flash message on `/login`).

## Routes

- `GET /login` — render the login form (already exists); if the user is already logged in, redirect to `landing` — public
- `POST /login` — validate credentials, set `session["user_id"]`, redirect to `landing` on success or re-render the form with an error and status 400 on failure — public
- `GET /logout` — clear the session, flash "You have been signed out.", redirect to `landing` — public (safe to call when not logged in)

`GET` and `POST` on `/login` are handled by the existing `login()` function in `app.py` (change the decorator to `methods=["GET", "POST"]`). `logout()` stays at `/logout` but is implemented properly.

No other stub routes (`/profile`, `/expenses/...`) are touched.

## Database changes

No database changes. The existing `users` table is sufficient.

New helper in `database/db.py` (no routes may contain SQL):

- `get_user_by_id(user_id)` — returns the `sqlite3.Row` for that id or `None`, using a parameterised query and `closing(get_db())`. Used to load the current user for templates.

`get_user_by_email()` already exists and is reused for credential lookup.

## Templates

- Create: none
- Modify:
  - `templates/login.html`
    - Change `action="/login"` to `action="{{ url_for('login') }}"` (no hardcoded URLs).
    - Re-populate the `email` input with the submitted value on error (never the password).
    - Keep the existing `{% if error %}` block and flash loop.
  - `templates/base.html`
    - In `.nav-links`, show a "Sign out" link (`url_for('logout')`) when a user is logged in, otherwise the existing "Sign in" and "Get started" links.
    - Use a `current_user` template variable supplied by a context processor (see below), not direct session access in the template.

## Files to change

- `app.py` — set `session` usage, accept POST on `/login`, implement `/logout`, add a `context_processor` that exposes `current_user` (via `get_user_by_id`), import `abort`/`session`/`check_password_hash` as needed
- `database/db.py` — add `get_user_by_id()`
- `templates/login.html` — see above
- `templates/base.html` — session-aware navbar

## Files to create

- None

## New dependencies

No new dependencies. `werkzeug.security.check_password_hash` and Flask's `session` are already available; `app.secret_key` is already configured.

## Validation and behaviour rules

Applied server-side in the route:

- `email`: stripped and lower-cased before lookup (matches how registration stores it)
- `password`: taken as-is (not stripped)
- Unknown email and wrong password produce the same generic error, "Invalid email or password.", so account existence is not leaked
- On failure: re-render `login.html` with `error` and `email` set, HTTP status 400. Do not return raw strings.
- On success: call `session.clear()` then set `session["user_id"] = user["id"]` (prevents session fixation), flash "Welcome back, <name>." and redirect via `url_for("landing")`
- Store only the user id in the session — never the password hash or other user fields
- If the session holds a `user_id` that no longer exists in the DB, `current_user` is `None` and the user is treated as logged out

## Rules for implementation

- No SQLAlchemy or ORMs
- Parameterised queries only — never f-strings in SQL
- Passwords verified with Werkzeug (`check_password_hash`); never log or echo the plain password
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- DB logic lives in `database/db.py` only; routes read the form, call helpers, and render or redirect
- Use `url_for()` for every link and redirect — never hardcode URLs
- Do NOT implement `/profile`, expense routes or any route protection decorators (Steps 4+); only login, logout and the navbar change
- Do not change the port (5001) or add packages
- Close DB connections (use `closing(get_db())`)

## Definition of done

- [ ] `GET /login` renders the form with status 200 when logged out
- [ ] Logging in as `demo@spendly.com` / `demo123` redirects (302) to `/` and the navbar then shows "Sign out" instead of "Sign in" / "Get started"
- [ ] A user created through `/register` can log in with the password they chose
- [ ] Email matching is case-insensitive (`DEMO@Spendly.com` logs in)
- [ ] A wrong password shows "Invalid email or password." with status 400 and does not log the user in
- [ ] An unknown email shows the identical "Invalid email or password." message
- [ ] On a failed attempt the email field keeps its value and the password field is empty
- [ ] The login form's action uses `url_for('login')` (no hardcoded `/login` in the template)
- [ ] `GET /login` while logged in redirects to `/`
- [ ] `GET /logout` while logged in clears the session, redirects (302) to `/`, shows the "signed out" flash and the navbar returns to "Sign in" / "Get started"
- [ ] `GET /logout` while logged out does not error and also redirects to `/`
- [ ] After logout, reusing the browser back button does not leave the navbar in a logged-in state on reload
- [ ] The session cookie contains only the user id (inspect via Flask's session, no password hash)
- [ ] Deleting the user row for a logged-in session makes the next page load behave as logged out without a 500 error
- [ ] The app starts without errors and `requirements.txt` is unchanged
