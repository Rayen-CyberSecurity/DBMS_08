# DBMS_09 – From Container to Docker Compose

**Module:** Databases · THGA Bochum  
**Lecturer:** Stephan Bökelmann · <sboekelmann@ep1.rub.de>  
**Repository:** <https://github.com/MaxClerkwell/DBMS_09>  
**Prerequisites:** DBMS_01 – DBMS_07, Lecture 09  
**Duration:** 120 minutes

---

## Learning Objectives

After completing this exercise you will be able to:

- Explain the difference between a **container** and a **virtual machine**
- Start, inspect, and remove Docker **containers** and **images**
- Write a minimal **Dockerfile** and build a custom image from it
- Demonstrate the **ephemeral** nature of container storage
- Attach a **named volume** to a PostgreSQL container and verify data persists
  across container restarts
- Explain why running two services in one container is an anti-pattern
- Connect two containers via a **custom bridge network** using container names
  as hostnames
- Describe all key sections of a **docker-compose.yml** file
- Start a multi-service application with `docker compose up` and take it down
  cleanly
- Use a **`.env` file** to keep credentials out of `docker-compose.yml`
- Explain what a **Multi-Stage Build** is and why it reduces image size
- Apply the **Principle of Least Privilege** by running containers as a
  non-root user
- Use **init scripts** to initialise a PostgreSQL database on first start

**After completing this exercise you should be able to answer the following questions independently:**

- What is the difference between a Docker image and a running container?
- Why does deleting a container without a volume lose all data written to it?
- Why can `host="localhost"` inside one container not reach a second container?
- What does `docker compose down -v` do that `docker compose down` does not?
- Why should credentials never appear directly in `docker-compose.yml`?

---

## Prerequisites Check

You need Docker and Docker Compose installed.

```bash
docker --version
docker compose version
```

> Both commands should succeed and show version numbers.

> **Screenshot 1:** Take a screenshot showing both version outputs.
>
> <img width="451" height="99" alt="image" src="https://github.com/user-attachments/assets/921d8d25-06a2-4d83-b814-4e99d17c55d7" />


---

## 1 – Hello Container

### Step 1 – Run the hello-world Image

```bash
docker run hello-world
```

> Docker pulls the image from Docker Hub (first run only), starts a container,
> prints a message, and exits.

List all containers, including stopped ones:

```bash
docker ps -a
docker images
```

> **Screenshot 2:** Take a screenshot showing `docker ps -a` and
> `docker images` output.
>
> <img width="1087" height="165" alt="image" src="https://github.com/user-attachments/assets/f1c96663-6d40-40a9-87a6-bdf8a73ed4ae" />


### Step 2 – Run an nginx Webserver

```bash
docker run -d -p 8080:80 --name webserver nginx
curl http://localhost:8080
docker logs webserver
docker exec -it webserver bash
```

Inside the container shell, inspect the running process and exit:

```bash
ps aux
exit
```

Stop and remove the container:

```bash
docker stop webserver && docker rm webserver
docker system df
```

### Questions for Section 1

**Question 1.1:** The flag `-d` starts the container in detached mode.
What happens without `-d`, and why is detached mode useful for a web server?

> Without `-d`, the container runs in the foreground and my terminal stays attached to it. Detached mode is useful for a web server because the server can keep running in the background while I continue using the terminal.

**Question 1.2:** `-p 8080:80` maps host port 8080 to container port 80.
Which port is the application actually listening on inside the container?
What would `-p 9000:80` change?

> Inside the container, nginx is listening on port `80`. The option `-p 8080:80` means I access it from the host on port `8080`; if I use `-p 9000:80`, the container still listens on `80`, but I open it on the host with `http://localhost:9000`.

---

## 2 – Writing a Dockerfile

### Step 1 – Create the Project Directory

```bash
mkdir ~/dbms09_dockerfile
cd ~/dbms09_dockerfile
git init
git remote add origin git@github.com:<your-username>/dbms09_dockerfile.git
```

### Step 2 – Write the Dockerfile

```bash
vim Dockerfile
```

```dockerfile
FROM debian:13
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
CMD ["bash"]
```

### Step 3 – Build and Run

```bash
docker build -t mein-debian .
docker run -it mein-debian
```

