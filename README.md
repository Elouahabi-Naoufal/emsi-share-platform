<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->
<a id="readme-top"></a>



<!-- PROJECT SHIELDS -->
<!--
*** I'm using markdown "reference style" links for readability.
*** Reference links are enclosed in brackets [ ] instead of parentheses ( ).
*** See the bottom of this document for the declaration of the reference variables
*** for contributors-url, forks-url, etc. This is an optional, concise syntax you may use.
*** https://www.markdownguide.org/basic-syntax/#reference-style-links
-->
[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]



<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE">
    <h3 align="center">EMSI Share Learn</h3>
  </a>

  <p align="center">
    A collaborative learning platform for schools and universities — rooms, resources,
    quizzes, forums, events, assignments, gradebook, attendance, timetable, and messaging,
    on Django REST Framework and a React 18 + TypeScript front end.
    <br />
    <a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE"><strong>Explore the source »</strong></a>
    <br />
    <br />
    <a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE/tree/main/backend">Backend</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE/tree/main/src">Frontend</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE/issues/new?labels=bug">Report Bug</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE/issues/new?labels=enhancement">Request Feature</a>
  </p>
</div>



<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#architecture">Architecture</a></li>
        <li><a href="#roles-and-permissions">Roles and Permissions</a></li>
        <li><a href="#repository-layout">Repository Layout</a></li>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
        <li><a href="#configuration">Configuration</a></li>
      </ul>
    </li>
    <li>
      <a href="#usage">Usage</a>
      <ul>
        <li><a href="#typical-flows">Typical Flows</a></li>
        <li><a href="#api-overview">API Overview</a></li>
        <li><a href="#deployment">Deployment</a></li>
      </ul>
    </li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li>
      <a href="#contributing">Contributing</a>
      <ul>
        <li><a href="#top-contributors">Top contributors:</a></li>
      </ul>
    </li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>



<!-- ABOUT THE PROJECT -->
## About The Project

**EMSI Share Learn** is a learning management system built for schools and universities. The unit
of organisation is a *room*: a teacher creates one for a subject, students join it with a code,
and everything inside it — resources, quizzes, forum topics, events, assignments — is scoped to
that room. Around that sit platform-wide features that are not room-bound: messaging,
announcements, an academic calendar, certificates, a course catalog, an audit log, and an
administration panel with database and SSH configuration views.

Two engineering decisions shape most of the code:

**Binary storage in PostgreSQL.** Uploaded files, avatars, profile pictures, certificates, and
event media are stored in `BinaryField` columns rather than on the filesystem. This keeps a
deployment to two stateful services — the database and Redis — and makes permissions travel with
the row, but it means file bytes ride through Django views and, in several serializers, through
base64 in JSON. `PRODUCTION_CHECKLIST.md` items 6 and 17 call this out as the main scaling
problem.

**JWT in HttpOnly cookies.** `SIMPLE_JWT` is configured with `AUTH_COOKIE`/`AUTH_COOKIE_REFRESH`
(`emsi_access` / `emsi_refresh`) and the front end reads the access token out of
`document.cookie` and sends it as a `Bearer` header. Access tokens last one day, refresh tokens
fourteen, and refresh tokens are not rotated.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Architecture

```
+--------------------------------------------------------------+
|  React 18 + TypeScript  (Vite, Tailwind, Radix, TanStack Q)   |
|  36 named routes + catch-all: landing, auth, rooms,           |
|  resources, quizzes, forum, events, assignments, gradebook,   |
|  attendance, timetable, messages, announcements, calendar,    |
|  certificates, audit logs, progress, parent portal, 2FA       |
+-------------------------------+------------------------------+
                                |  HTTP / JSON
                                |  VITE_API_URL + Bearer JWT
+-------------------------------v------------------------------+
|  Django 5.2 + DRF 3.15 + SimpleJWT 5.5                       |
|  Daphne ASGI server - channels 4 + channels-redis             |
|                                                              |
|  users  rooms  resources  quizzes  forums  events             |
|  notifications  platform_settings  assignments  gradebook     |
|  attendance  timetable  messaging  announcements              |
|  calendar_app  certificates  audit_logs                       |
+--------------+--------------------------------+---------------+
               |                                |  WebSocket groups
               v                                v
     +----------------------+        +----------------------------+
     |  PostgreSQL          |        |  Redis (channel layer)     |
     |  incl. binary files  |        +----------------------------+
     +----------------------+
```

