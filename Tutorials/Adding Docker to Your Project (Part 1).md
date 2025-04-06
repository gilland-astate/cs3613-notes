We are going to use [[Docker - Container Development|Docker]] in order to streamline and ensure our projects can be deployed and developed in the same environment.
## Install Docker
Follow the official documentation for [installing Docker](https://docs.docker.com/engine/install/). 
## Enable Docker Compose for your Project
1. Create a file named `docker-compose.yml` in your main directory of your project with the following contents:
```yaml
# docker-compose.yml
services:
	web:
		build:
			context: .
			dockerfile: django/Dockerfile
		command: sh -c "python manage.py collectstatic --noinput && python manage.py migrate && python manage.py runserver 0.0.0.0:8000"
		image: "web"
		stdin_open: true
		tty: true
		volumes:
		- ./django:/app
		- ./django/staticfiles:/app/staticfiles
	ports:
		- "8000:8000"
	depends_on:
		db:
			condition: service_healthy
	env_file:
		- .env
	environment:
		- PYTHONPATH=/app
	db:
		image: postgres:15-alpine
		expose:
			- "5449"
		ports:
			- "5449:5449"
		env_file:
			- .env
		volumes:
			- postgres_data:/var/lib/postgresql/data/
		healthcheck:
			test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
			timeout: 5s
			interval: 1s
			retries: 10

volumes:
	postgres_data:
	django_logs:
```
2. You should notice a couple of things in doing this: 
	- There is the line `dockerfile: django/Dockerfile` which tells compose where the setup is for your web server container.
	- We are changing the way we are storing our data from the sqlite database into our Postgres container `db`.
	- We also enabled the ability to use environment variable to store our secrets.
## Create our Dockerfile for Web Container
1. Create the Dockerfile in our django directory with the following contents:
```Dockerfile
# Create our initial image with our dependancies
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim AS builder
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
ENV UV_PYTHON_DOWNLOADS=0

WORKDIR /deps
RUN --mount=type=cache,target=/root/.cache/uv \
	--mount=type=bind,source=./uv.lock,target=uv.lock \
	--mount=type=bind,source=./pyproject.toml,target=pyproject.toml \
	uv sync --frozen --no-install-project --no-dev

# Then, use a final image without uv
FROM python:3.13-slim-bookworm
RUN useradd -m app

# Copy the virtual environment from builder
COPY --from=builder --chown=app:app /deps/.venv /deps/.venv
ENV PATH="/deps/.venv/bin:$PATH"
WORKDIR /app

# Run the application
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
## Setup for using Postgres Database
Our Docker Compose file already imports an appropriate image and exposes the ports we need to connect, however we need to make our Django application work appropriately with Postgres instead of SQLite.
1. Add the postgres add-in to your python project to enable interface with Postgres:
```bash
uv add "psycopg[binary,pool]" django-environ
```
2. Edit the `settings.py` file for your project and the following near the top of the file:
```python
import environ
env = environ.Env()
```
2. Still in the `settings.py` file update the `DATABASES` setting to the following:
```python
DATABASES = {
	"default": env.db(),
} 
```
3. Add the following to a file named `.env` in your base project directory and add the information filling in some non-default values for the bracketed areas below (<>).
```toml
# Postgres
POSTGRES_USER="<username>"
POSTGRES_PASSWORD="<password>"
POSTGRES_DB="<database name>"
DATABASE_URL="postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}"
```
4. Now that we have our setup appropriately you can delete your `db.sqlite3` file. If you need to reset your database from here on out you will need to remove the volume of your database container.
## Setup Our Django Secrets and Environment Variables
Since we now have the ability to environment variables we can start to secure our application for future production deployment as well as allow our project to be checked into source control.
1. Let's start by updating our `.gitignore` file. Open your file and add the following:
```toml
# Python-generated files
__pycache__/
*.py[oc]
build/
dist/
wheels/
*.egg-info

# VSCode
.vscode/
  
# Virtual environments
.venv

# Environment variables
.env
.env.prod

# Static files
django/staticfiles

# Jupyter Notebook checkpoints
.ipynb_checkpoints

# macOS files
.DS_Store

# Logs
*.log

# Temporary files
*.tmp
*.temp

# Node.js modules
node_modules/

# Ruff cache
.ruff_cache/
```
2. Now that we have ensured that our `.env` file will not be checked into source control let's add a bit more information into it. This file will represent our dev settings:
```toml
# This is our DEVELOPMENT environment file

# General
DEBUG=1

# Django
DJANGO_ALLOWED_HOSTS="*"
DJANGO_SECRET_KEY="<copy your insecure key here>"
DJANGO_TIME_ZONE=America/Chicago
```
3. Since we are adding these settings to our environment, we can update out `settings.py` file to decouple these settings from the source and instead move these into the environment and allow users to setup their own settings. Change the following options in your setting to the following:
```python
SECRET_KEY = env.str("DJANGO_SECRET_KEY")
DEBUG = env.bool("DEBUG", default=False)

ALLOWED_HOSTS = env.list("DJANGO_ALLOWED_HOSTS", default=[""])

# This one is near the bottom of the file before the static files section
TIME_ZONE = env.str("DJANGO_TIME_ZONE", default="UTC")
```
4. The last thing we need to talk about with our docker setup before moving forward with everything is we need to make sure our static files are collected appropriately so they can be loaded into our docker container. As such edit `settings.py` and make the Static Files section of your settings look as such:
```python
# Static files (CSS, JavaScript, Images)
STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
STATICFILES_DIRS = [
	BASE_DIR / "static",
]
```
## Start Your Docker Containers!
1. Since we are using docker compose this makes everything very easy! Simply run the command from your base directory (where the compose file is):
```bash
docker compose up
```
2. This will download and start up your containers, after which you should be presented with the typical terminal after launching using `runserver`.
3. At this point if you want to leave your server running, but lose the access to the error logs from your console (these are still available via docker) you can run:
```bash
docker compose up -d
```