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
| **Containerization** | Docker + Tilt (dev), K8s (prod) |

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
├── k8s/                  # Kubernetes manifests (prod)
├── Tiltfile              # Tilt config for local dev
└── package.json          # pnpm workspaces root
```

## Local Development (Tilt)

Tilt orchestrates local dev with hot-reload for all services:

```bash
# Start everything
tilt up

# Dashboard at http://localhost:10350
```

### Tiltfile Overview
```python
# Tiltfile
docker_build('home-osx-api', './apps/api', dockerfile='./docker/Dockerfile.api')
docker_build('home-osx-web', './apps/web', dockerfile='./docker/Dockerfile.web')

# Live reload - sync code without full rebuild
docker_build('home-osx-api', './apps/api',
  live_update=[
    sync('./apps/api/src', '/app/src'),
    run('pnpm install', trigger=['./apps/api/package.json']),
  ]
)

# Port forwards
k8s_resource('api', port_forwards='3001:3001')
k8s_resource('web', port_forwards='3000:3000')
k8s_resource('postgres', port_forwards='5432:5432')
```

### Services in Dev
| Service | Port | Description |
|---------|------|-------------|
| web | 3000 | React frontend (Vite HMR) |
| api | 3001 | Hono backend |
| postgres | 5432 | PostgreSQL database |

## Deployment

- **Network Access:** Tailscale (private mesh VPN)
- **Infrastructure:** Mac Mini homelab cluster
- **Runtime:** Docker Compose initially, Kubernetes later
- **Domain:** Optional - accessible via Tailscale magic DNS

## Authentication

### Architecture
```
┌─────────────────┐         ┌─────────────────┐
│   Dumb Client   │         │    Hono API     │
│  (React App)    │         │                 │
├─────────────────┤         ├─────────────────┤
│                 │  POST   │                 │
│  Login Form ────┼────────►│  /api/auth/login│
│                 │         │       │         │
│                 │◄────────┼───────┘         │
│  Store JWT      │  JWT    │  Validate creds │
│  in memory      │         │  Return JWT     │
│                 │         │                 │
│                 │  GET    │                 │
│  Any Request ───┼────────►│  /api/*         │
│  Authorization: │         │       │         │
│  Bearer <jwt>   │         │  Auth Middleware│
│                 │         │  Validates JWT  │
│                 │◄────────┼───────┘         │
│                 │  Data   │                 │
└─────────────────┘         └─────────────────┘
```

### Principles
- **Stateless:** No server-side sessions, JWT contains all auth info
- **Dumb client:** Frontend only renders UI, all logic in API
- **Single login endpoint:** `POST /api/auth/login` returns JWT
- **Bearer token:** All requests include `Authorization: Bearer <token>`
- **Middleware validation:** Every `/api/*` route (except login) validates JWT

### JWT Structure
```json
{
  "sub": "user-uuid",
  "email": "user@example.com",
  "role": "admin" | "user",
  "iat": 1699900000,
  "exp": 1699986400
}
```

### User Model
```typescript
interface User {
  id: string;           // UUID
  email: string;
  firstName: string;
  lastName: string;
  passwordHash: string | null;  // null until first login
  role: 'admin' | 'user';
  createdAt: Date;
  updatedAt: Date;
}

interface Device {
  id: string;           // UUID
  userId: string;
  name: string;         // e.g., "iPad", "Calum's Mac"
  refreshToken: string;
  lastUsedAt: Date;
  createdAt: Date;
}
```

### Seed Users (Database Migration)
```typescript
// Always seeded on fresh DB
const seedUsers = [
  {
    email: 'calumpeterwebb@icloud.com',
    firstName: 'Calum',
    lastName: 'Webb',
    role: 'admin',
    passwordHash: null,  // Set on first login
  },
  {
    email: 'christie.cabus@yahoo.com',
    firstName: 'Christie',
    lastName: 'Cabus',
    role: 'user',
    passwordHash: null,  // Set on first login
  },
];
```

### First Login Flow
```
1. User enters email
2. If user exists AND passwordHash is null:
   → Show "Set your password" screen
   → User creates password (16 char min)
   → Save passwordHash, proceed to device naming
3. If user exists AND has password:
   → Normal login flow
4. If user doesn't exist:
   → "No account found" error
```

### Device Registration
On every new login (no valid refresh token for this device):
```
1. Login succeeds
2. Prompt: "Name this device" (e.g., "iPad", "Calum's Mac")
3. Save device with name + refresh token
4. Device name shown in "Who's logged in" widget later
```

### Endpoints
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/login` | No | Email/password → JWT + refresh |
| POST | `/api/auth/setup-password` | No | First-time password setup |
| POST | `/api/auth/refresh` | Refresh | Get new access token |
| POST | `/api/auth/register-device` | Yes | Save device name |
| GET | `/api/auth/me` | Yes | Get current user info |
| GET | `/api/auth/devices` | Yes | List user's devices |
| DELETE | `/api/auth/devices/:id` | Yes | Revoke a device |
| POST | `/api/admin/invite` | Admin | Create invite for new user |
| POST | `/api/auth/register` | Invite | Register with invite token |

### Token Strategy (Persistent Login)
```
Login returns:
├── Access Token (15 min) → used for API calls
└── Refresh Token (90 days) → used to get new access token

On API call:
1. Use access token
2. If 401 expired → call /api/auth/refresh
3. Get new access token, retry original request
4. If refresh fails → redirect to login
```

- **Access Token:** 15 min, stored in memory
- **Refresh Token:** 90 days, stored in localStorage (web) or Capacitor Secure Storage (iPad)
- **Result:** User logs in once, stays logged in for 90 days

### Token Storage
| Token | Web | iPad (Capacitor) |
|-------|-----|------------------|
| Access | Memory (React state) | Memory (React state) |
| Refresh | `localStorage` | `@capacitor/secure-storage` |

### Security
- JWT signed with HS256
- Access token: 15 min expiry
- Refresh token: 90 days expiry
- HTTPS only (Tailscale handles this)
- Rate limiting on login endpoint
- Refresh tokens stored in DB (can be revoked)
- Invite tokens single-use and time-limited

### Password Manager Compliance (1Password, etc.)

```html
<form autocomplete="on">
  <input type="email" name="email" autocomplete="email" />
  <input type="password" name="password" autocomplete="current-password" />
</form>
```

- Use proper `<form>` element
- Correct `type` and `autocomplete` attributes
- Never disable paste on password fields
- Registration uses `autocomplete="new-password"`

### Password Policy
- Minimum 16 characters
- No other complexity requirements
- Allow paste in password fields

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
- [ ] Home Assistant integration (if needed)

### Pinned for Later
- [ ] AI assistant
- [ ] "Who's logged in" widget on home screen (show active sessions/users)
- [ ] Email integration (for invites, notifications, etc.)
- [ ] Write Tailscale setup guide (how to configure for home-osx access)

### Research TODOs
- [ ] Check if WiFi router has an API we can call to get network data (connected devices, bandwidth, etc.)

## Target Device

- **Primary:** iPad Pro 12.9" 4th Generation (2732 × 2048 px @ 264 ppi)
- **Mode:** Kiosk mode via Capacitor native app
- **Mount:** Wall or desk mounted
- **Secondary:** Any web browser (laptops, phones)

## Design System

### Philosophy
Apple-inspired, minimal, purpose-built interface. Takes cues from "Sign in with Apple" button aesthetics - clean, confident, modern.

### Colors

```css
/* Backgrounds */
--bg-primary: #000000;        /* Pure black - main background */
--bg-elevated: #1C1C1E;       /* Elevated surfaces, cards */
--bg-secondary: #2C2C2E;      /* Secondary containers */

