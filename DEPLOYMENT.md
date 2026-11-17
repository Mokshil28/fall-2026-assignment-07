# Assignment 07 deployment plan

The deliverable is the existing article app running publicly on Render. The browser calls `/api`, Render forwards those requests to Express, and Express stores users and articles in Postgres. Database migrations run automatically before the backend starts.

## What you need

- Your repository: `https://github.com/Mokshil28/fall-2026-assignment-07`.
- Docker Desktop running and Node.js 24 for local verification.
- A GitHub account with Actions enabled for this fork (open the Actions tab and enable workflows if prompted).
- A Render account connected to this repository.
- For a private GHCR image, a GitHub classic PAT with `read:packages` for Render. Local manual publishing also needs `write:packages`. Store the token in Docker/Render credentials, never in source control.
- A generated JWT secret, the Render database internal connection URL, and the final backend/frontend URLs.

## Implemented in this repository

- [x] Multi-stage, non-root backend Docker image using Node 24.
- [x] Production compilation emits `dist/index.js` and compiled database migrations.
- [x] Container build excludes tokens, environment files, and local dependencies.
- [x] Backend honors Render's `PORT` and requires production database/JWT settings.
- [x] Local Compose waits for Postgres readiness.
- [x] GitHub Actions builds a Linux amd64 image and publishes it to GHCR on server changes pushed to `main`, or via **Run workflow**.

## 1. Verify locally

From the repository root:

```sh
npm ci --prefix server
npm run build --prefix server
npm ci --prefix client
npm run build --prefix client
docker build --platform linux/amd64 -t assignment-07:latest server
docker compose -f server/docker-compose.dev.yml up -d --wait
curl --fail http://localhost:8080/health
```

Expect `{"status":"ok"}`. Then run the frontend with `npm run dev --prefix client` and test signup, login, article creation, search, and article details. Stop the containers afterward with `docker compose -f server/docker-compose.dev.yml down`.

The local database uses host port `55407` to avoid conflicts with other assignments; containers connect to it internally on port `5432`.

For the existing integration tests, use a disposable local database with no other app processes writing to it. Start the database using `docker compose -f server/docker-compose.yml up -d --wait`, run `npm run migrate --prefix server`, then `npm test --prefix server`. The tests delete articles and users from the configured database; never point them at the deployed database.

## 2. Publish the image

- [ ] Commit and push the implementation to your repository's `main` branch.
- [ ] Open **Actions → Publish backend image**, enable Actions for the fork if needed, and run the workflow if it did not trigger automatically.
- [ ] Wait for success and check your GitHub profile's Packages section for `assignment-07`.

The workflow uses GitHub's built-in token, so no PAT is needed for Actions publishing. Image address: `ghcr.io/mokshil28/assignment-07:latest`.

Manual alternative, from the repository root (Docker prompts for your PAT as the password):

```sh
docker login ghcr.io -u Mokshil28
docker build --platform linux/amd64 -t ghcr.io/mokshil28/assignment-07:latest server
docker push ghcr.io/mokshil28/assignment-07:latest
```

The amd64 target avoids publishing an Apple Silicon-only image that Render cannot run.

## 3. Create Render Postgres

- [ ] Create a Render project named `assignment-07`.
- [ ] Add a **Postgres** database on the **Free** plan.
- [ ] Choose a region and use the same region for the backend.
- [ ] Wait until ready, then copy **Connect → Internal Database URL** into Render's backend environment settings below.

Free databases expire after 30 days; schedule your submission/demo accordingly. See [Render's free-plan documentation](https://render.com/docs/free).

## 4. Create the backend web service

- [ ] Add a **Web Service → Existing Image** using `ghcr.io/mokshil28/assignment-07:latest`.
- [ ] If private, supply your GitHub username and a classic PAT with `read:packages` as the registry credential.
- [ ] Select **Free**, use the database's region, and set health check path `/health`.
- [ ] Set these environment variables:

| Variable | Value |
| --- | --- |
| `NODE_ENV` | `production` (also set in the image) |
| `DB_CONNECTION_STRING` | Render Postgres internal connection URL |
| `JWT_SECRET` | A random secret generated below |
| `PORT` | `8080`, or use Render's assigned port |

Generate a secret locally with `openssl rand -hex 32`; paste it directly into Render.

- [ ] Deploy and confirm logs report successful migrations and server startup.
- [ ] Open `https://YOUR_BACKEND.onrender.com/health` and confirm `{"status":"ok"}`.

After publishing a new `latest` image, manually deploy the latest image in Render unless you configure image redeploy automation there.

## 5. Create the frontend static site

- [ ] Add a **Static Site**, connect this GitHub repository, and select `main`.
- [ ] Set **Root Directory** to `client`.
- [ ] Set **Build Command** to `npm ci && npm run build`.
- [ ] Set **Publish Directory** to `dist`.
- [ ] Use Node 24 (set `NODE_VERSION=24` if necessary).
- [ ] Add these **Redirects/Rewrites**, with the API rule first:

| Source | Destination | Action |
| --- | --- | --- |
| `/api/*` | `https://YOUR_BACKEND.onrender.com/*` | Rewrite |
| `/*` | `/index.html` | Rewrite |

Replace the backend hostname with your actual backend URL. The second rule supports refreshing React routes; the first forwards API requests without the `/api` prefix. `VITE_API_HOST` only configures the local Vite development proxy; it does not replace the Render rewrite. See [Render rewrite documentation](https://render.com/docs/redirects-rewrites).

## 6. Final verification and submission

- [ ] Open the public frontend URL.
- [ ] Sign up with a new username and log in.
- [ ] Create an article, find it in the list/search, and open its details.
- [ ] Refresh an article details URL directly and confirm it still loads.
- [ ] Log out and log back in to confirm the article persists.
- [ ] Confirm `/api/health` on the frontend returns backend JSON.
- [ ] Record the frontend URL, backend health URL, GHCR image URL, and repository URL for submission.
- [ ] Capture any screenshots required by your course portal. The supplied README does not specify the submission format; check the portal for it.

## Local verification results (November 17, 2026)

- Backend and frontend production builds passed.
- Both existing API integration tests passed against a separate disposable Postgres database.
- Compiled backend started in production mode, ran migrations, honored a custom `PORT`, and returned `{"status":"ok"}` from `/health`.
- Docker Compose configuration validated successfully.
- The Linux amd64 production Docker image built successfully. Its build stage omits optional development binaries and install scripts because compilation only needs TypeScript.
- The production container passed health, signup, login, article creation, and article retrieval smoke checks.
- The existing backend production dependency audit reports three advisories (moderate, high, and critical) in `qs`, `body-parser`, and `proxy-addr`. Dependency updates remain a separate follow-up.
- GitHub authentication works outside the filesystem sandbox. The repository is confirmed to be a fork of `ITSC-3155-Krevat/fall-2026-assignment-07`.
- Public Render deployment and browser verification remain pending account setup.

## GitHub contribution credit

Commits pushed only to a fork do not count toward the profile contribution graph. GitHub says those commits can count if merged into the parent repository. For standalone repositories, eligible commits normally need to be on the default or `gh-pages` branch and authored with an email associated with your GitHub account; updates can take 24 hours. Course grading is separate from the contribution graph: follow the course's required repository/submission instructions.

Source: [GitHub: troubleshooting missing contributions](https://docs.github.com/en/account-and-profile/how-tos/contribution-settings/troubleshooting-missing-contributions).
