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