Inside the container:

```bash
curl --version
whoami
exit
```

> **Screenshot 3:** Take a screenshot showing the `docker build` output and
> the commands run inside the container.
>
> <img width="1144" height="643" alt="image" src="https://github.com/user-attachments/assets/b927886b-99cf-4043-942d-228a3ee8aa70" />


### Step 4 – Commit

```bash
git add Dockerfile
git commit -m "feat: minimal Debian image with curl"
git push -u origin main
```

### Questions for Section 2

**Question 2.1:** Why does the `RUN` instruction combine `apt-get update`,
`apt-get install`, and `rm -rf /var/lib/apt/lists/*` in a single line?
What would happen to the image size if these were three separate `RUN` lines?

> The commands are combined in one `RUN` line so Docker creates only one layer and the apt cache is removed in the same layer. If they were three separate `RUN` lines, the old package lists could remain in previous layers and make the image larger.

**Question 2.2:** `EXPOSE 80` in a Dockerfile does **not** actually open port
80. What does it do, and what is required at `docker run` time to actually
forward a port?

> `EXPOSE 80` is mainly documentation/metadata showing that the container application expects to use port `80`. To actually make the port reachable from the host, I still need to publish it at runtime with something like `docker run -p 8080:80 ...`.

---

## 3 – The Persistence Problem

### Step 1 – Start a PostgreSQL Container Without a Volume

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -d postgres:16
```

Connect and create a table:

```bash
docker exec -it pg psql -U postgres
```

Inside `psql`:

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Hallo Docker');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate the Container

```bash
docker stop pg && docker rm pg
docker run --name pg -e POSTGRES_PASSWORD=geheim -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> Expected output: `ERROR: relation "test" does not exist`

> **Screenshot 4:** Take a screenshot showing the error message.
>
> <img width="931" height="119" alt="image" src="https://github.com/user-attachments/assets/869c84f1-8fdd-4348-8052-f95d6370ef9a" />


### Questions for Section 3

**Question 3.1:** You stopped and removed the container but the image
`postgres:16` still exists on your machine. Why does recreating a container
from the same image not restore the data?

> A Docker image is only the starting template. The data I created was stored in the writable layer of the container, and when I removed the container, that layer was deleted. Creating a new container from the same image starts fresh.

**Question 3.2:** `docker stop` sends SIGTERM and waits for the process to
exit cleanly. `docker kill` sends SIGKILL immediately. Why is `docker stop`
preferred for a database container?

> `docker stop` gives the database time to shut down cleanly, finish writes, and close files correctly. `docker kill` stops it immediately, which can interrupt database operations and may risk inconsistent data.

---

## 4 – Named Volumes

### Step 1 – Create a Volume and Attach It

```bash
docker volume create pg_data
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Insert data:

```bash
docker exec -it pg psql -U postgres
```

```sql
CREATE TABLE test (id SERIAL PRIMARY KEY, wert TEXT);
INSERT INTO test (wert) VALUES ('Daten überleben');
SELECT * FROM test;
\q
```

### Step 2 – Destroy and Recreate With the Same Volume

```bash
docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
docker exec -it pg psql -U postgres -c "SELECT * FROM test;"
```

> The data should still be there.

```bash
docker volume ls
docker volume inspect pg_data
```

> **Screenshot 5:** Take a screenshot showing the `SELECT` result after
> container recreation, and the `docker volume inspect` output.
>
> <img width="934" height="489" alt="image" src="https://github.com/user-attachments/assets/ce6379eb-5786-4276-b199-77e1358ea165" />


### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker volume rm pg_data
```

### Questions for Section 4

**Question 4.1:** `docker volume inspect pg_data` shows a `Mountpoint` on
the host filesystem. Why is it still recommended to use named volumes instead
of bind-mounting that path directly with `-v /var/lib/docker/volumes/...`?

> Named volumes are managed by Docker, so Docker handles the real storage path, permissions, and portability. The internal mountpoint path under `/var/lib/docker/...` is an implementation detail and should not be used directly because it can change or cause permission problems.

**Question 4.2:** You want to back up the database. Which `docker` command
lets you copy files out of a running container, and how would you copy the
volume contents to a `.tar.gz` archive on the host?

