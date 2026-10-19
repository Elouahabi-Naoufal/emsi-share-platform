<a id="readme-top"></a>

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

<br />
<div align="center">
  <h3 align="center">EMSI Share</h3>

  <p align="center">
    A collaborative learning management system with student, teacher and admin roles —
    learning rooms, resources, quizzes, forums, assignments and analytics.
    <br />
    <a href="https://github.com/Elouahabi-Naoufal/emsi-share-platform"><strong>Explore the source »</strong></a>
    <br />
    <br />
    <a href="https://github.com/Elouahabi-Naoufal/emsi-share-platform/issues/new?labels=bug">Report Bug</a>
    &middot;
    <a href="https://github.com/Elouahabi-Naoufal/emsi-share-platform/issues/new?labels=enhancement">Request Feature</a>
  </p>
</div>

<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>

## About The Project

EMSI Share is a learning management platform built as a 3rd-year EMSI project. It pairs a
React 18 single-page application with a Django 5 REST API and PostgreSQL, and adds Redis-backed
Channels for real-time chat and notifications.

The platform is role-aware: students join rooms, consume resources, take quizzes, submit
assignments and track their progress; teachers create rooms and content, grade submissions and
follow attendance; administrators manage users, review pending resources and audit platform
activity.

Implemented areas include:

* Learning rooms with join codes, invitations and participant roles
* Resource uploads with categories, search, preview and an approval workflow
* Quizzes with questions, options, attempts and automatic scoring
* Forums with categories, topics, posts, votes, attachments and subscriptions
* Events, academic calendar, announcements, attendance, timetable, assignments and gradebook
* Direct messaging, notifications, certificates and a parent portal
* JWT authentication, password reset, email verification, TOTP two-factor setup and CSV bulk import
* Audit logs, GDPR-style data export and English/French/Arabic locale files
* Docker packaging with a Daphne entrypoint and nginx configuration

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Built With

