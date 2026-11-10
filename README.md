# RaceDay

![CI](https://github.com/<your-username>/<your-repo>/actions/workflows/ci.yml/badge.svg)

RaceDay is a web application for running, trail, walking and cycling events. **Organisers** create events and categories and
capture results, **participants** find events, enter them and see their finish times, and **admins** manage users and roles.

| Part | Technology |
|---|---|
| Web app (`Race/`) | ASP.NET Core 8 MVC, ASP.NET Core Identity, Entity Framework Core, SQL Server |
| Web API (`RaceDay.Api/`) | ASP.NET Core 8 Web API, Azure Blob Storage (event banner images) |
| Tests (`Race.Tests/`) | xUnit + Moq |
| Delivery | Docker + docker-compose, GitHub Actions CI |

```
 Browser ──► RaceDay MVC (Race/) ──► SQL Server        (users, roles, events, enrolments, results)
                  │
                  └── POST /api/banners (X-Api-Key) ──► RaceDay API (RaceDay.Api/) ──► Azure Blob Storage
                                                                                          │
   Event.BannerImageUrl (saved in SQL) ◄── blob URL returned by the API ◄─────────────────┘
```

## Run with Docker (one command)

Requirements: Docker Desktop and an Azure Storage account (for banner images).

```powershell
copy .env.example .env      # then edit .env: SA_PASSWORD, API_KEY, AZURE_STORAGE_CONNECTION_STRING
docker compose up --build
```

Open <http://localhost:8080>. The API health check is at <http://localhost:5100/health>.
This starts **three containers**: the MVC app, the Web API and SQL Server. The database is migrated and seeded automatically
on first start (give SQL Server ~30 seconds).

To stop: `docker compose down` (add `-v` to also delete the database volume).

### Demo accounts (seeded automatically)

| Role | Email | Password |
|---|---|---|
| Admin | `admin@admin.com` | `Admin12@` |
| Organiser | `organiser@raceday.com` | `Raceday1@` |
| Participant | `participant@raceday.com` | `Raceday1@` |

You can also use **Register** to create a new Participant or Organiser account. Set `Seed__DemoData` to `false` to skip the demo users and events.

## Run locally (Visual Studio / .NET 8 SDK)

```powershell
# 1. API: store the Azure connection string outside source control
cd RaceDay.Api
dotnet user-secrets set "AzureStorage:ConnectionString" "<your storage account connection string>"
dotnet run                     # listens on http://localhost:5100

# 2. MVC app (second terminal) - applies migrations and seeds on start-up
cd ../Race
dotnet run
```

In Visual Studio choose **Configure Startup Projects > Multiple startup projects** and start both `Race` and `RaceDay.Api`.
Both projects use the shared key `RaceDayApi:ApiKey` (default `dev-only-change-me` for local use; override it for anything else).
The default SQL connection string targets `localhost\SQLEXPRESS01` - change `ConnectionStrings:DefaultConnection` in `Race/appsettings.json` if yours differs.

## Azure Blob Storage set-up

1. Create a Storage account (Standard, LRS).
2. Storage account > Settings > Configuration > **Allow Blob anonymous access = Enabled**.
3. Copy the connection string from Security + networking > Access keys.
4. The API creates the `event-banners` container (public read for blobs) on the first upload.

Images are uploaded **MVC -> API -> Azure**; only the returned URL is stored in the `Events.BannerImageUrl` column and displayed in the MVC pages.
The storage key lives only in the API's configuration (user-secrets locally, `.env` for Docker) and is never committed.

## Features

**Authentication and role-based navigation**
- Register (choose Participant or Organiser) and Login with ASP.NET Core Identity.
- On login the **session stores the user id, name and role**; **logout clears the session**.
- Each role gets its own menu and landing dashboard. Unauthenticated visitors are redirected to the login page; wrong-role access goes to *Access denied*.

**Organiser interface** (`Dashboard`, `Events`, `Categories`, `Enrolments`, `Results`)
- Dashboard: events run, upcoming events, total enrolments, results captured, events still awaiting results.
- Events: create / edit / delete with a banner image (stored in Azure through the API). Organisers only see and change their own events.
- Categories: age and distance categories per event.
- Enrolments: see who entered each event, grouped by category.
- Results: enter finish time and position per participant (validated: both or neither, no duplicate positions, only after the event).

**Participant interface**
- Home dashboard: next event countdown, upcoming entries, recent results, events open for entry.
- Find events: browse upcoming events as cards with banners, search and filter by type.
- Event details with an **Enter event** button (choose a category).
- My events (withdraw before the event) and **My results** (finish time and position).
- Profile page: name, picture and phone number.

**Admin**: dashboard across all events and a *Manage users* page to change roles.

## Tests

```powershell
dotnet test RaceDay.sln
```

Covers controller authorisation rules (who may call which action), anti-forgery protection, view-model building, role management,
session helpers, the MVC->API client (fake HTTP handler) and the API (file validation, signature check, API-key filter).

## Smoke test

After `docker compose up --build`:

```powershell
powershell -ExecutionPolicy Bypass -File scripts/smoke-test.ps1 -ApiKey "<API_KEY from .env>"
```

The script checks both services, role-based access, the session/logout behaviour and an Azure upload through the API.
The full manual checklist (create an event with a banner, enrol, capture results) is in [docs/SMOKE-TEST.md](docs/SMOKE-TEST.md).

## CI/CD

`.github/workflows/ci.yml` runs on every push and pull request: validates the repository structure, restores, builds, runs the tests,
then builds both Docker images and validates `docker-compose.yml`.

![CI green build](docs/ci-green-build.png)
<!-- Add your screenshot of the green workflow run as docs/ci-green-build.png -->

## Demo video

<!-- Paste your video link here (YouTube unlisted / OneDrive / Google Drive) -->
Video link: **TODO - add link**

## Repository layout

```
RaceDay.sln
docker-compose.yml  .env.example  .gitignore
.github/workflows/ci.yml
Race/                MVC app (Controllers, Models, Views, Areas/Identity, Data, Services, Dockerfile)
RaceDay.Api/         Web API for Azure Blob Storage (Controllers, Services, Security, Dockerfile)
Race.Tests/          xUnit tests
docs/                rubric checklist, smoke test checklist, video script
scripts/             smoke-test.ps1 and make-commits.ps1 (git history helper)
```

## Security notes

- Never commit `.env`, storage keys or `secrets.json` (they are in `.gitignore`). If a key was ever shown on screen or in a recording, rotate it in
  Azure Portal > Storage account > Access keys.
- The API only accepts requests that carry the shared `X-Api-Key`, validates extension, content type, size and the real file signature, and only deletes blobs under `banners/`.
- Change the seeded admin password and `API_KEY` before using this anywhere real.

## Troubleshooting

| Problem | Fix |
|---|---|
| "The image service is not reachable" | The API container is not running, or `RaceDayApi__BaseUrl` / `API_KEY` do not match. |
| Upload says it could not be uploaded | Check `AZURE_STORAGE_CONNECTION_STRING` and that blob anonymous access is enabled. |
| Web container restarts on first start | SQL Server is still starting - the app retries for about 50 seconds. |
| `LocalDB is not supported on this platform` | LocalDB does not work in Linux containers; use `docker compose` (SQL Server container). |