> The command `docker cp` can copy files out of a running container. To archive a volume, I can start a temporary container with the volume mounted and run a command like `docker run --rm -v pg_data:/data -v "$PWD":/backup alpine tar czf /backup/pg_data.tar.gz -C /data .`.

---

## 5 – Two Containers and the Network Problem

### Step 1 – Reproduce the Connectivity Failure

Start PostgreSQL:

```bash
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    -d postgres:16
```

Start a second container and try to reach the first via `localhost`:

```bash
docker run --rm -it postgres:16 \
    psql -h localhost -U postgres -c "SELECT 1;"
```

> This should fail with a connection refused error.

> **Screenshot 6:** Take a screenshot showing the connection error.
>
> <img width="841" height="133" alt="image" src="https://github.com/user-attachments/assets/e48c3f8e-2901-4ce9-a8fa-1cce514f637e" />


### Step 2 – Fix It With a Custom Bridge Network

```bash
docker network create mein-netz

docker stop pg && docker rm pg
docker run --name pg \
    -e POSTGRES_PASSWORD=geheim \
    -v pg_data:/var/lib/postgresql/data \
    --network mein-netz \
    -d postgres:16

docker run --rm -it \
    --network mein-netz \
    postgres:16 \
    psql -h pg -U postgres -c "SELECT 1;"
```

> Notice that `-h pg` uses the **container name** as the hostname — Docker's
> internal DNS resolves it automatically.

```bash
docker network inspect mein-netz
```

### Step 3 – Clean Up

```bash
docker stop pg && docker rm pg
docker network rm mein-netz
docker volume rm pg_data
```

### Questions for Section 5

**Question 5.1:** Without a custom bridge network, containers are placed on
the default bridge. Why can containers on the default bridge **not** resolve
each other by name, while containers on a user-defined bridge can?

> On the default bridge network, Docker does not automatically provide name resolution between containers. On a user-defined bridge network, Docker provides internal DNS, so containers can reach each other using their container names.

**Question 5.2:** You could find the IP address of the `pg` container with
`docker inspect` and hard-code it. Why is using the container name as a
hostname strongly preferable?

> A container IP address can change when the container is recreated. Using the container name is more stable, easier to read, and works automatically with Docker's internal DNS on a user-defined network.

---

## 6 – Docker Compose

Managing multiple `docker run` commands by hand is error-prone. Docker Compose
describes an entire multi-service application in a single YAML file.

### Step 1 – Create the Project

```bash
mkdir ~/dbms09_compose
cd ~/dbms09_compose
git init
git remote add origin git@github.com:<your-username>/dbms09_compose.git
```

### Step 2 – Write docker-compose.yml

```bash
vim docker-compose.yml
```

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend

  api:
    image: python:3.12-slim
    working_dir: /app
    volumes:
      - ./api:/app
    command: >
      bash -c "pip install fastapi uvicorn psycopg2-binary --quiet
               && uvicorn main:app --host 0.0.0.0 --port 8000"
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    networks:
      - backend

volumes:
  pg_data:

networks:
  backend:
```

### Step 3 – Create the API Code

```bash
mkdir api
vim api/main.py
```

```python
import psycopg2
import psycopg2.extras
from fastapi import FastAPI

app = FastAPI(title="Studenten-API")

DB_CONFIG = {
    "dbname": "vorlesung",
    "user": "vorlesung",
    "password": "geheim",
    "host": "postgres",   # container name as hostname
    "port": 5432,
}

@app.get("/")
def root():
    return {"status": "API läuft"}

@app.get("/studenten")
def alle_studenten():
    conn = psycopg2.connect(**DB_CONFIG)
    cur = conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor)
    cur.execute("SELECT id, matrikel, nachname, vorname FROM student ORDER BY nachname")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return rows
