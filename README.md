# Climate-ADAPT

Full stack for [climate-adapt.eea.europa.eu](https://climate-adapt.eea.europa.eu/) — Plone 6 backend + Volto frontend in a monorepo.

## Quick Start

```shell
make install        # Install backend + frontend dependencies
make stack-start    # Start the full Docker Compose stack
```

The site is available at http://cca.localhost.

### Backend (Plone)

Run inside the backend Docker container:

```shell
docker compose exec -it backend /bin/bash
# Then start Plone in the foreground
```

- Python: `/app/bin/python3`
- Port: `8080`
- Site ID: `cca`

### Frontend (Volto)

Run on the host in a separate terminal:

```shell
cd frontend && make start    # Dev server on localhost:3000
```

Uses `yarn` (not npm).

## Common Commands

| Where | Command | Description |
|-------|---------|-------------|
| root | `make stack-start` | Start Docker stack |
| root | `make stack-stop` | Stop Docker stack |
| root | `make install` | Install all dependencies |
| backend/ | `make test` | Run backend tests |
| backend/ | `make check` | Lint/format |
| backend/ | `make i18n` | Update locales |
| frontend/ | `make start` | Start Volto dev server |
| frontend/ | `make build` | Production build |
| frontend/ | `make test src/addons/volto-cca-policy` | Run Jest tests |

## Repository Layout

- **backend/** — Plone backend, add-ons in `sources/`, micro-services for translation
- **frontend/** — Volto frontend, main add-on at `src/addons/volto-cca-policy/`
- **devops/** — Docker Stack, Ansible, cache settings