# Spec: Registration

## Overview

Make the existing `/register` page functional. Today `GET /register` only renders a static form, and the form's POST has no handler. This step adds a `POST /register` handler that validates the input, hashes the password, creates a row in `users`, on success the user is shown with a success message and sends the new user to the login page. It builds on the data layer from Step 1 and is the prerequisite for login and logout (Step 3). Sessions are NOT introduced here: registering does not log the user in.

## Depends on

- Step 1 — Database setup (`users` table, `get_db()`, `init_db()`).

## Routes

- `GET /register` — render the registration form (already exists, unchanged behaviour) — public
- `POST /register` — validate form data, create the user, redirect to `/login` on success or re-render the form with an error on failure — public

Both methods are handled by the existing `register()` function in `app.py` (change the decorator to `methods=["GET", "POST"]`).

## Database changes

No database changes. The existing `users` table (`id`, `name`, `email` UNIQUE, `password_hash`, `created_at`) is sufficient.

New helpers in `database/db.py` (no routes may contain SQL):

- `create_user(name, email, password)` — hashes the password with `generate_password_hash`, inserts the row with a parameterised query, returns the new user id. Lets `sqlite3.IntegrityError` propagate when the email already exists, or catches it and returns `None` (pick one and be consistent; the route must show the "email already registered" error either way).
- `get_user_by_email(email)` — returns the `sqlite3.Row` for that email or `None`. (Reused by login in Step 3.)

## Templates

- Create: none
- Modify:
  - `templates/register.html`
    - Change `action="/register"` to `action="{{ url_for('register') }}"` (no hardcoded URLs).
    - Re-populate `name` and `email` inputs with the submitted values on error (never the password).
    - Keep the existing `{% if error %}` block for showing errors.
    - Add `minlength="8"` to the password input to match the placeholder.

## Files to change

- `app.py` — accept POST on `/register`, add validation and redirect logic, import the new db helpers and `request`, `redirect`, `url_for`
- `database/db.py` — add `create_user()` and `get_user_by_email()`
- `templates/register.html` — see above

## Files to create

- None

## New dependencies

No new dependencies. `werkzeug.security` is already installed.

## Validation rules

Applied server-side in the route (client-side `required` attributes are not enough):

- `name`: stripped, non-empty
- `email`: stripped, lower-cased, non-empty, contains `@` (basic check, no regex library)
- `password`: at least 8 characters
- Email must not already exist → error "An account with that email already exists."
- On any failure: re-render `register.html` with `error` set and HTTP status 400 (422 is also acceptable; pick one). Do not return raw strings.

## Rules for implementation

- No SQLAlchemy or ORMs
- Parameterised queries only — never f-strings in SQL
- Passwords hashed with Werkzeug (`generate_password_hash`); never store or log the plain password
- Use CSS variables — never hardcode hex values
- All templates extend `base.html`
- DB logic lives in `database/db.py` only; the route just reads the form, calls helpers, and renders or redirects
- Use `url_for()` for every link and redirect
- Do NOT implement sessions, login, logout or any other stub route (Steps 3+)
- Do not change the port (5001)
- Close DB connections (use `closing(get_db())` like the existing helpers)

## Definition of done

- [ ] `GET /register` still renders the form with status 200
- [ ] Submitting valid name, email and password (8+ chars) creates a row in `users` and redirects (302) to `/login`
- [ ] The stored `password_hash` is a Werkzeug hash, not the plain password (inspect the DB)
- [ ] Registering the same email again re-renders the form with an "already exists" error and creates no second row
- [ ] Email uniqueness is case-insensitive (`A@x.com` then `a@x.com` is rejected)
- [ ] A password shorter than 8 characters shows an error and creates no row
- [ ] A blank or whitespace-only name shows an error and creates no row
- [ ] An email without `@` shows an error and creates no row
- [ ] On error, the name and email fields keep their values and the password field is empty
- [ ] The form's action uses `url_for('register')` (no hardcoded `/register` in the template)
- [ ] The demo user (`demo@spendly.com`) is unaffected and the app starts without errors
- [ ] `requirements.txt` is unchanged