```

### Step 4 – Start and Test

```bash
docker compose up -d
docker compose ps
docker compose logs api
```

Wait for the API to start, then test:

```bash
curl http://localhost:8000/
curl http://localhost:8000/studenten
```

> The `/studenten` endpoint will return an empty list for now — that is
> expected. You will add the schema in Section 7.

> **Screenshot 7:** Take a screenshot showing `docker compose ps` and the
> `curl /` response.
>
> <img width="1247" height="368" alt="image" src="https://github.com/user-attachments/assets/82cc4866-447e-4563-989e-8553b17eacf5" />


### Step 5 – Observe Compose Networking

```bash
docker network ls
docker network inspect dbms09_compose_backend
```

> Compose automatically prefixes network names with the project directory name.

### Step 6 – Take It Down

```bash
docker compose down
docker compose down -v   # also removes the named volume
```

> **Question:** What is the difference between `down` and `down -v`?
> When would you use each?
>
> `docker compose down` stops and removes the containers and networks, but it keeps the named volumes. `docker compose down -v` also removes the volumes, so database data is deleted. I would use `down` for a normal stop/restart and `down -v` when I want to reset everything and start with an empty database.

### Step 7 – Commit

```bash
git add docker-compose.yml api/main.py
git commit -m "feat: initial docker-compose setup with postgres and api"
git push -u origin main
```

### Questions for Section 6

**Question 6.1:** `depends_on: postgres` ensures the `postgres` service
starts before `api`. Does it guarantee that PostgreSQL is **ready to accept
connections** when the API starts? What is the correct way to handle this?

> `depends_on` only controls start order. It does not guarantee that PostgreSQL is already ready to accept connections. The correct solution is to use a healthcheck and/or retry logic in the API so it waits until the database is actually reachable.

**Question 6.2:** The `api` service uses `volumes: - ./api:/app` (a bind
mount). What is the advantage of this during development compared to
`COPY`-ing the code into an image at build time?

> The bind mount makes the local `./api` folder available directly inside the container. This is useful during development because code changes on the host are immediately visible in the container without rebuilding the image every time.

---

## 7 – Init Script for PostgreSQL

The official `postgres` image runs all scripts placed in
`/docker-entrypoint-initdb.d/` on first start. This lets you initialise the
schema automatically.

### Step 1 – Write init.sql

```bash
vim init.sql
```

```sql
CREATE TABLE IF NOT EXISTS student (
    id        INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    matrikel  CHAR(8)      NOT NULL UNIQUE,
    nachname  VARCHAR(100) NOT NULL,
    vorname   VARCHAR(100) NOT NULL,
    email     VARCHAR(200)
);

INSERT INTO student (matrikel, nachname, vorname, email) VALUES
    ('12345678', 'Meier',   'Anna',  'a.meier@stud.thga.de'),
    ('23456789', 'Schmidt', 'Ben',   'b.schmidt@stud.thga.de'),
    ('34567890', 'Yilmaz',  'Ceren', 'c.yilmaz@stud.thga.de'),
    ('45678901', 'Nguyen',  'David', 'd.nguyen@stud.thga.de');
```

### Step 2 – Mount the Script in docker-compose.yml

Add a bind mount to the `postgres` service so the script lands in the init
directory:

```yaml
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: vorlesung
      POSTGRES_PASSWORD: geheim
      POSTGRES_DB: vorlesung
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
```

### Step 3 – Reinitialise and Test

The init script only runs when the data directory is empty. Remove the
existing volume first:

```bash
docker compose down -v
docker compose up -d
```

Wait a moment, then query the data:

```bash
curl http://localhost:8000/studenten
```

> You should now see all four students in the JSON response.

> **Screenshot 8:** Take a screenshot showing the `curl /studenten` response
> with all four rows.
>
> <img width="1241" height="64" alt="image" src="https://github.com/user-attachments/assets/de7eaa47-44e0-4fda-b3a7-29098e7d0aed" />


### Step 4 – Commit

```bash
git add init.sql docker-compose.yml
git commit -m "feat: add init.sql for automatic schema and seed data"
git push
```

### Questions for Section 7

**Question 7.1:** You run `docker compose down` (without `-v`), change
`init.sql`, and run `docker compose up -d` again. The schema change does
**not** appear in the database. Why not, and how do you force re-initialisation?

> The init script only runs on the first start when the PostgreSQL data directory is empty. If I only run `docker compose down`, the volume remains, so the old database stays. To force re-initialisation, I use `docker compose down -v` to remove the volume and then start again.

**Question 7.2:** `GENERATED ALWAYS AS IDENTITY` is used instead of
`SERIAL`. What is the practical difference? Which one is the modern
SQL-standard approach?

> `SERIAL` is a PostgreSQL shortcut that creates a sequence behind the scenes. `GENERATED ALWAYS AS IDENTITY` is the modern SQL-standard way to create auto-generated IDs, so it is preferred for new projects.

---

## 8 – Secrets via .env

Passwords must not appear in `docker-compose.yml` because that file is
committed to version control.

### Step 1 – Create .env

```bash
vim .env
```

```
POSTGRES_USER=vorlesung
POSTGRES_PASSWORD=geheim
POSTGRES_DB=vorlesung
```

### Step 2 – Reference Variables in docker-compose.yml

Replace the hard-coded values in the `postgres` service:

```yaml
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
```

Also update `api/main.py` to read the password from the environment:

```python
import os

