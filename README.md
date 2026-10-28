# Trio Bank

Educational simulation of a Polish retail bank — accounts, transfers, ATM, interest and an admin panel — built with Java / Spring Boot, PostgreSQL and Angular.

> ⚠️ Educational project. No real money, no real personal data.

**Team:** Marko Zhenchuk · Hryhorii Martynkian · Kateryna Valentyuk

## Documentation

Everything about the project — requirements, data model, API contract, architecture, process and roadmap — is in [`/docs`](docs/README.md).

## Running locally

```bash
cp .env.example .env
docker compose up --build
```

- Frontend: http://localhost:4200
- Backend API: http://localhost:8080/api/v1
- Swagger UI: http://localhost:8080/swagger-ui.html

Details: [docs/10-devops.md](docs/10-devops.md#8-local-development).

## Repository layout

| Folder | Content |
|---|---|
| `backend/` | Spring Boot API |
| `frontend/` | Angular application |
| `e2e/` | Playwright tests |
| `docs/` | Project documentation |
