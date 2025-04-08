Okay, here is a detailed step-by-step guide on setting up a development environment for a FastAPI project using Docker Compose, integrating PostgreSQL for the database, Redis for caching/tasks, and Alembic for database migrations. This guide focuses on the structure, commands, and configuration descriptions, without writing the actual application code.

**Assumptions:**

- You have a basic understanding of FastAPI, PostgreSQL, Redis, and Alembic concepts.
- Your FastAPI application uses SQLAlchemy for database interaction (required for Alembic).
- You are using a dependency manager like Poetry or pip with `requirements.txt`.

---

**Phase 1: Prerequisites and Project Structure**

1.  **Install Prerequisites:** Ensure you have Docker and Docker Compose installed on your local machine.

    - Follow the official installation guides for your operating system:
      - Docker: [https://docs.docker.com/engine/install/]([invalid%20URL%20removed])
      - Docker Compose: [https://docs.docker.com/compose/install/]([invalid%20URL%20removed])

2.  **Define Project Structure:** Create a root directory for your project. A suggested structure might look like this:

    ```
    your-fastapi-project/
    ├── app/                  # Your FastAPI application code
    │   ├── __init__.py
    │   ├── main.py           # FastAPI app instance
    │   ├── core/             # Configuration, settings
    │   ├── db/               # Database session, base models
    │   ├── models/           # SQLAlchemy models
    │   ├── schemas/          # Pydantic schemas
    │   ├── api/              # API routers/endpoints
    │   └── crud/             # Database interaction logic (optional)
    ├── alembic/              # Alembic migration environment (created later)
    ├── alembic.ini           # Alembic configuration (created later)
    ├── .env                  # Environment variables (DO NOT commit sensitive data)
    ├── .gitignore
    ├── Dockerfile            # Instructions to build the FastAPI service image
    ├── docker-compose.yaml   # Defines all services (FastAPI, Postgres, Redis)
    ├── pyproject.toml        # If using Poetry
    ├── poetry.lock           # If using Poetry
    └── requirements.txt      # If using pip
    ```

---

**Phase 2: FastAPI Application Setup (Conceptual)**

_This phase describes what your application code needs; you won't run commands here yet._

1.  **Core Application (`app/main.py`):** Define your FastAPI app instance.
2.  **Configuration (`app/core/config.py`):** Set up configuration management (e.g., using Pydantic's `BaseSettings`) to read database URLs, Redis URLs, and other settings from environment variables.
3.  **Database Session (`app/db/session.py`):** Configure SQLAlchemy to connect to PostgreSQL using the URL from your settings. Define your `declarative_base()` (e.g., `Base`).
4.  **Models (`app/models/`):** Define your SQLAlchemy models, inheriting from the `Base` created above.
5.  **Redis Connection:** Implement logic (e.g., in `app/db/cache.py` or similar) to connect to Redis using the Redis URL from your settings.
6.  **Dependency File:** Ensure your `pyproject.toml` or `requirements.txt` includes:
    - `fastapi`, `uvicorn`
    - `sqlalchemy`
    - `alembic`
    - Your PostgreSQL driver (`psycopg2-binary` or `asyncpg`)
    - Your Redis client (`redis` or `aioredis`)
    - Any other necessary libraries.

---

**Phase 3: Dockerfile for FastAPI Service**

_Create a file named `Dockerfile` in your project root._

1.  **Base Image:** Start with an official Python base image matching your development version.
    ```dockerfile
    # Dockerfile
    FROM python:3.10-slim
    ```
2.  **Environment Variables:** Set non-secret environment variables. Prevent Python from writing bytecode files and ensure output is unbuffered.
    ```dockerfile
    ENV PYTHONDONTWRITEBYTECODE 1
    ENV PYTHONUNBUFFERED 1
    ```
3.  **Set Working Directory:** Define the working directory inside the container.
    ```dockerfile
    WORKDIR /app
    ```
4.  **Install OS Dependencies (If Needed):** Install packages required by your Python dependencies (e.g., `build-essential`, `libpq-dev` for `psycopg2`).
    ```dockerfile
    # Example for psycopg2:
    # RUN apt-get update && apt-get install -y build-essential libpq-dev && rm -rf /var/lib/apt/lists/*
    ```
5.  **Install Python Dependencies:**
    - **Using Poetry:** Copy `pyproject.toml` and `poetry.lock`. Install Poetry and then the dependencies.
      ```dockerfile
      # Using Poetry (example)
      RUN pip install poetry
      COPY pyproject.toml poetry.lock ./
      RUN poetry config virtualenvs.create false && poetry install --no-dev --no-interaction --no-ansi
      ```
    - **Using pip:** Copy `requirements.txt`. Install the dependencies.
      ```dockerfile
      # Using pip (example)
      COPY requirements.txt ./
      RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements.txt
      ```
6.  **Copy Application Code:** Copy your `app` directory into the container's working directory.
    ```dockerfile
    COPY ./app /app/app
    ```
7.  **Expose Port:** Expose the port your FastAPI application will run on inside the container (e.g., 8000).
    ```dockerfile
    EXPOSE 8000
    ```
8.  **Default Command:** Specify the command to run when the container starts (using Uvicorn).
    ```dockerfile
    # Adjust app.main:app, host, and port as needed
    CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```

---

**Phase 4: Docker Compose Configuration**

_Create a file named `docker-compose.yaml` in your project root._

1.  **Version:** Specify the Docker Compose file version.

    ```yaml
    # docker-compose.yaml
    version: '3.8'
    ```

2.  **Services:** Define the containers: `api` (FastAPI), `db` (Postgres), `redis`.

    ```yaml
    services:
      # PostgreSQL Service
      db:
        image: postgres:15 # Use a specific version
        container_name: fastapi_db
        volumes:
          - postgres_data:/var/lib/postgresql/data/ # Persist data
        environment:
          POSTGRES_USER: ${POSTGRES_USER:-user} # Use env vars from .env file or defaults
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
          POSTGRES_DB: ${POSTGRES_DB:-appdb}
        ports:
          - '${POSTGRES_PORT:-5432}:5432' # Optional: Expose port to host for direct access
        networks:
          - fastapi_network # Add network definition later

      # Redis Service
      redis:
        image: redis:7 # Use a specific version
        container_name: fastapi_redis
        ports:
          - '${REDIS_PORT:-6379}:6379' # Optional: Expose port to host
        networks:
          - fastapi_network

      # FastAPI Application Service
      api:
        container_name: fastapi_api
        build: . # Build from Dockerfile in the current directory
        command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload # Use --reload for development
        volumes:
          - ./app:/app/app # Mount local app code into container for live reload
        ports:
          - '${API_PORT:-8000}:8000' # Expose API port to host
        environment:
          # Pass connection details and other settings to the application
          DATABASE_URL: postgresql://${POSTGRES_USER:-user}:${POSTGRES_PASSWORD:-password}@db:5432/${POSTGRES_DB:-appdb}
          REDIS_URL: redis://redis:6379/0
          # Add any other environment variables your app needs
        depends_on:
          - db # Ensure db starts before api
          - redis # Ensure redis starts before api
        networks:
          - fastapi_network
    ```

3.  **Volumes:** Define named volumes for data persistence.

    ```yaml
    volumes:
      postgres_data: # Volume for PostgreSQL data
    ```

4.  **Networks:** Define a custom network for services to communicate easily using service names.

    ```yaml
    networks:
      fastapi_network:
        driver: bridge
    ```

5.  **Create `.env` File:** Create a `.env` file in your project root to store environment variables (Docker Compose automatically loads this). **Do not commit sensitive credentials.**

    ```ini
    # .env
    POSTGRES_USER=myuser
    POSTGRES_PASSWORD=mypassword
    POSTGRES_DB=myappdb
    POSTGRES_PORT=5432 # Optional: Port for host access
    REDIS_PORT=6379    # Optional: Port for host access
    API_PORT=8000      # Port for host access
    ```

    _Your FastAPI app configuration (`app/core/config.py`) should read these variables (e.g., `DATABASE_URL`, `REDIS_URL`). Notice how the `DATABASE_URL` inside the `api` service uses the service name `db` as the hostname._

---

**Phase 5: Alembic Initialization and Configuration**

1.  **Build the API Image (without starting):** This is needed so we can run commands inside it.

    ```bash
    docker-compose build api
    ```

2.  **Initialize Alembic:** Run the `alembic init` command _inside_ a temporary container based on your `api` service image. This creates `alembic.ini` and the `alembic/` directory _on your host_ because we'll mount the current directory.

    ```bash
    docker-compose run --rm --entrypoint "" api \
      sh -c "alembic init alembic"
    # If using poetry and it's not found, try:
    # docker-compose run --rm --entrypoint "" api \
    #   sh -c "/root/.local/bin/alembic init alembic" # Adjust path if needed
    ```

    - `--rm`: Removes the container after the command finishes.
    - `--entrypoint ""`: Overrides the default `CMD`/`ENTRYPOINT` from the Dockerfile.
    - `api`: The service name from `docker-compose.yaml`.
    - `sh -c "alembic init alembic"`: The command to run inside the container. Adjust the path to `alembic` if needed (e.g., inside a Poetry venv if not installed globally in the image).

3.  **Configure `alembic.ini`:** Edit the generated `alembic.ini`.

    - Find the `sqlalchemy.url` line.
    - **Important:** Set it up to read from the _same environment variable_ your application uses. Alembic commands run via `docker-compose exec` will inherit environment variables from the `docker-compose.yaml`.

      ```ini
      # alembic.ini
      # Option 1: Reference environment variable directly (requires specific config in env.py)
      # sqlalchemy.url =postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}

      # Option 2: Leave placeholder and load dynamically in env.py (Recommended)
      sqlalchemy.url = driver://user:pass@host/dbname
      ```

4.  **Configure `alembic/env.py`:** Edit `alembic/env.py`.

    - **Import Base and Models:** Ensure the script imports your SQLAlchemy `Base` and all your models from your `app` directory (adjust `sys.path` if necessary, similar to the example in the previous Alembic answer).
    - **Set `target_metadata`:** Assign your imported `Base.metadata` to the `target_metadata` variable.
    - **Configure Database URL (Crucial for Docker):** Ensure the `run_migrations_online` function correctly gets the database URL. The best way is to read the `DATABASE_URL` environment variable passed by Docker Compose.

      ```python
      # alembic/env.py - Inside run_migrations_online() function

      # Add this at the top of the function or where 'config' is defined
      import os
      db_url = os.getenv('DATABASE_URL') # Read from environment set by docker-compose

      # Option A: If you directly configure the engine/connectable
      connectable_config = config.get_section(config.config_ini_section)
      connectable_config['sqlalchemy.url'] = db_url # Override ini setting
      connectable = engine_from_config(
          connectable_config,
          prefix="sqlalchemy.",
          poolclass=pool.NullPool,
      )

      # Option B: If context.configure uses the URL directly
      # context.configure(url=db_url, ...) # Ensure url is passed here

      # Rest of the function...
      with connectable.connect() as connection:
          context.configure(
              connection=connection, target_metadata=target_metadata
          )
          with context.begin_transaction():
              context.run_migrations()
      ```

---

**Phase 6: Running the Environment**

1.  **Build and Start Services:** Build the images (if not already done or changed) and start all services in detached mode.

    ```bash
    docker-compose up --build -d
    ```

    - `--build`: Forces Docker Compose to rebuild images if the Dockerfile or context has changed.
    - `-d`: Runs containers in the background (detached mode).

2.  **Check Container Status:**

    ```bash
    docker-compose ps
    ```

    _(All services: `db`, `redis`, `api` should show 'Up' or 'running')_

3.  **View Logs:**

    ```bash
    docker-compose logs -f # View logs from all services and follow
    docker-compose logs -f api # View logs only for the api service
    ```

---

**Phase 7: Running Alembic Migrations**

Run Alembic commands _inside_ the running `api` container using `docker-compose exec`.

1.  **Create Initial Migration (or subsequent ones):**

    ```bash
    docker-compose exec api alembic revision --autogenerate -m "Initial database structure"
    # If using poetry and alembic isn't in default PATH:
    # docker-compose exec api /root/.local/bin/alembic revision --autogenerate -m "Initial database structure"
    ```

    - `api`: The service name where Alembic and your models are installed.
    - Check the generated script inside the `alembic/versions/` directory on your host machine.

2.  **Apply Migrations:**

    ```bash
    docker-compose exec api alembic upgrade head
    # Adjust path if needed:
    # docker-compose exec api /root/.local/bin/alembic upgrade head
    ```

    This command connects to the `db` service (using the `DATABASE_URL` environment variable available inside the `api` container) and applies the migrations.

3.  **Other Alembic Commands:** Use the same pattern:

    ```bash
    docker-compose exec api alembic current
    docker-compose exec api alembic history
    docker-compose exec api alembic downgrade -1
    ```

---

**Phase 8: Accessing Services**

- **FastAPI App:** Access via `http://localhost:8000` (or the `API_PORT` you set in `.env`).
- **PostgreSQL (Optional Direct Access):** If you exposed the port, connect using a client like `psql` or DBeaver to `localhost:5432` (or the `POSTGRES_PORT` you set) using the credentials from your `.env` file.
- **Redis (Optional Direct Access):** If you exposed the port, connect using `redis-cli -p 6379` (or the `REDIS_PORT`).

---

**Phase 9: Stopping the Environment**

```bash
docker-compose down
# To remove volumes (WARNING: deletes postgres data):
# docker-compose down -v
```

This detailed guide provides the structure and commands needed to set up your described development environment using Docker Compose. Remember to replace placeholders and adjust paths/commands based on your specific project setup and choices (like Poetry vs. pip).