/* Text */
--text-primary: #FFFFFF;      /* Primary text */
--text-secondary: #8E8E93;    /* Secondary/muted text */
--text-tertiary: #48484A;     /* Disabled/hint text */

/* Accent Colors (bold, vibrant) */
--accent-blue: #0A84FF;       /* Primary actions */
--accent-green: #30D158;      /* Success, active states */
--accent-orange: #FF9F0A;     /* Warnings, attention */
--accent-red: #FF453A;        /* Errors, destructive */
--accent-purple: #BF5AF2;     /* Special features */
--accent-teal: #64D2FF;       /* Info, links */

/* Borders & Dividers */
--border-subtle: #38383A;     /* Subtle separators */
--border-prominent: #48484A;  /* More visible borders */
```

### Typography

- **Font Family:** SF Pro Display / SF Pro Text (system font on Apple devices)
- **Fallback:** -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif
- **Weights:** Regular (400), Medium (500), Semibold (600), Bold (700)

| Element | Size | Weight |
|---------|------|--------|
| Page Title | 34px | Bold |
| Section Header | 22px | Semibold |
| Card Title | 17px | Semibold |
| Body | 17px | Regular |
| Caption | 13px | Regular |
| Button | 17px | Semibold |

### Spacing & Layout

- **Base unit:** 8px
- **Corner radius:** 12px (cards), 10px (buttons), 8px (inputs), 20px (modals)
- **Card padding:** 16px
- **Grid gap:** 16px
- **Safe areas:** Respect iPad safe areas for edge content

### Components

#### Cards / Widgets
```
- Background: var(--bg-elevated)
- Border-radius: 12px
- Padding: 16px
- No borders (use elevation/color difference)
- Subtle shadow optional: 0 2px 8px rgba(0,0,0,0.3)
```

#### Buttons (Apple-style)
```
Primary:
- Background: var(--accent-blue) or white
- Text: white or black
- Border-radius: 10px
- Height: 50px
- Font: 17px Semibold
- Full-width on mobile, auto on desktop

Secondary:
- Background: var(--bg-secondary)
- Text: var(--text-primary)
- Same dimensions as primary
```

#### Input Fields
```
- Background: var(--bg-secondary)
- Border: none (or 1px var(--border-subtle) on focus)
- Border-radius: 10px
- Height: 50px
- Padding: 0 16px
- Placeholder: var(--text-tertiary)
```

### iPad 12.9" Layout

```
┌─────────────────────────────────────────────────────────┐
│  Status Bar Area (respect safe area)                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│   │ Weather │  │  Time   │  │  Music  │  │ Widget  │   │
│   │         │  │         │  │         │  │         │   │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│                                                         │
│   ┌───────────────────────┐  ┌───────────────────────┐ │
│   │                       │  │                       │ │
│   │    Large Widget       │  │    Large Widget       │ │
│   │    (e.g., Sonos)      │  │    (e.g., Lights)     │ │
│   │                       │  │                       │ │
│   └───────────────────────┘  └───────────────────────┘ │
│                                                         │
│   ┌─────────────────────────────────────────────────┐  │
│   │              Quick Actions Bar                   │  │
│   └─────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Animations

- **Duration:** 200-300ms for micro-interactions
- **Easing:** `cubic-bezier(0.25, 0.1, 0.25, 1)` (Apple's default)
- **Principles:** Subtle, purposeful, never distracting
- **Touch feedback:** Scale down slightly on press (0.97)

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