Realtime messaging and notification delivery go through Django Channels over a
`channels_redis` layer; the front end connects in `src/hooks/useChatSocket.ts` and
`src/hooks/useNotificationSocket.ts`.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Roles and Permissions

`User.ROLE_CHOICES` in `backend/users/models.py` defines nine roles. The model groups them
into three sets that permission checks use:

```python
STAFF_ROLES      = ['teacher', 'librarian', 'counselor', 'coordinator', 'staff']
ADMIN_ROLES      = ['admin', 'administration']
PRIVILEGED_ROLES = STAFF_ROLES + ADMIN_ROLES
```

| Role | Purpose |
| --- | --- |
| `student` | Default on self-registration. Joins rooms with a code, submits work, posts in forums |
| `teacher` | Creates and owns rooms, quizzes, events; grades; sees room analytics |
| `librarian` | Library resource management |
| `counselor` | Student guidance and support |
| `coordinator` | Programme / department coordination |
| `staff` | General administrative and IT staff |
| `admin` | Platform administrator |
| `administration` | Institutional management, full user CRUD |
| `parent` | Guardian view of a linked student |

`staff_title` and `department` are free-text fields on `User` for staff-organisational
display. Email is the `USERNAME_FIELD` and is unique, so login is by email.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Repository Layout

```
.
├── src/                              # React front end
│   ├── pages/                        # one component per route
│   ├── components/
│   │   ├── ui/                       # shadcn/ui primitives
│   │   ├── layout/                   # Header, Sidebar, MainLayout, NotificationDropdown
│   │   ├── dashboard/                # Student / Teacher / Administration dashboards
│   │   ├── resources/  forum/  events/  rooms/  editor/  admin/
│   ├── contexts/                     # AuthContext, PlatformContext, ThemeContext
│   ├── services/                     # api.ts, api_modified.ts (duplicate)
│   ├── hooks/                        # useChatSocket, useNotificationSocket, use-mobile, use-toast
│   ├── i18n/locales/                 # en.json, fr.json, ar.json
│   └── lib/                          # markdown.ts, utils.ts (cn)
│
├── backend/                          # Django project
│   ├── manage.py
│   ├── requirements.txt
│   ├── entrypoint.sh                 # migrate → bootstrap superuser → daphne
│   └── <app>/                        # one directory per domain, each with models/views/serializers/urls
│
├── Dockerfile                        # node:20-alpine build → nginx:alpine
├── nginx.conf                        # SPA fallback: try_files $uri /index.html
├── docker-compose.yml                # frontend + backend + redis
├── PRODUCTION_CHECKLIST.md           # 25 numbered security / performance findings
├── ROADMAP.md                        # feature history (largely shipped by now)
└── SECURITY_UPDATES.md               # dependency advisories and mitigations
```

Two things worth knowing before you read further:

- `backend/analytics/` exists on disk but is **not** in `INSTALLED_APPS` and has no `urls.py`.
  It is a leftover scaffold; nothing routes to it.
- `.vs/` contains a Visual Studio workspace state directory and a `.wsuo` file, and there are
  five PDF viewer components in `src/components/resources/` (`PDFViewer`, `PDFViewerSafe`,
  `SimplePDFViewer`, `ModernPDFViewer`, `UniversalFileViewer`). See the Roadmap.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



### Built With

