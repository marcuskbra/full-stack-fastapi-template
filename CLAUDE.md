# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Development Commands

### Full Stack Development
```bash
# Start the entire stack (preferred method)
docker compose watch

# Alternative: Start specific services
docker compose up -d backend frontend db adminer

# View logs
docker compose logs -f [service_name]

# Stop services
docker compose down
docker compose down -v  # Also removes volumes/data
```

### Backend Development
```bash
# Setup backend environment (from ./backend/)
uv sync
source .venv/bin/activate

# Run backend locally (outside Docker)
cd backend
fastapi dev app/main.py  # Development with auto-reload
fastapi run app/main.py  # Production mode

# Run tests
bash ./scripts/test.sh
# Or within running container:
docker compose exec backend bash scripts/tests-start.sh
docker compose exec backend bash scripts/tests-start.sh -x  # Stop on first error

# Database migrations
docker compose exec backend bash
alembic revision --autogenerate -m "Description of changes"
alembic upgrade head

# Type checking and linting
cd backend
uv run mypy app
uv run ruff check app
uv run ruff format app
```

### Frontend Development
```bash
# Setup frontend (from ./frontend/)
fnm use  # or nvm use
npm install

# Run frontend locally
npm run dev

# Build frontend
npm run build

# Generate API client (after backend changes)
./scripts/generate-client.sh
# Or manually:
cd frontend && npm run generate-client

# Linting
npm run lint

# Run E2E tests (requires backend running)
docker compose up -d --wait backend
npx playwright test
npx playwright test --ui  # Interactive mode
```

### Pre-commit Setup
```bash
# Install pre-commit hooks
uv run pre-commit install

# Run manually on all files
uv run pre-commit run --all-files
```

## High-Level Architecture

### Authentication & Authorization Flow
The app uses JWT-based authentication with a critical dependency chain:

1. **Token Generation** (`backend/app/api/routes/login.py`):
   - User submits credentials via OAuth2PasswordRequestForm
   - Password verified using bcrypt hashing
   - JWT token created with user UUID as subject

2. **Token Validation** (`backend/app/api/deps.py`):
   - `TokenDep` extracts JWT from Authorization header
   - `get_current_user()` validates token and retrieves user from DB
   - `CurrentUser` dependency injected into protected routes
   - `get_current_active_superuser()` adds role-based access control

3. **Frontend Token Management** (`frontend/src/main.tsx`):
   - Token stored in localStorage
   - OpenAPI.TOKEN async function provides token to all API calls
   - 401/403 responses trigger automatic logout and redirect

### Database Session & Transaction Pattern
All database operations use SQLModel with a specific session management pattern:

1. **Session Dependency** (`backend/app/api/deps.py:21-23`):
   - `get_db()` creates a session per request
   - `SessionDep` type annotation for dependency injection
   - Sessions automatically close after request completion

2. **Model Relationships**:
   - User ↔ Item: One-to-many with cascade delete
   - All models use UUID primary keys (not auto-increment integers)
   - Foreign keys use `ondelete="CASCADE"` for referential integrity

### API Client Generation Bridge
Critical for frontend-backend synchronization:

1. Backend OpenAPI schema exposed at `/api/v1/openapi.json`
2. `scripts/generate-client.sh` extracts schema from Python app
3. Frontend `openapi-ts` generates TypeScript client in `frontend/src/client/`
4. Must regenerate after ANY backend API changes

### Environment-Specific Routing
The app uses different routing strategies based on environment:

- **Local Development**: Services on different ports (backend:8000, frontend:5173)
- **Production**: Traefik proxy routes by subdomain (api.domain.com, dashboard.domain.com)
- **Private Routes**: Only available when `ENVIRONMENT=local` (see `backend/app/api/main.py:13-14`)

### Email Template System
Two-stage email template process:

1. **Source**: MJML files in `backend/app/email-templates/src/`
2. **Build**: HTML files in `backend/app/email-templates/build/`
3. **Usage**: Templates loaded by `backend/app/utils.py` for password recovery, etc.
4. **Important**: Use VS Code MJML extension to compile `.mjml` → `.html`

### Frontend Routing Architecture
TanStack Router with file-based routing:

1. **Route Files** (`frontend/src/routes/`):
   - `__root.tsx`: App wrapper with providers
   - `_layout.tsx`: Protected layout requiring authentication
   - `login.tsx`, `signup.tsx`: Public routes
   - `_layout/*.tsx`: Protected routes (admin, items, settings)

2. **Route Generation**:
   - Routes auto-generated into `routeTree.gen.ts`
   - Must restart dev server after adding new route files

### CORS Configuration
Multi-layer CORS handling:

1. **Backend** (`backend/app/main.py:24-31`): Reads from BACKEND_CORS_ORIGINS env var
2. **Default Origins**: Includes localhost:5173, localhost, localhost.tiangolo.com variants
3. **Production**: Must update BACKEND_CORS_ORIGINS with production frontend URL

### Docker Compose Layering
Three-layer Docker configuration:

1. **Base** (`docker-compose.yml`): Production-ready configuration
2. **Override** (`docker-compose.override.yml`): Development additions (hot reload, volume mounts)
3. **Traefik** (`docker-compose.traefik.yml`): Production proxy configuration

The override file enables:
- Source code volume mounting for hot reload
- `fastapi run --reload` instead of production server
- Traefik dashboard on port 8090
- Local development network configuration

### Testing Strategy

1. **Backend Tests** (`backend/tests/`):
   - API route tests with authenticated/unauthenticated scenarios
   - CRUD operation tests
   - Utility function tests
   - Run with pytest, coverage reports in `htmlcov/index.html`

2. **Frontend E2E Tests** (`frontend/tests/`):
   - Playwright-based browser automation
   - Test full user workflows
   - Requires backend running
   - Screenshots on failure

### Migration Management
Alembic migrations with auto-generation:

1. Models defined in `backend/app/models.py`
2. Alembic configured to auto-import models
3. `scripts/prestart.sh` runs migrations on container start
4. Migration files tracked in `backend/app/alembic/versions/`
5. To disable migrations: Uncomment `SQLModel.metadata.create_all(engine)` in `backend/app/core/db.py`

## Critical Configuration Files

- `.env`: Environment variables (secrets, database credentials, domains)
- `backend/app/core/config.py`: Pydantic settings with validation
- `frontend/.env`: Frontend-specific vars (VITE_API_URL)
- `.pre-commit-config.yaml`: Code quality enforcement

## Service URLs

### Local Development (default)
- Frontend: http://localhost:5173
- Backend API: http://localhost:8000
- API Docs: http://localhost:8000/docs
- Adminer: http://localhost:8080
- Traefik Dashboard: http://localhost:8090

### With Domain Configuration (DOMAIN=localhost.tiangolo.com)
- Frontend: http://dashboard.localhost.tiangolo.com
- Backend API: http://api.localhost.tiangolo.com
- All services use Traefik routing