DB_CONFIG = {
    "dbname": os.environ.get("POSTGRES_DB", "vorlesung"),
    "user": os.environ.get("POSTGRES_USER", "vorlesung"),
    "password": os.environ.get("POSTGRES_PASSWORD", ""),
    "host": "postgres",
    "port": 5432,
}
```

Add `env_file` to the `api` service in `docker-compose.yml`:

```yaml
  api:
    ...
    env_file: .env
```

### Step 3 – Gitignore .env

```bash
echo ".env" >> .gitignore
```

### Step 4 – Restart and Verify

```bash
docker compose down -v
docker compose up -d
curl http://localhost:8000/studenten
```

> The response should be identical to before.

### Step 5 – Commit

```bash
git add docker-compose.yml api/main.py .gitignore
git commit -m "feat: move credentials to .env file"
git push
```

> **Do not add `.env` to the commit.** Confirm with `git status` that it
> is untracked.

> **Screenshot 9:** Take a screenshot showing `git status` confirming
> `.env` is not staged, and the working `curl` response.
>
> <img width="1237" height="360" alt="image" src="https://github.com/user-attachments/assets/5c3b4f3d-828a-436a-bbdc-d4580aa5720a" />


### Questions for Section 8

**Question 8.1:** A teammate clones your repository and runs
`docker compose up -d`. The application fails because `.env` is missing.
What is the standard practice to document which variables are required
without committing the actual secrets?

> The standard practice is to commit an example file such as `.env.example` with the required variable names but fake or empty values. The real `.env` stays local and is not committed.

**Question 8.2:** Even with `.env` excluded from git, the password is still
stored in plain text on disk. Name one mechanism Docker provides for
production-grade secret management that avoids plain-text env files entirely.

> Docker provides Docker Secrets, for example with `docker secret` in Docker Swarm. This is better for production because secrets are managed separately instead of being stored as plain text in an `.env` file.

---

## 9 – Multi-Stage Build

A Python image that includes `pip`, build tools, and cache is larger than
necessary. A Multi-Stage Build separates dependency installation from the
final runtime image.

### Step 1 – Create an api/Dockerfile

```bash
vim api/Dockerfile
```

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Add pyproject.toml

```bash
vim api/pyproject.toml
```

```toml
[project]
name = "studenten-api"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi",
    "uvicorn[standard]",
    "psycopg2-binary",
]
```

### Step 3 – Switch the api Service to Use the Build

Replace the `image:` key with `build:` in the `api` service:

```yaml
  api:
    build: ./api
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    env_file: .env
    networks:
      - backend
