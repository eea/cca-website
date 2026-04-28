# Climate-ADAPT Development Guide for Agents

This repository contains the full Climate-ADAPT stack: a **Plone 6** backend and a **Volto** frontend, co-located in a monorepo.

- Production: https://climate-adapt.eea.europa.eu/
- Local: http://cca.localhost (when the root Docker Compose stack is running)

## Repository Layout

```
.
├── backend/                          # Plone backend
│   ├── sources/                      # Source-checked-out Plone add-ons
│   │   ├── eea.climateadapt/         # Main business-logic add-on (content types, translation, REST API)
│   │   ├── eea.volto.policy/         # EEA Volto policy
│   │   ├── eea.api.dataconnector/    # Data connector API
│   │   ├── eea.plotly/               # Plotly integration
│   │   ├── plone.restapi/            # RestAPI (fork/patch)
│   │   ├── collective.exportimport/  # Export/import
│   │   └── collective.volto.subsites/# Subsites
│   ├── instance/etc/                 # Zope configuration (zope.ini, relstorage.conf, …)
│   ├── climateadapt-async-translate/ # Node.js/BullMQ micro-service for async translation
│   ├── volto-blocks-converter/       # Micro-service: Volto blocks JSON <-> HTML
│   ├── scripts/                      # Helper scripts (e.g. create_site.py)
│   ├── docker-compose.yml            # Backend-only dev stack (Postgres, backend, redis, converter, async)
│   └── Makefile
├── frontend/                         # Volto frontend
│   ├── src/addons/volto-cca-policy/  # Main frontend add-on (components, theme, blocks)
│   ├── src/                          # Volto project source
│   ├── package.json
│   ├── jsconfig.json / mrs.developer.json
│   └── Makefile
├── docker-compose.yml                # Root stack: Traefik + frontend + backend + db
└── Makefile                          # Top-level orchestration
```

## How the Developer Runs Things

### Backend (Plone)

The developer runs the backend **from the Docker Compose stack in the foreground**.

1. The backend container is started (kept alive with `tail -f /dev/null`).
2. The developer `docker compose exec -it backend /bin/bash` into the container.
3. Inside the container, the Plone process is started in the foreground manually.

> **Important:** When you need to run Python scripts, tests, or management commands against the backend, they must be executed **inside the backend container**. Use `docker compose exec backend <command>` (or exec into an interactive shell first).

- Python with full environment inside the container: `/app/bin/python3`
- Backend listens on port `8080` inside the container.
- The site ID in production-like setups is `cca`.

### Frontend (Volto)

The developer runs the Volto process **in the foreground from a separate terminal**, directly on the host (not inside Docker).

- Uses `yarn` (not npm).
- Needs `NODE_OPTIONS="--max-old-space-size=16384"` for builds/starts.
- The main add-on lives at `frontend/src/addons/volto-cca-policy/`.

> **DO NOT read any `start_*` scripts in `frontend/` — they contain environment variables that must not be exposed.**

## Key Development Areas

1. **Automatic Translation Pipeline**
   - Triggered on content change/publish.
   - `climateadapt-async-translate` (BullMQ) queues jobs.
   - Plone calls the EU eTranslation service.
   - `volto-blocks-converter` converts Volto blocks to HTML for translation and back to JSON afterwards.
   - Auth between services and Plone: shared `TRANSLATION_AUTH_TOKEN`.

2. **Content Types & REST API**
   - Defined mainly in `backend/sources/eea.climateadapt/eea/climateadapt/`.
   - Custom REST API endpoints in `…/restapi/`.
   - Mission Signatory Profile, Country Profiles, ECDE, PAM, AST Navigation, etc.

3. **Frontend Components**
   - Custom views, blocks, and theme components in `frontend/src/addons/volto-cca-policy/src/components/`.
   - The add-on depends on several EEA Volto packages (design system, tabs, searchlib, embed, etc.).

## Common Commands

### Root-level (orchestration)

| Command | Purpose |
|---------|---------|
| `make stack-start` | Start the full root Docker Compose stack (Traefik, frontend image, backend image, Postgres) |
| `make stack-stop` | Stop the root stack |
| `make stack-rm` | Tear down the root stack **and remove the Postgres volume** |
| `make install` | Install both backend (local venv) and frontend dependencies |
| `make start` | Start backend + frontend locally (outside Docker) |

### Backend (inside the backend directory)

| Command | Purpose |
|---------|---------|
| `make install` | Create venv, run `mxdev`, install packages |
| `make start` | Start Plone on localhost:8080 (outside Docker) |
| `make test` | Run backend tests via `tox -e test` |
| `make check` | Lint/format via `tox -e lint` |
| `make i18n` | Update locales |

> **Inside the Docker container** use `/app/bin/python3` and the installed scripts in `/app/bin/`.

