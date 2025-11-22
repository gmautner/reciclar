# Requirements



## Web Application

### Framework

Use Python FastAPI with HTMX, Tailwind CSS and DaisyUI.

For the database, use PostgreSQL. Credentials should either be sourced from an `.env` file (make sure to exclude it in `.gitignore`) or from environment variables.

Pages (or partials) should be retrieved using this schematic approach:

```text
[User Click]
     ↓
 HTMX sends AJAX (hx-post, hx-get)
     ↓
 FastAPI endpoint
     ↓
 SQLModel session → DB
     ↓
 Render Jinja2 partial → HTML
     ↓
 HTMX swaps snippet into DOM
```

Make sure to use up to date versions of the libraries and frontend components.

### Authentication

Users may sign up and login using a Google account.

The first user to sign up should be the admin. After that, any subsequent user that signs up will be subject to the admin's approval to access the system.

## Docker packaging

Run with `uvicorn`. No need to package Postgres as it will be provided externally. Include the `cmk` CLI in the Docker image.

Provide instructions in the README on how to build and run the Docker image, including how to set the environment variables and local Postgres container.