```

Remove the `volumes: - ./api:/app` bind mount and the `command:` key — they
were only needed for the quick-start approach.

### Step 4 – Build and Test

```bash
docker compose down -v
docker compose build
docker images    # compare sizes
docker compose up -d
curl http://localhost:8000/studenten
```

> **Screenshot 10:** Take a screenshot showing `docker images` with the
> final image size and the working `curl` response.
>
> <img width="1238" height="366" alt="image" src="https://github.com/user-attachments/assets/8849be44-2252-44e6-8654-c2fe3306f2bf" />


### Step 5 – Commit

```bash
git add api/Dockerfile api/pyproject.toml docker-compose.yml
git commit -m "feat: multi-stage Dockerfile for slim production image"
git push
```

### Questions for Section 9

**Question 9.1:** `COPY --from=builder /app/.venv .venv` copies the virtual
environment from the builder stage. The final image does not contain `pip` or
`uv`. What security advantage does this provide?

> The final image has fewer tools installed, so there is a smaller attack surface. If `pip` and `uv` are not present, an attacker has fewer easy tools available to install or modify packages inside the running container.

**Question 9.2:** The builder stage installs dependencies from `pyproject.toml`
before copying the application code. Why does this ordering improve build
cache efficiency when you frequently change only `main.py`?

> Docker can cache the dependency installation layer because `pyproject.toml` changes less often than the application code. If I only change `main.py`, Docker can reuse the dependency layer and rebuild much faster.

---

## 10 – Non-Root User

Containers run as `root` by default. If an attacker escapes the container,
they have root on the host.

### Step 1 – Add a Non-Root User to the Dockerfile

Open `api/Dockerfile` and add the user before the `CMD`:

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install uv && uv sync --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv .venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
RUN adduser --disabled-password --gecos "" appuser
USER appuser
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Step 2 – Rebuild and Verify

```bash
docker compose build
docker compose up -d
docker compose exec api whoami
```

> Expected output: `appuser`

> **Screenshot 11:** Take a screenshot showing `docker compose exec api whoami`
> returning `appuser`.
>
> <img width="676" height="63" alt="image" src="https://github.com/user-attachments/assets/5d08934e-a6c8-4b1c-9782-52b2191603fd" />


### Step 3 – Commit

```bash
git add api/Dockerfile
git commit -m "feat: run api container as non-root appuser"
git push
```

### Questions for Section 10

**Question 10.1:** The `USER appuser` instruction is placed after
`COPY . .`. Why would placing it *before* `COPY` cause a permission problem?

> If `USER appuser` is placed before `COPY`, the non-root user may not have permission to write files into the application directory or may get files with wrong ownership. Copying files first as root and switching to `appuser` afterward avoids this problem.

**Question 10.2:** State the **Principle of Least Privilege** in one
sentence, and name one other place in a typical web application stack
(outside of containers) where this principle is applied.

> The Principle of Least Privilege means that every user, service, or process should only get the permissions it really needs. Another example is using a database user that can only access the required tables instead of using a database superuser.

---

## 11 – Reflection

**Question A – The Monolith Anti-Pattern:**  
Section 6 of the lecture shows a Dockerfile that runs both PostgreSQL and
FastAPI in a single container. Describe two concrete operational problems
this causes in a production environment.

> Running PostgreSQL and FastAPI in one container makes it hard to restart, update, or scale them independently. It also mixes logs, health checks, backups, and failures, so one service can disturb the other.

**Question B – Volume vs. Bind Mount:**  
Compare named volumes and bind mounts. When is each type appropriate?

> Named volumes are managed by Docker and are best for persistent application data such as database files. Bind mounts connect a specific host folder to a container and are useful for development code, configuration files, or init scripts.

**Question C – Compose and Reproducibility:**  
A colleague says: "I can just write the `docker run` commands in a shell
script — why do I need `docker-compose.yml`?" Give two specific advantages
of Compose over a shell script of `docker run` commands.

> Compose gives one clear, version-controlled YAML file that describes services, networks, volumes, ports, and dependencies. It is also easier to reproduce on another machine than a shell script because Compose automatically manages service names, networks, and volumes.

**Question D – The Complete Chain:**  
You have now built and containerised the full stack: PostgreSQL in a
container with a named volume and init script → FastAPI in a slim
non-root image → both orchestrated by Docker Compose with credentials
in `.env`. Describe in two sentences what each layer contributes to
**portability** and **security**.

> PostgreSQL in a container with a named volume and init script makes the database portable and keeps its data persistent. The FastAPI slim non-root image improves security by reducing tools and privileges, while Docker Compose and `.env` make the stack easy to start and keep credentials out of the repository.

---

## Further Reading

- [Docker – Get started](https://docs.docker.com/get-started/)
- [Docker – Volumes](https://docs.docker.com/storage/volumes/)
- [Docker – Networking overview](https://docs.docker.com/network/)
- [Docker Compose – Reference](https://docs.docker.com/compose/compose-file/)
- [Docker – Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [postgres Docker Hub – Environment variables](https://hub.docker.com/_/postgres)
- Lecture 09 handout