* [![React][React.js]][React-url]
* [![TypeScript][TypeScript]][TypeScript-url]
* [![Vite][Vite]][Vite-url]
* [![Tailwind CSS][TailwindCSS]][TailwindCSS-url]
* [![Radix UI][Radix]][Radix-url]
* [![TanStack Query][TanStack-Query]][TanStack-Query-url]
* [![Django][Django]][Django-url]
* ![![DRF][DRF]][DRF-url]
* [![PostgreSQL][PostgreSQL]][PostgreSQL-url]
* [![Redis][Redis]][Redis-url]
* [![Channels][Channels]][Channels-url]
* [![Daphne][Daphne]][Daphne-url]
* [![Docker][Docker]][Docker-url]
* [![nginx][NGINX]][NGINX-url]
* [![i18next][i18next]][i18next-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- GETTING STARTED -->
## Getting Started

The intended development loop is two processes: `manage.py runserver` (or `daphne`) on the
backend and `npm run dev` on the front end, both talking to the same PostgreSQL database.
Redis is only needed for the WebSocket features.

### Prerequisites

* Node.js 18+ and npm (or bun — a `bun.lockb` is committed alongside `package-lock.json`)
* Python 3.10+ — `backend/Dockerfile` uses `python:3.11-slim`
* PostgreSQL 14+ with a database the app can write to
* Redis 7 — only required for Channels / WebSockets, and in `docker-compose.yml`

### Installation

1. Clone the repo
   ```sh
   git clone https://github.com/Elouahabi-Naoufal/EMSI-SHARE.git
   cd EMSI-SHARE
   ```

2. Backend
   ```sh
   cd backend

   python3 -m venv env
   source env/bin/activate          # env\Scripts\activate on Windows
   pip install -r requirements.txt

   # Create the database
   createdb emsi_share_db

   # Point Django at it (see Configuration below), then:
   python manage.py migrate
   python manage.py createsuperuser
   ```

   If you want to see the platform's own fixtures for rooms and forum categories first:
   ```sh
   python manage.py create_forum_categories
   python forums/populate_categories.py      # or run it directly
   ```

3. Frontend, in a second terminal (from the repository root)
   ```sh
   npm install

   cp .env.development .env.local
   npm run dev
   ```

4. Open the app
   | URL | What |
   | --- | --- |
   | <http://localhost:8080> | Front end — Vite dev server, port set in `vite.config.ts` |
   | <http://127.0.0.1:8000> | Django API |
   | <http://127.0.0.1:8000/admin/> | Django admin |

### Configuration

**Frontend.** One variable, read in `src/services/api.ts` as
`import.meta.env.VITE_API_URL || 'http://127.0.0.1:8000/api'`:

| File | Value |
| --- | --- |
| `.env.development` | `VITE_API_URL=http://127.0.0.1:8001/api` |
| `.env.production` | `VITE_API_URL=http://10.42.0.22:8000/api` |

Both committed values need changing for your environment: the development port is `8001` while
the Django dev server defaults to `8000`, and the production file holds a private LAN address
rather than a public URL.

**Backend.** `backend/backend_project/settings.py` reads these from the environment, falling
back to `backend/db_config.json` (git-ignored) and then to hard-coded defaults:

| Variable | Default | Notes |
| --- | --- | --- |
| `SECRET_KEY` | a committed insecure literal | Also signs the JWTs via `SIMPLE_JWT.SIGNING_KEY` |
| `DEBUG` | `'True'` | Compared as the *string* `'True'` |
| `DB_NAME` | `emsi_share_db` | |
| `DB_USER` | `postgres` | |
| `DB_PASSWORD` | `0000` | |
| `DB_HOST` | `localhost` | |
| `DB_PORT` | `5432` | |
| `REDIS_HOST` / `REDIS_PORT` | `127.0.0.1` / `6379` | Channels layer |

```sh
export SECRET_KEY="$(python3 -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())')"
export DEBUG=False
export DB_NAME=emsi_share_db DB_USER=postgres DB_PASSWORD='…' DB_HOST=localhost DB_PORT=5432
export REDIS_HOST=127.0.0.1 REDIS_PORT=6379

python manage.py runserver
```

`entrypoint.sh`, used by the container, additionally expects `SUPERUSER_EMAIL`,
`SUPERUSER_USERNAME`, and `SUPERUSER_PASSWORD`, and creates an `admin`-role superuser on first
boot if that email does not already exist.

Three settings shipped in the repository are development-only and are listed as critical in
`PRODUCTION_CHECKLIST.md`: `ALLOWED_HOSTS = ['*']`, `CORS_ALLOW_ALL_ORIGINS = True`, and
`AUTH_COOKIE_SECURE = False`. Tighten all three before exposing the API.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- USAGE EXAMPLES -->
## Usage

### Typical Flows

**Create a room and share a resource.** A teacher signs in, opens **Rooms**, creates a room with
a name, subject, and visibility, and shares the join code. Once students join with that code, the
teacher uploads a file from the room's Resources tab. Teacher uploads publish immediately;
student uploads land in a pending state (`src/components/admin/PendingResourcesPanel.tsx` on the
admin side, `MyPendingResourcesPanel.tsx` on the owner side) until approved.

**Preview a resource in the browser.** Resources are classified by type and rendered by the
matching viewer in `src/components/resources/` — `react-pdf` for PDFs, `@react-pdf-viewer` for
the other document types, `marked` + `highlight.js` for Markdown, `katex` for maths, `xlsx` for
spreadsheets.

**Run a quiz.** A teacher creates a quiz and its questions from the room; students take it from
**Quiz**. Scores and attempts are stored server-side and surface in **Analytics** and in the
teacher dashboard.

**Grade and track.** Assignments are created per room, students submit files or text, and the
teacher grades in **Gradebook**. Attendance sessions are opened per class or event in
**Attendance**. **Progress Report** aggregates attendance, quiz scores, assignments, resources
accessed, and forum participation for a student.

**Moderate a discussion.** The forum supports topics, posts, replies, up/down voting, a likes
table, marked solutions, subscriptions, view counts, and file attachments — implemented across
six migrations in `backend/forums/migrations/`.

**Translate the interface.** `src/i18n/index.ts` registers English, French, and Arabic with
`i18next-browser-languagedetector`, detecting from `localStorage` first and falling back to the
browser language; Arabic resources are present but RTL layout support is not something the
repository claims to have solved.

### API Overview

Everything lives under `/api/`. DRF `DefaultRouter` viewsets:

| Path | Model |
| --- | --- |
| `rooms/`, `resources/`, `resource-categories/`, `quizzes/`, `questions/` | Rooms, resources, quizzes |
| `forums/categories/`, `forums/topics/`, `forums/posts/`, `forums/attachments/` | Forum |
| `notifications/` | Notifications |

Plus plain `path()` routes for auth, users, and the per-app `urls.py` modules:

| Path | Method | Purpose |
| --- | --- | --- |
| `/api/token/` | POST | Obtain JWT pair (sets `emsi_access` / `emsi_refresh` cookies) |
| `/api/token/refresh/` | POST | Refresh the access token |
| `/api/auth/register/` | POST | Register |
| `/api/auth/me/` | GET/PATCH | Current profile |
| `/api/auth/forgot-password/`, `/api/auth/reset-password/` | POST | Password recovery |
| `/api/auth/logout/` | POST | Clear cookies |
| `/api/auth/verify-email/` | POST | Email verification |
| `/api/auth/export-data/` | GET | GDPR-style data export |
| `/api/auth/progress/`, `/api/auth/progress/<id>/` | GET | Student progress report |
| `/api/auth/2fa/setup/`, `/api/auth/2fa/verify/` | POST | TOTP two-factor (pyotp + qrcode) |
| `/api/auth/parent/` | GET | Parent portal |
| `/api/auth/profile/picture/` | GET/POST/DELETE | Profile picture |
| `/api/users/` | GET | List users (privileged roles) |
| `/api/users/create/`, `/api/users/bulk-import/` | POST | Admin user creation, CSV import |
| `/api/users/<id>/`, `/api/users/<id>/update/` | DELETE/PATCH | Admin user management |
| `/api/events/` | * | Events (own `urls.py`) |
| `/api/platform/public/` | GET | Branding, unauthenticated |
| `/api/platform/settings/` | GET/POST | Platform settings (authenticated) |
| `/api/platform/database/`, `/api/platform/ssh/` | GET/POST | DB config, SSH config, test and command execution (paramiko) |
| `/api/assignments/`, `gradebook/`, `attendance/`, `timetable/`, `messaging/`, `announcements/`, `calendar/`, `certificates/`, `audit-logs/` | * | One app each |

```sh
# Log in (email is the username field)
curl -X POST http://127.0.0.1:8000/api/token/ \
  -H 'Content-Type: application/json' \
  -c cookies.txt \
  -d '{"email": "teacher@example.com", "password": "…"}'

# Authenticated call using the stored access token
curl http://127.0.0.1:8000/api/rooms/ -H "Authorization: Bearer $(grep emsi_access cookies.txt | awk '{print $7}')"
```

The Django admin at `/admin/` is registered per app — `users`, `resources`, `forums`,
`audit_logs`, and `platform_settings` all ship an `admin.py`.

### Deployment

The committed Docker setup builds the front end with Node 20 and serves `dist/` from
`nginx:alpine`, with an SPA fallback in `nginx.conf`, and runs the backend on Daphne behind
`backend/entrypoint.sh`.

```sh
export SECRET_KEY=… DEBUG=False
export DB_NAME=… DB_USER=… DB_PASSWORD=… DB_HOST=… DB_PORT=5432

docker compose up --build
```

Two caveats, both verifiable in the files:

- `docker-compose.yml` uses `expose:` rather than `ports:`, so nothing is published to the host.
  Add port mappings for the frontend (80) and backend (8000) to reach them from outside the
  Docker network.
- No PostgreSQL service is defined in `docker-compose.yml` — only `frontend`, `backend`, and
  `redis`. `DB_HOST` has to point at a PostgreSQL you provide. (`backend/analytics/migrations/`
  and `backend/audit_logs/` suggest an earlier Postgres service was part of this file.)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ROADMAP -->
## Roadmap

`ROADMAP.md` is out of date: most of the features it lists as Priority 1 and 2 now have a
Django app and a front-end route behind them. The list below reflects the repository as it
stands.

Shipped:

- [x] Rooms with join codes and participant roles
- [x] Resources with in-database binary storage, categories, search, and an approval workflow
- [x] Quizzes with questions, attempts, and scoring
- [x] Forum: categories, topics, posts, votes, likes, subscriptions, attachments
- [x] Events with attendees, collaborators, and media
- [x] Assignments and submissions, gradebook, attendance, timetable
- [x] Messaging over WebSockets, announcements, academic calendar, certificates
- [x] Bulk CSV user import, email verification, password reset, TOTP 2FA, parent portal
- [x] Audit log and GDPR data export
- [x] i18n with English, French, and Arabic locale files
- [x] Docker build for the front end and Daphne entrypoint for the back end

Open work, derived from the code:

- [ ] Pick one package manager. Three dependency artefacts are committed — `package-lock.json`,
      `bun.lockb`, and `package.json.backup` — and only `package-lock.json` is used by the
      committed `Dockerfile` (`npm ci`).
- [ ] Replace the committed `SECRET_KEY`, set `DEBUG=False`, restrict `ALLOWED_HOSTS`, set
      `CORS_ALLOWED_ORIGINS`, and flip `AUTH_COOKIE_SECURE` for any shared environment.
      `PRODUCTION_CHECKLIST.md` items 1, 2, 3, 7, and 8.
- [ ] Fix the port and host values in `.env.development` (8001) and `.env.production`
      (a private LAN address), and replace the hard-coded `http://127.0.0.1:8000` fallbacks in
      `AuthContext.tsx`, `CourseCatalog.tsx`, `ParentPortal.tsx`, `TwoFactorSettings.tsx`, and
      `ProgressReport.tsx` with `import.meta.env.VITE_API_URL`.
- [ ] Add `Pillow` to `backend/requirements.txt`. `User.avatar` is an `ImageField`, which
      Django cannot migrate or serve without Pillow installed; it is not currently pinned.
- [ ] Delete the dead duplicates: `src/services/api_modified.ts` and
      `src/components/resources/ResourceUploadDialog_modified.tsx`. Consolidate the five PDF
      viewer components down to one — `SECURITY_UPDATES.md` already records that
      `PDFViewerSafe.tsx` supersedes the `@react-pdf-viewer`-based `PDFViewer.tsx`.
- [ ] Publish port mappings for the frontend and backend in `docker-compose.yml`, and add a
      PostgreSQL service or document the external one.
- [ ] Add throttle classes and a default pagination class to `REST_FRAMEWORK`
      (`PRODUCTION_CHECKLIST.md` items 4 and 15), and add `select_related` / `prefetch_related`
      to the room, resource, and quiz querysets (item 16).
- [ ] Add `db_index=True` to the frequently filtered columns called out in item 20
      (`Resource.status`, `Notification.recipient`, `AuditLog.action`,
      `AttendanceRecord.student`).
- [ ] Sanitise rich-text fields. `dompurify` is a front-end dependency, but
      `backend/forums/models.py` and `backend/assignments/models.py` store post and assignment
      bodies as-is (`PRODUCTION_CHECKLIST.md` item 13).
- [ ] Remove `backend/analytics/` or register it — it has migrations and models but no entry in
      `INSTALLED_APPS` and no URLs.
- [ ] Remove the committed `.vs/` workspace directory.
- [ ] Plagiarism detection on assignment submissions and live-session (Jitsi) integration — both
      described in `ROADMAP.md` and neither present in the code.

See the [open issues](https://github.com/Elouahabi-Naoufal/EMSI-SHARE/issues) for a full list of proposed features (and known issues).

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

Before opening a pull request:

- Front end: `npm run lint` and `npm run build` (the build runs `tsc` first, so type errors
  fail the build).
- Back end: `python manage.py check` and `python manage.py migrate --check`.
- Keep new back-end endpoints under `backend/<app>/urls.py` and register them in
  `backend/backend_project/urls.py`; each domain app owns its own views, serializers, and
  migration folder.
- Follow the existing role constants in `users.User` rather than introducing a parallel
  permission check.

### Top contributors:

<a href="https://github.com/Elouahabi-Naoufal/EMSI-SHARE/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=Elouahabi-Naoufal/EMSI-SHARE" alt="contrib.rocks image" />
</a>

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- LICENSE -->
## License

No license file is currently bundled with this repository. Until one is added, the default GitHub
terms apply and the repository should be treated as unlicensed — do not redistribute it without
asking.

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- CONTACT -->
## Contact

Naoufal Elouahabi - [LinkedIn](https://www.linkedin.com/in/elouahabi-naoufal/) - elouahabi-naoufal1@gmail.com

Project Link: [https://github.com/Elouahabi-Naoufal/EMSI-SHARE](https://github.com/Elouahabi-Naoufal/EMSI-SHARE)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

* Built as a final-year project (PFA) at EMSI — École Marocaine des Sciences de l'Ingénieur.
* [Django](https://www.djangoproject.com/) and [Django REST Framework](https://www.django-rest-framework.org/)
* [djangorestframework-simplejwt](https://github.com/jazzband/djangorestframework-simplejwt),
  [django-cors-headers](https://github.com/adamchainz/django-cors-headers), and
  [django-filter](https://django-filter.readthedocs.io/)
* [Django Channels](https://channels.readthedocs.io/) with a Redis channel layer, served by Daphne
* [paramiko](https://www.paramiko.org/) for the SSH management views and
  [pyotp](https://github.com/lepture/pyotp) / [qrcode](https://github.com/lincolnloop/python-qrcode)
  for TOTP two-factor
* [Vite](https://vite.dev/), [Tailwind CSS](https://tailwindcss.com/), [Radix UI](https://www.radix-ui.com/),
  and [shadcn/ui](https://ui.shadcn.com/) — `components.json` is the shadcn config
* [TanStack Query](https://tanstack.com/query), [Recharts](https://recharts.org/), and
  [i18next](https://www.i18next.com/)
* [Img Shields](https://shields.io)
* [Choose an Open Source License](https://choosealicense.com)

<p align="right">(<a href="#readme-top">back to top</a>)</p>



<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/Elouahabi-Naoufal/EMSI-SHARE.svg?style=for-the-badge
[contributors-url]: https://github.com/Elouahabi-Naoufal/EMSI-SHARE/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/Elouahabi-Naoufal/EMSI-SHARE.svg?style=for-the-badge
[forks-url]: https://github.com/Elouahabi-Naoufal/EMSI-SHARE/network/members
[stars-shield]: https://img.shields.io/github/stars/Elouahabi-Naoufal/EMSI-SHARE.svg?style=for-the-badge
[stars-url]: https://github.com/Elouahabi-Naoufal/EMSI-SHARE/stargazers
[issues-shield]: https://img.shields.io/github/issues/Elouahabi-Naoufal/EMSI-SHARE.svg?style=for-the-badge
[issues-url]: https://github.com/Elouahabi-Naoufal/EMSI-SHARE/issues
[license-shield]: https://img.shields.io/github/license/Elouahabi-Naoufal/EMSI-SHARE.svg?style=for-the-badge
[license-url]: https://github.com/Elouahabi-Naoufal/EMSI-SHARE
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/elouahabi-naoufal/
[React.js]: https://img.shields.io/badge/React-18.2-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[React-url]: https://react.dev/
[TypeScript]: https://img.shields.io/badge/TypeScript-5.2-3178C6?style=for-the-badge&logo=typescript&logoColor=white
[TypeScript-url]: https://www.typescriptlang.org/
[Vite]: https://img.shields.io/badge/Vite-6.3-646CFF?style=for-the-badge&logo=vite&logoColor=white
[Vite-url]: https://vite.dev/
[TailwindCSS]: https://img.shields.io/badge/Tailwind%20CSS-3.4-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white
[TailwindCSS-url]: https://tailwindcss.com/
[Radix]: https://img.shields.io/badge/Radix%20UI-111827?style=for-the-badge&logo=radixui&logoColor=white
[Radix-url]: https://www.radix-ui.com/
[TanStack-Query]: https://img.shields.io/badge/TanStack%20Query-5.28-FF4155?style=for-the-badge&logo=tanstack&logoColor=white
[TanStack-Query-url]: https://tanstack.com/query
[Django]: https://img.shields.io/badge/Django-5.2-092E20?style=for-the-badge&logo=django&logoColor=white
[Django-url]: https://www.djangoproject.com/
[DRF]: https://img.shields.io/badge/DRF-3.15-ff1709?style=for-the-badge
[DRF-url]: https://www.django-rest-framework.org/
[PostgreSQL]: https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white
[PostgreSQL-url]: https://www.postgresql.org/
[Redis]: https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white
[Redis-url]: https://redis.io/
[Channels]: https://img.shields.io/badge/Django%20Channels-4.0-092E20?style=for-the-badge&logo=django&logoColor=white
[Channels-url]: https://channels.readthedocs.io/
[Daphne]: https://img.shields.io/badge/Daphne-4.0-2A5CAA?style=for-the-badge
[Daphne-url]: https://github.com/django/daphne
[Docker]: https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white
[Docker-url]: https://www.docker.com/
[NGINX]: https://img.shields.io/badge/nginx-009639?style=for-the-badge&logo=nginx&logoColor=white
[NGINX-url]: https://nginx.org/
[i18next]: https://img.shields.io/badge/i18next-26.0-2E86DE?style=for-the-badge&logo=i18next&logoColor=white
[i18next-url]: https://www.i18next.com/