### Frontend (inside the frontend directory)

| Command | Purpose |
|---------|---------|
| `make install` | `yarn install` with increased memory limit |
| `make start` | Start Volto dev server on localhost:3000 |
| `make build` | Production build |
| `make test src/addons/volto-cca-policy` | Run Jest tests for the main add-on |
| `make omelette` | Create symlink `omelette -> node_modules/@plone/volto` |

> **Note:** The `frontend/omelette` symlink points to the Volto package source code and is useful as a quick reference for core Volto components, actions, and reducers when writing customizations.

## Plone Documentation & Working with Relations

### Checking Plone Documentation

The official Plone 6 documentation is available at `https://6.docs.plone.org/`. A machine-readable index of all pages is available at:

- **`https://6.docs.plone.org/llms.txt`** — a flat text file listing every documentation page URL. Use this to discover relevant sections (e.g., `backend/relations.md`, `plone.api/relation.md`, `volto/development/pluggables.md`).

When you need to verify how a Plone subsystem works (behaviors, workflows, relations, REST API, etc.), fetch the specific `.md` source file from the `_sources/` path (e.g., `https://6.docs.plone.org/_sources/backend/relations.md`).

### Working with `relatedItems` / Relations

Climate-ADAPT content types reuse the standard Plone behavior `plone.app.relationfield.behavior.IRelatedItems` for the `relatedItems` field. In Plone 6, the canonical way to create, read, or delete relations programmatically is through **`plone.api.relation`**:

```python
from plone import api

# Create a relation (updates the field and the relation catalog)
api.relation.create(source=obj_a, target=obj_b, relationship="relatedItems")

# Read relations
relations = api.relation.get(source=obj_a, relationship="relatedItems")

# Delete relations
api.relation.delete(source=obj_a, relationship="relatedItems")
```

**Key rules:**
- Always use `plone.api.relation` instead of manually constructing `z3c.relationfield.RelationValue` objects or calling `IIntIds` utilities directly. The API handles catalog updates, event firing, and schema synchronization automatically.
- The `relationship` parameter corresponds to the field name (`"relatedItems"` for the built-in behavior).
- If a relation is based on a `RelationChoice` or `RelationList` field, `api.relation.create/delete` will create or remove the `RelationValue` objects on the source object and update the `zc.relation` catalog accordingly.
- For more details, see the Plone docs: `backend/relations.md` and `plone.api/relation.md`.

## Testing

- **Backend:** `tox` drives the test suite. Run with `make test` from `backend/`.
- **Frontend:** Jest tests are configured per add-on. Run with `make test src/addons/volto-cca-policy` from `frontend/`. The `test` target captures the positional argument and sets `RAZZLE_JEST_CONFIG=<path>/jest-addon.config.js` before running `yarn test`.
  - To run tests **once** (non-interactive, no watch mode) and dump full output: `CI=true make test src/addons/volto-cca-policy`
- **Acceptance:** Cypress tests live in the add-on and can be run via the add-on's Makefile targets (`cypress-run`, `cypress-open`).

## Operations

- **Never delete heavy directories** (e.g. `frontend/`, `backend/`, `node_modules/`). When replacing a directory with a submodule or restructuring, **move it aside first** (`mv frontend frontend.bak`), then proceed. Restore or clean up only after confirming success.

## Conventions & Constraints

- **Do not read `frontend/start_*` scripts.** They contain sensitive environment variables.
- Backend commands that need the full Plone environment must run **inside the backend Docker container** (the developer runs Plone in the foreground from within the container).
- The frontend is run **on the host** in a separate terminal, in the foreground.
- The repo uses `yarn` (not `npm`) and `mrs-developer` for add-on management.
- Python code follows Plone standards (black, isort, codespell via `tox -e lint`).
- Zope configuration parsing: if writing standalone scripts that need to read `zope.conf`/`relstorage.conf`, use `Zope2.Startup.options.ZopeWSGIOptions` rather than plain `ZConfig.loadSchema` to avoid schema errors with Zope-specific keys.

## GitHub Automation Notes

### Editing Pull Requests

Some repos have legacy "Projects (classic)" settings that break `gh pr edit`. If you hit a GraphQL error like:

```
GraphQL: Projects (classic) is being deprecated in favor of the new Projects experience...
```

Fall back to the GitHub REST API using `gh api`:

```bash
gh api -X PATCH /repos/{owner}/{repo}/pulls/{pr_number} \
  -f title="New Title" \
  -f body="New Body"
```

For a multi-line body, use a heredoc:

```bash
gh api -X PATCH /repos/{owner}/{repo}/pulls/{pr_number} \
  -f title="New Title" \
  -f body="$(cat <<'EOF'
## Summary

Description here...
EOF
)"
