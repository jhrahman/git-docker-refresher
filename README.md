# Docker refresher: 3-tier todo app

My notes for the multi-container drill. Three services wired together with Compose, code split across two GitLab repos.

## Layout

| Service    | Source               | Notes                                       |
|------------|----------------------|---------------------------------------------|
| `db`       | `postgres:16-alpine` | Named volume `dbdata`, creds via env        |
| `backend`  | `./backend`          | Node/Express, reads `DATABASE_URL`          |
| `frontend` | `./frontend`         | nginx serving a static page, `8080:80`      |

Only `frontend` publishes a port. `backend` and `db` live on the Compose network and talk to each other by service name (`db:5432`).

## Repo split

`backend` and `frontend` are separate private repos on GitLab (`todo-backend`, `todo-frontend`). The compose file sits in the parent folder and isn't tracked in either one.

To seed a repo, run this in `backend/`, then again in `frontend/`:

```sh
git init -b main
git add . && git commit -m "Initial commit"
git remote add origin git@gitlab.com:<user>/todo-backend.git
git push -u origin main
```

## Starting from scratch

```sh
mkdir myapp && cd myapp
git clone git@gitlab.com:<user>/todo-backend.git backend
git clone git@gitlab.com:<user>/todo-frontend.git frontend
cp <path>/docker-compose.yml .
docker compose up --build
```

Then open http://localhost:8080.

## Commands I keep reaching for

```sh
docker compose up --build -d      # rebuild and run in the background
docker compose logs -f backend    # follow one service
docker compose exec db psql -U app appdb
docker compose down               # stop, keep the data
docker compose down -v            # stop and wipe dbdata
```

## Gotchas

- `depends_on` only controls start order. It doesn't wait for Postgres to be ready, so the backend either retries or you add a `healthcheck` plus `condition: service_healthy`.
- The `POSTGRES_*` vars only apply the first time the volume is initialized. Changing them later does nothing until you run `down -v`.
- There are no bind mounts, so any code change needs `--build`.
- The credentials in the compose file are fine for practice. Move them to an `.env` file for anything real.
