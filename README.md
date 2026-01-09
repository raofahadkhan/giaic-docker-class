# FastAPI + OpenAI Agents SDK

A minimal FastAPI server using OpenAI Agents SDK with Google Gemini.

## Setup

```bash
uv sync
```

Create `.env` file with your Gemini API key:
```
GEMINI_API_KEY=your_key_here
```

## Run

```bash
uv run main.py
```

Server runs at http://localhost:8000

API docs: http://localhost:8000/docs

## Usage

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What is 25 * 4"}'
```

## Docker

### What is Docker?

Docker allows you to package your application with all its dependencies into a standardized unit called a **container**. This ensures your app runs the same way on any machine.

### Prerequisites

- Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Make sure Docker is running (check the system tray icon)

### Building the Docker Image

A Docker **image** is like a template/blueprint for your application.

```bash
docker build -t fastapi-agents .
```

**Explanation:**
- `docker build` - Command to build an image
- `-t fastapi-agents` - Tags/names the image as "fastapi-agents"
- `.` - Look for the Dockerfile in the current directory

### Running the Container

A **container** is a running instance of an image.

**Option 1: Pass environment variable directly**
```bash
docker run -p 8000:8000 -e GEMINI_API_KEY=your_key_here fastapi-agents
```

**Option 2: Use your .env file**
```bash
docker run -p 8000:8000 --env-file .env fastapi-agents
```

**Explanation:**
- `docker run` - Create and start a container
- `-p 8000:8000` - Map port 8000 on your machine to port 8000 in the container
- `-e GEMINI_API_KEY=...` - Set an environment variable
- `--env-file .env` - Load environment variables from a file
- `fastapi-agents` - The image name to run

### Running in Background (Detached Mode)

```bash
docker run -d -p 8000:8000 --env-file .env --name my-api fastapi-agents
```

- `-d` - Run in detached/background mode
- `--name my-api` - Give the container a name for easy reference

### Development Mode with Bind Mount

Use a bind mount to enable live code reloading - changes to `main.py` on your machine will be reflected in the container without rebuilding.

**On Windows (use absolute path):**
```bash
docker run -d -p 8000:8000 --env-file .env --name my-api -v "%cd%\main.py:/app/main.py" fastapi-agents
```

**On Linux/Mac:**
```bash
docker run -d -p 8000:8000 --env-file .env --name my-api -v ./main.py:/app/main.py fastapi-agents
```

- `-v` - Bind mount that maps your local `main.py` to the container's `/app/main.py`
- On Windows, use `%cd%\main.py` to get the absolute path, or use the full path: `"C:\Users\YourName\path\to\main.py"`

**How it works:**
1. The container uses your local `main.py` instead of the one baked into the image
2. When you edit `main.py`, uvicorn's `--reload` flag detects changes and restarts the server
3. No need to rebuild the image for code changes

**Note:** If your path contains spaces, make sure to quote the entire volume mount path.

### Useful Docker Commands

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop my-api

# Start a stopped container
docker start my-api

# View container logs
docker logs my-api

# Follow logs in real-time
docker logs -f my-api

# Remove a container
docker rm my-api

# List images
docker images

# Remove an image
docker rmi fastapi-agents
```

### Understanding the Dockerfile

```dockerfile
FROM python:3.11-slim              # Base image with Python 3.11

WORKDIR /app                       # Set working directory inside container

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv  # Install uv

COPY pyproject.toml uv.lock ./     # Copy dependency files first (for caching)

RUN uv sync --frozen --no-dev      # Install dependencies

COPY main.py .                     # Copy application code

EXPOSE 8000                        # Document which port the app uses

CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]  # Start command
```

**Why copy dependencies before code?**
Docker caches each layer. If your code changes but dependencies don't, Docker reuses the cached dependency layer, making rebuilds faster.

---

## Docker Volumes (PostgreSQL Example)

### What are Volumes?

Volumes are Docker-managed storage that persists data even when containers are removed. This is critical for databases - you don't want to lose your data!

### Step 1: PostgreSQL WITHOUT a Volume (The Problem)

First, let's see what happens when we run PostgreSQL without a volume.

