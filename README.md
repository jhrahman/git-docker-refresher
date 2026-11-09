# Git + Docker + GitLab refresher

Quick-reference for the daily loop: push code to GitLab, clone it fresh, pick the right branch, run the whole stack with Compose. The practice app is a todo list with three services. Longer walkthroughs and the labs live in [KB.md](KB.md).

## The app

| Service    | Source               | Notes                                  |
|------------|----------------------|----------------------------------------|
| `db`       | `postgres:16-alpine` | Named volume `dbdata`, creds via env   |
| `backend`  | `./backend`          | Node/Express, reads `DATABASE_URL`     |
| `frontend` | `./frontend`         | nginx serving a static page, `8080:80` |

Only `frontend` publishes a port. `backend` and `db` sit on the Compose network and find each other by service name (`db:5432`).

`backend` and `frontend` are separate private GitLab repos (`todo-backend`, `todo-frontend`). `docker-compose.yml` lives in the parent folder and isn't tracked in either.

## Fresh machine to running app

```sh
mkdir myapp && cd myapp
git clone git@gitlab.com:<user>/todo-backend.git backend
git clone git@gitlab.com:<user>/todo-frontend.git frontend
cp <path>/docker-compose.yml .
docker compose up --build
```

App: http://localhost:8080

## Git

Seed a new GitLab project (empty, private, no README) from a local folder:

```sh
git init -b main
git add . && git commit -m "Initial commit"
git remote add origin git@gitlab.com:<user>/todo-backend.git
git push -u origin main
```

Branches:

```sh
git clone -b beta <url> backend          # clone straight onto a known branch
git clone -b beta --single-branch <url>  # same, but skip every other branch
git fetch --all --prune                  # refresh remote branches, drop deleted ones
git branch -r                            # list remote branches
git checkout beta                        # switch (bare name, no origin/)
git branch --show-current                # where am I
```

Newest branch when nobody tells you the name:

```sh
git for-each-ref --sort=-committerdate refs/remotes/origin \
  --format='%(committerdate:short)  %(refname:strip=3)' | grep -v ' HEAD$'
```

Repo inside a repo (parent shows `backend` as an empty pointer):

```sh
rm -rf backend/.git frontend/.git        # make them plain folders (history is lost)
git rm --cached backend frontend         # drop the stale pointers from the index
```

Or keep them separate and add `backend/` and `frontend/` to the parent `.gitignore`.

## Docker

```sh
docker build -t todo-backend .
docker run -d -p 8081:80 --name web nginx:alpine
docker ps                                # add -a for stopped ones
docker logs -f <name>
docker stop <name> && docker rm <name>
docker network create todo-net           # containers on it resolve each other by name
```

## Compose

```sh
docker compose up --build -d      # rebuild and run in the background
docker compose ps
docker compose logs -f backend    # follow one service
docker compose exec db psql -U app appdb
docker compose down               # stop, keep the data
docker compose down -v            # stop and wipe dbdata
```

## GitLab vs GitHub

| GitHub                        | GitLab                       |
|-------------------------------|------------------------------|
| Repository                    | Project                      |
| Organization                  | Group                        |
| Pull request                  | Merge request                |
| Actions (`.github/workflows`) | Pipelines (`.gitlab-ci.yml`) |

## Gotchas

- `git clone` grabs every branch but checks out only the default one. Use `git checkout <name>`, not `origin/<name>`, or you end up in detached HEAD.
- `depends_on` only controls start order. It doesn't wait for Postgres to be ready, so the backend retries or you add a `healthcheck` plus `condition: service_healthy`.
- The `POSTGRES_*` vars only apply when the volume is first initialized. Changing them later does nothing until `down -v`.
- No bind mounts, so any code change (or switching branches) needs `--build`.
- Inside a container, `localhost` is that container. Use the service name.
- `rm -rf` on a `.git` folder can't be undone. Push first.
- The credentials in the compose file are for practice. Move them to an untracked `.env` for anything real.