* [![React][React.js]][React-url]
* [![TypeScript][TypeScript]][TypeScript-url]
* [![Vite][Vite]][Vite-url]
* [![Tailwind CSS][TailwindCSS]][TailwindCSS-url]
* [![Django][Django]][Django-url]
* [![Django REST Framework][DRF]][DRF-url]
* [![PostgreSQL][PostgreSQL]][PostgreSQL-url]
* [![Redis][Redis]][Redis-url]
* [![Docker][Docker]][Docker-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Getting Started

The intended development loop is two processes: Django on `:8000` and Vite on `:8080`, both
using the same PostgreSQL database. Redis is needed only for the WebSocket features.

### Prerequisites

* Node.js 18+ and npm (a `bun.lockb` is also committed, but the Dockerfile uses `npm ci`)
* Python 3.10+ (the backend image uses `python:3.11-slim`)
* PostgreSQL 14+
* Redis 7 for Channels / WebSockets
* Docker and Docker Compose, optionally, for the containerised setup

### Installation

1. Clone the repository
   ```sh
   git clone https://github.com/Elouahabi-Naoufal/emsi-share-platform.git
   cd emsi-share-platform
   ```

2. Create the database and install the backend
   ```sh
   createdb emsi_share_db
   cd backend
   python3 -m venv env
   source env/bin/activate
   pip install -r requirements.txt
   ```

3. Configure the backend environment
   ```sh
   export SECRET_KEY="replace-with-a-long-random-value"
   export DEBUG=False
   export DB_NAME=emsi_share_db
   export DB_USER=postgres
   export DB_PASSWORD=postgres
   export DB_HOST=localhost
   export DB_PORT=5432
   export REDIS_HOST=127.0.0.1
   export REDIS_PORT=6379
   ```

4. Migrate and start the API
   ```sh
   python manage.py migrate
   python manage.py createsuperuser
   python manage.py runserver 8000
   ```

5. Install and start the frontend, from the repository root
   ```sh
   npm install
   cp .env.development .env.local
   npm run dev
   ```

6. Open the app

   | URL | Service |
   | --- | --- |
   | <http://localhost:8080> | React front end |
   | <http://127.0.0.1:8000> | Django API |
   | <http://127.0.0.1:8000/admin/> | Django admin |

   The committed `.env.development` points at port `8001` while `runserver` defaults to `8000`.
   Set `VITE_API_URL` to the URL you actually run before starting the frontend.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Usage

1. Sign in or register; the API issues a JWT that the front end stores and sends with requests.
2. Teachers create a room and share its join code. Students join and see the room's resources
   and quizzes.
3. Uploaded resources can be browsed and previewed; administrator approval controls visibility.
4. Quizzes can be attempted from the room and are graded automatically.
5. Forums, messaging and notifications are available from the main layout.
6. Teachers and administrators use the dashboards for gradebook, attendance, analytics, pending
   resources, bulk import and audit logs.

The REST API is rooted at `/api/`. JWT endpoints are `/api/token/` and `/api/token/refresh/`;
the browsable Django admin remains available at `/admin/`.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Roadmap

- [x] Rooms, resources, quizzes and forums
- [x] Assignments, gradebook, attendance and timetable
- [x] Messaging, announcements, calendar, certificates and notifications
- [x] JWT auth, password reset, email verification, TOTP setup, parent portal and data export
- [x] English, French and Arabic locales
- [x] Docker image and Daphne entrypoint
- [ ] Replace the committed development `SECRET_KEY`, set `DEBUG=False`, restrict
      `ALLOWED_HOSTS` and configure explicit CORS origins
- [ ] Align `VITE_API_URL` in both env files and replace hard-coded `127.0.0.1:8000` fallbacks
- [ ] Add `Pillow` to the backend requirements for the avatar `ImageField`
- [ ] Consolidate the duplicated PDF viewer components and remove the `*_modified` files
- [ ] Add API throttling and pagination, and add `select_related`/`prefetch_related` to hot queries
- [ ] Register or remove the `analytics` app, which currently has no URLs
- [ ] Add plagiarism detection and live-session integration

See the [open issues](https://github.com/Elouahabi-Naoufal/emsi-share-platform/issues) for a full
list of proposed features and known issues.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire,
and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull
request. You can also simply open an issue with the tag "enhancement".

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

### Top contributors:

<a href="https://github.com/Elouahabi-Naoufal/emsi-share-platform/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=Elouahabi-Naoufal/emsi-share-platform" alt="contrib.rocks image" />
</a>

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## License

This project is not licensed — all rights reserved.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Contact

Naoufal Elouahabi - [LinkedIn](https://www.linkedin.com/in/elouahabi-naoufal/) - elouahabi-naoufal1@gmail.com

Project Link: [https://github.com/Elouahabi-Naoufal/emsi-share-platform](https://github.com/Elouahabi-Naoufal/emsi-share-platform)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Acknowledgments

* [Django REST Framework](https://www.django-rest-framework.org/)
* [Django Channels](https://channels.readthedocs.io/)
* [shadcn/ui](https://ui.shadcn.com/) and [Radix UI](https://www.radix-ui.com/)
* [i18next](https://www.i18next.com/)
* [Best-README-Template](https://github.com/othneildrew/Best-README-Template)
* [Img Shields](https://shields.io)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

[contributors-shield]: https://img.shields.io/github/contributors/Elouahabi-Naoufal/emsi-share-platform.svg?style=for-the-badge
[contributors-url]: https://github.com/Elouahabi-Naoufal/emsi-share-platform/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/Elouahabi-Naoufal/emsi-share-platform.svg?style=for-the-badge
[forks-url]: https://github.com/Elouahabi-Naoufal/emsi-share-platform/network/members
[stars-shield]: https://img.shields.io/github/stars/Elouahabi-Naoufal/emsi-share-platform.svg?style=for-the-badge
[stars-url]: https://github.com/Elouahabi-Naoufal/emsi-share-platform/stargazers
[issues-shield]: https://img.shields.io/github/issues/Elouahabi-Naoufal/emsi-share-platform.svg?style=for-the-badge
[issues-url]: https://github.com/Elouahabi-Naoufal/emsi-share-platform/issues
[license-shield]: https://img.shields.io/badge/license-none-lightgrey?style=for-the-badge
[license-url]: #license
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/elouahabi-naoufal/
[React.js]: https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB
[React-url]: https://react.dev/
[TypeScript]: https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white
[TypeScript-url]: https://www.typescriptlang.org/
[Vite]: https://img.shields.io/badge/Vite-646CFF?style=for-the-badge&logo=vite&logoColor=white
[Vite-url]: https://vite.dev/
[TailwindCSS]: https://img.shields.io/badge/Tailwind%20CSS-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white
[TailwindCSS-url]: https://tailwindcss.com/
[Django]: https://img.shields.io/badge/Django-092E20?style=for-the-badge&logo=django&logoColor=white
[Django-url]: https://www.djangoproject.com/
[DRF]: https://img.shields.io/badge/DRF-A30000?style=for-the-badge&logo=django&logoColor=white
[DRF-url]: https://www.django-rest-framework.org/
[PostgreSQL]: https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white
[PostgreSQL-url]: https://www.postgresql.org/
[Redis]: https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white
[Redis-url]: https://redis.io/
[Docker]: https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white
[Docker-url]: https://www.docker.com/