**Start PostgreSQL container:**
```bash
docker run -d --name my-postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres
```

**Create a database and add some data:**
```bash
# Create database
docker exec -it my-postgres psql -U postgres -c "CREATE DATABASE school;"

# Create table
docker exec -it my-postgres psql -U postgres -d school -c "CREATE TABLE students (id SERIAL PRIMARY KEY, name TEXT);"

# Insert data
docker exec -it my-postgres psql -U postgres -d school -c "INSERT INTO students (name) VALUES ('Ali'), ('Sara'), ('Ahmed');"

# Verify data exists
docker exec -it my-postgres psql -U postgres -d school -c "SELECT * FROM students;"
```

**Output:**
```
 id | name
----+-------
  1 | Ali
  2 | Sara
  3 | Ahmed
```

**Now remove and recreate the container:**
```bash
# Remove container
docker rm -f my-postgres

# Start a new container
docker run -d --name my-postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres

# Try to access the data - DATABASE IS GONE!
docker exec -it my-postgres psql -U postgres -c "\l"
```

The `school` database no longer exists. All data was lost when we removed the container.

---

### Step 2: PostgreSQL WITH a Volume (The Solution)

Now let's run PostgreSQL with a volume to persist the data.

**Start PostgreSQL container with a volume:**
```bash
docker run -d --name my-postgres \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  -v postgres-data:/var/lib/postgresql/data \
  postgres
```

**Explanation:**
- `-v postgres-data:/var/lib/postgresql/data` - Creates a volume named `postgres-data` and mounts it to PostgreSQL's data directory
- The volume is auto-created if it doesn't exist

**Create database and add data (same as before):**
```bash
docker exec -it my-postgres psql -U postgres -c "CREATE DATABASE school;"
docker exec -it my-postgres psql -U postgres -d school -c "CREATE TABLE students (id SERIAL PRIMARY KEY, name TEXT);"
docker exec -it my-postgres psql -U postgres -d school -c "INSERT INTO students (name) VALUES ('Ali'), ('Sara'), ('Ahmed');"
docker exec -it my-postgres psql -U postgres -d school -c "SELECT * FROM students;"
```

**Now remove and recreate the container:**
```bash
# Remove container
docker rm -f my-postgres

# Start a NEW container with the SAME volume
 docker run -d --name my-postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 -v postgres-data:/var/lib/postgresql postgres

# Check data - IT'S STILL THERE!
docker exec -it my-postgres psql -U postgres -d school -c "SELECT * FROM students;"
```

**Output:**
```
 id | name
----+-------
  1 | Ali
  2 | Sara
  3 | Ahmed
```

Data persists because it's stored in the volume, not inside the container.

---

### Volume Commands

```bash
# List all volumes
docker volume ls

# Inspect a volume (see details and location)
docker volume inspect postgres-data

# Remove a volume (WARNING: deletes all data!)
docker volume rm postgres-data

# Remove all unused volumes
docker volume prune
```

---

### Visual Summary

```
WITHOUT VOLUME:                         WITH VOLUME:
┌─────────────────┐                     ┌─────────────────┐
│   Container     │                     │   Container     │
│ ┌─────────────┐ │                     │ ┌─────────────┐ │
│ │ PostgreSQL  │ │                     │ │ PostgreSQL  │ │
│ │   Data      │ │                     │ │ /var/lib/.. │─┼───┐
│ └─────────────┘ │                     │ └─────────────┘ │   │
└─────────────────┘                     └─────────────────┘   │
        │                                                     │
        ▼                                        ┌────────────▼────────────┐
   Container removed                             │      Volume:            │
   = Data LOST                                   │    postgres-data        │
                                                 │   = Data PERSISTS       │
                                                 └─────────────────────────┘
```

---

### Bind Mount vs Volume - Quick Comparison

| Feature | Bind Mount | Volume |
|---------|------------|--------|
| **Syntax** | `-v ./local/path:/container/path` | `-v volume-name:/container/path` |
| **Managed by** | You (host filesystem) | Docker |
| **Use case** | Development, live code editing | Databases, persistent app data |
| **Data location** | You choose the path | Docker manages it |
| **Portability** | Tied to host paths | Works on any machine |
