# Dev Test Accounts

Local development bypass for testing all three portals without Auth0 configured.

---

## Quick Start

1. Start the stack: `docker compose up`
2. Open [http://localhost:3000/dev-login](http://localhost:3000/dev-login)
3. Click the role you want to test — you are redirected to that portal immediately

---

## Accounts

| Role | Name | Email | Portal URL |
|---|---|---|---|
| Admin | — | `wegofwd2020@gmail.com` | `/admin/login` |
| Teacher | Dev Teacher | `dev.teacher@studybuddy.dev` | `/school/dashboard` |
| Student | Dev Student | `dev.student@studybuddy.dev` | `/student/dashboard` |

**Admin password:** `1304!Yosimite`
Teacher and student accounts have no password — use the `/dev-login` bypass page.

### Student details
- Grade: 8
- Locale: `en`
- Account status: `active`

### Teacher details
- Role: `teacher`
- School: Dev School (`school_id = 00000000-0000-0000-0000-000000000001`)
- Account status: `active`

---

## How It Works

### Backend — `POST /api/v1/auth/dev-login`

Accepts `{"role": "student" | "teacher"}`.

- Only registered when `APP_ENV=development` (guarded in `main.py`)
- Creates the school, teacher, and student records on first call (idempotent)
- Returns a **7-day JWT** + name + email

Source: `backend/src/auth/dev_router.py`

### Web — `/dev-login` page

Source: `web/app/(public)/dev-login/page.tsx`

On login:
1. Calls the backend dev-login endpoint
2. Stores the token in `localStorage` (`sb_token` for student, `sb_teacher_token` for teacher)
3. Sets a `sb_dev_session` cookie (base64 JSON `{name, email}`) for server-side layouts
4. Redirects to the portal

### Web — server layout fallback

Sources: `web/app/(student)/layout.tsx`, `web/app/(school)/layout.tsx`

Both layouts now do:
```
session = auth0.getSession() ?? getDevSession()
```

Auth0 takes priority. If Auth0 is not configured (local dev), the `sb_dev_session` cookie is used as a fallback. Helper: `web/lib/dev-session.ts`.

---

## Files Changed / Added

| File | Change |
|---|---|
| `backend/src/auth/dev_router.py` | New — dev-login endpoint |
| `backend/main.py` | Conditionally registers dev router |
| `web/lib/dev-session.ts` | New — server-side dev session cookie reader |
| `web/app/(student)/layout.tsx` | Added `getDevSession()` fallback |
| `web/app/(school)/layout.tsx` | Added `getDevSession()` fallback |
| `web/app/(public)/dev-login/page.tsx` | New — dev login UI |

---

## Security Notes

- The `/auth/dev-login` endpoint returns HTTP 403 if `APP_ENV != "development"` — it cannot be called in production even if the route were somehow registered
- The `sb_dev_session` cookie is not signed; it is display-only (name/email for the portal header). API authorization always uses the JWT in `localStorage`, verified server-side
- Tokens expire after 7 days; re-visit `/dev-login` to refresh
