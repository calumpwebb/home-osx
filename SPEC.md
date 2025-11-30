# home-osx

A personal home dashboard and control panel for iPad/web.

## Overview

Dark mode interface designed for wall or desk mounting on an iPad Pro 12.9" (4th gen), with full web access from any device. Open source, self-hosted on homelab.

## Tech Stack

| Layer | Technology |
|-------|------------|
| **Frontend** | React 18 + TypeScript (strict) + Vite |
| **Backend** | Hono + TypeScript (strict) |
| **Database** | PostgreSQL |
| **ORM** | Drizzle |
| **Auth** | Email/password, invite-only, JWT sessions |
| **Monorepo** | pnpm workspaces |
| **iPad App** | Capacitor (WebView wrapper) |
| **Containerization** | Docker Compose (K8s-ready) |

## Architecture

```
home-osx/
├── apps/
│   ├── web/              # React frontend (Vite)
│   └── api/              # Hono backend
├── packages/
│   └── shared/           # Shared types, validation schemas (Zod)
├── ios/                  # Capacitor iOS project
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.web
│   └── docker-compose.yml
├── k8s/                  # Kubernetes manifests (future)
└── package.json          # pnpm workspaces root
```

## Deployment

- **Network Access:** Tailscale (private mesh VPN)
- **Infrastructure:** Mac Mini homelab cluster
- **Runtime:** Docker Compose initially, Kubernetes later
- **Domain:** Optional - accessible via Tailscale magic DNS

## Authentication

- Email/password authentication
- Invite-only registration (admin creates accounts)
- JWT-based sessions
- All routes protected by default

## Features

### MVP / Phase 1
- [ ] User authentication (login/logout)
- [ ] Home screen with weather widget
- [ ] Dark mode UI
- [ ] Responsive design (iPad + web)

### Phase 2 - Integrations
- [ ] Sonos integration (control speakers)
- [ ] Spotify integration (via Sonos or direct)
- [ ] Weather API integration

### Phase 3 - Expansion
- [ ] Additional smart home integrations
- [ ] AI assistant (pinned for later)
- [ ] Home Assistant integration (if needed)

## Target Device

- **Primary:** iPad Pro 12.9" 4th Generation
- **Mode:** Kiosk mode via Capacitor native app
- **Mount:** Wall or desk mounted
- **Secondary:** Any web browser (laptops, phones)

## Development

```bash
# Install dependencies
pnpm install

# Start development
pnpm dev

# Build for production
pnpm build

# Run with Docker
docker-compose up
```

## License

Open source (TBD - MIT or similar)
