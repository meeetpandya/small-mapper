# Smart Indoor/Outdoor Navigation System — Production Backend Setup Guide

This guide gives a **complete, implementable backend blueprint** for building a smart navigation platform for campuses, office complexes, and event venues.

---

## 1) 📐 System Architecture

### 1.1 High-level architecture (text diagram)

```text
┌──────────────────────┐
│      Web Client      │
│ (React/Next on Vercel│
└──────────┬───────────┘
           │
           │ 1) Login (OAuth2)
           ▼
┌──────────────────────┐
│        Clerk         │
│ Google/GitHub/Email  │
└──────────┬───────────┘
           │
           │ 2) JWT (session token)
           ▼
┌──────────────────────────────────────────────────────┐
│              Node.js + Express REST API             │
│------------------------------------------------------│
│ Middleware:                                          │
│ - clerkAuth() JWT verification                       │
│ - RBAC (admin/user)                                  │
│ - rate limiting                                      │
│ Services:                                            │
│ - Location service                                   │
│ - Routing service (A*/Dijkstra)                      │
│ - Cache service                                      │
└──────────┬──────────────────────┬────────────────────┘
           │                      │
           │ 3a) graph + metadata │ 3b) cached routes/hot reads
           ▼                      ▼
┌──────────────────────┐   ┌──────────────────────────┐
│  NeonDB (Postgres)   │   │   Upstash Redis          │
│ nodes/edges/zones    │   │ route:{src}:{dst}:{opts} │
│ users/roles          │   │ location-list caches      │
└──────────────────────┘   └──────────────────────────┘
```

### 1.2 Request flow with auth

1. User logs in via Clerk (Google/GitHub/Email).
2. Clerk issues JWT session token to frontend.
3. Frontend calls Express with `Authorization: Bearer <token>`.
4. Express verifies JWT with Clerk middleware.
5. API performs RBAC check (`admin` for write endpoints).
6. API tries Upstash cache (for routes/location lists).
7. On cache miss, reads NeonDB, computes route (A*/Dijkstra), writes cache.
8. Returns response with path metadata and ETA.

### 1.3 Why this design

- **NeonDB/Postgres**: strong relational integrity for graph + zones + admin edits.
- **Upstash Redis**: low-latency cache for repeated path queries.
- **Clerk**: managed OAuth + identity; backend handles JWT verification only.
- **Express modular architecture**: easy scale into microservices later.
- **Graph model in DB**: supports indoor/outdoor transitions and constraints.

---

## 2) 🗄️ Database Design (NeonDB)

> Use UUID keys, PostGIS-free default approach (simple lat/lng doubles). You can later upgrade to PostGIS.

### 2.1 Core entities

- `app_users`: app-level profile, links to Clerk user ID.
- `buildings`: building metadata.
- `zones`: floor/area segmentation (indoor or outdoor).
- `nav_nodes`: route graph nodes (waypoints, doors, stairs, rooms).
- `nav_edges`: weighted directed edges between nodes.
- `locations`: POI destinations (washroom, cafeteria, event room, etc.).

### 2.2 SQL schema

```sql
-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Users linked to Clerk
CREATE TABLE app_users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_user_id TEXT UNIQUE NOT NULL,
  email TEXT UNIQUE NOT NULL,
  display_name TEXT,
  role TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Buildings
CREATE TABLE buildings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  address TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Zones (e.g., Building A Floor 2, Outdoor Quad)
CREATE TABLE zones (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  building_id UUID REFERENCES buildings(id) ON DELETE SET NULL,
  name TEXT NOT NULL,
  level INTEGER, -- floor number; null for outdoor
  zone_type TEXT NOT NULL CHECK (zone_type IN ('indoor', 'outdoor')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Navigation graph nodes
CREATE TABLE nav_nodes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  zone_id UUID REFERENCES zones(id) ON DELETE SET NULL,
  node_type TEXT NOT NULL CHECK (
    node_type IN ('walkway', 'door', 'stairs', 'elevator', 'room_entry', 'poi')
  ),
  label TEXT,
  latitude DOUBLE PRECISION,
  longitude DOUBLE PRECISION,
  x_coord DOUBLE PRECISION, -- optional indoor local coordinate
  y_coord DOUBLE PRECISION, -- optional indoor local coordinate
  is_accessible BOOLEAN NOT NULL DEFAULT TRUE,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Directed weighted graph edges
CREATE TABLE nav_edges (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  from_node_id UUID NOT NULL REFERENCES nav_nodes(id) ON DELETE CASCADE,
  to_node_id UUID NOT NULL REFERENCES nav_nodes(id) ON DELETE CASCADE,
  distance_meters DOUBLE PRECISION NOT NULL CHECK (distance_meters > 0),
  expected_seconds INTEGER NOT NULL CHECK (expected_seconds > 0),
  edge_type TEXT NOT NULL DEFAULT 'walk' CHECK (edge_type IN ('walk', 'stairs', 'elevator', 'ramp')),
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  is_accessible BOOLEAN NOT NULL DEFAULT TRUE,
  conditions JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (from_node_id, to_node_id)
);

-- Named locations users search for
CREATE TABLE locations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  zone_id UUID REFERENCES zones(id) ON DELETE SET NULL,
  node_id UUID NOT NULL REFERENCES nav_nodes(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  category TEXT NOT NULL CHECK (
    category IN ('hackerspace', 'food', 'washroom', 'rest_area', 'event_room', 'office', 'other')
  ),
  description TEXT,
  tags TEXT[] NOT NULL DEFAULT '{}',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_by UUID REFERENCES app_users(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 2.3 Indexes and optimization

```sql
CREATE INDEX idx_app_users_clerk_user_id ON app_users(clerk_user_id);
CREATE INDEX idx_locations_category_active ON locations(category, is_active);
CREATE INDEX idx_locations_name_trgm ON locations USING gin (name gin_trgm_ops);
CREATE INDEX idx_nav_nodes_zone ON nav_nodes(zone_id);
CREATE INDEX idx_nav_edges_from_active ON nav_edges(from_node_id) WHERE is_active = TRUE;
CREATE INDEX idx_nav_edges_to_active ON nav_edges(to_node_id) WHERE is_active = TRUE;
CREATE INDEX idx_nav_edges_accessible ON nav_edges(is_accessible, is_active);
CREATE INDEX idx_nav_nodes_accessible ON nav_nodes(is_accessible);
```

> Also run: `CREATE EXTENSION IF NOT EXISTS pg_trgm;` before trigram index.

---

## 3) 🔐 Authentication Flow (Clerk OAuth2)

### 3.1 Clerk OAuth login flow (Google example)

1. Frontend renders Clerk sign-in.
2. User chooses Google.
3. Google OAuth consent handled by Clerk.
4. Clerk creates/links user and issues session token (JWT).
5. Frontend includes token in Bearer header for API calls.
6. Express verifies token via Clerk middleware.

### 3.2 Clerk setup steps

1. Create Clerk app.
2. Enable providers:
   - Google OAuth (**required**)
   - GitHub OAuth (**recommended**)
   - Email/password (**fallback**)
3. Configure allowed redirect URLs for frontend domain(s).
4. Copy keys:
   - `CLERK_PUBLISHABLE_KEY` (frontend)
   - `CLERK_SECRET_KEY` (backend)
5. Configure JWT template / authorized parties if required.

### 3.3 Express JWT verification middleware

Install:

```bash
npm i @clerk/express
```

`src/middleware/auth.js`

```js
import { clerkMiddleware, requireAuth, getAuth } from '@clerk/express';

export const clerkAuthMiddleware = clerkMiddleware();

export const requireAuthenticated = requireAuth();

export function attachUserContext(req, res, next) {
  const auth = getAuth(req);
  req.auth = {
    userId: auth.userId,
    sessionId: auth.sessionId,
    orgId: auth.orgId,
    claims: auth.sessionClaims || {},
  };
  next();
}
```

### 3.4 Role-based access (admin vs user)

Store role in `app_users.role`. You can sync this from Clerk webhooks or on-demand at request time.

`src/middleware/rbac.js`

```js
export function requireRole(...allowed) {
  return async (req, res, next) => {
    try {
      const user = await req.services.userService.findByClerkId(req.auth.userId);
      if (!user) return res.status(403).json({ error: 'User profile not provisioned' });
      req.currentUser = user;
      if (!allowed.includes(user.role)) {
        return res.status(403).json({ error: 'Insufficient role' });
      }
      next();
    } catch (err) {
      next(err);
    }
  };
}
```

### 3.5 Public vs protected example

- Public: `GET /health`, `GET /api/v1/locations`
- Protected user: `POST /api/v1/routes/compute`
- Admin only: `POST /api/v1/locations`

---

## 4) 🧭 Navigation Logic

### 4.1 Graph model

- Node = waypoint/door/room entry/POI
- Edge = traversable segment with cost
- Cost = `expected_seconds` (primary), distance optional secondary

### 4.2 A* vs Dijkstra

- Use **A*** if geographic coords available (faster with heuristic).
- Fall back to **Dijkstra** for dense indoor local graphs without strong heuristic.

### 4.3 Routing service (Node.js snippet)

`src/services/routingService.js`

```js
class MinHeap {
  constructor() { this.data = []; }
  push(item) { this.data.push(item); this.#up(this.data.length - 1); }
  pop() {
    if (!this.data.length) return null;
    const top = this.data[0];
    const end = this.data.pop();
    if (this.data.length) {
      this.data[0] = end;
      this.#down(0);
    }
    return top;
  }
  get size() { return this.data.length; }
  #up(i) {
    while (i > 0) {
      const p = Math.floor((i - 1) / 2);
      if (this.data[p].f <= this.data[i].f) break;
      [this.data[p], this.data[i]] = [this.data[i], this.data[p]];
      i = p;
    }
  }
  #down(i) {
    const n = this.data.length;
    while (true) {
      let l = i * 2 + 1;
      let r = i * 2 + 2;
      let m = i;
      if (l < n && this.data[l].f < this.data[m].f) m = l;
      if (r < n && this.data[r].f < this.data[m].f) m = r;
      if (m === i) break;
      [this.data[m], this.data[i]] = [this.data[i], this.data[m]];
      i = m;
    }
  }
}

function heuristic(a, b) {
  if (a.latitude == null || b.latitude == null) return 0;
  const dx = a.latitude - b.latitude;
  const dy = a.longitude - b.longitude;
  return Math.sqrt(dx * dx + dy * dy) * 111000 / 1.4; // sec estimate
}

export async function computeShortestPath({ graphRepo, fromNodeId, toNodeId, accessibleOnly = false }) {
  const { nodesById, adjacency } = await graphRepo.loadGraph({ accessibleOnly });
  if (!nodesById[fromNodeId] || !nodesById[toNodeId]) throw new Error('Invalid nodes');

  const open = new MinHeap();
  const gScore = new Map([[fromNodeId, 0]]);
  const prev = new Map();

  open.push({ nodeId: fromNodeId, f: heuristic(nodesById[fromNodeId], nodesById[toNodeId]) });

  while (open.size) {
    const { nodeId } = open.pop();
    if (nodeId === toNodeId) break;

    const edges = adjacency[nodeId] || [];
    for (const edge of edges) {
      const candidate = (gScore.get(nodeId) ?? Infinity) + edge.expected_seconds;
      if (candidate < (gScore.get(edge.to_node_id) ?? Infinity)) {
        gScore.set(edge.to_node_id, candidate);
        prev.set(edge.to_node_id, nodeId);
        const f = candidate + heuristic(nodesById[edge.to_node_id], nodesById[toNodeId]);
        open.push({ nodeId: edge.to_node_id, f });
      }
    }
  }

  if (!gScore.has(toNodeId)) return null;

  const path = [];
  let cur = toNodeId;
  while (cur) {
    path.push(cur);
    cur = prev.get(cur);
    if (cur === fromNodeId) {
      path.push(cur);
      break;
    }
  }

  path.reverse();
  return {
    nodePath: path,
    totalSeconds: gScore.get(toNodeId),
    etaMinutes: Math.ceil(gScore.get(toNodeId) / 60),
  };
}
```

---

## 5) 🔌 API Design (REST)

Base path: `/api/v1`

### 5.1 Public endpoints

#### `GET /health`
Response:

```json
{ "status": "ok", "service": "small-mapper-api" }
```

#### `GET /locations?category=food&zoneId=<uuid>&q=cafe`
Response:

```json
{
  "items": [
    {
      "id": "loc_uuid",
      "name": "Main Cafeteria",
      "category": "food",
      "nodeId": "node_uuid",
      "zoneId": "zone_uuid"
    }
  ]
}
```

### 5.2 Protected (authenticated)

#### `POST /routes/compute`
Headers: `Authorization: Bearer <clerk_jwt>`

Request:

```json
{
  "fromNodeId": "uuid",
  "toNodeId": "uuid",
  "accessibleOnly": true
}
```

Response:

```json
{
  "path": ["nodeA", "nodeB", "nodeC"],
  "totalSeconds": 420,
  "etaMinutes": 7,
  "cached": false
}
```

### 5.3 Protected (admin)

#### `POST /locations` (admin only)

Request:

```json
{
  "zoneId": "uuid",
  "nodeId": "uuid",
  "name": "Hackerspace Alpha",
  "category": "hackerspace",
  "description": "Open 9 AM - 8 PM",
  "tags": ["makerspace", "3d-printer"]
}
```

Response:

```json
{
  "id": "new_location_uuid",
  "message": "Location created"
}
```

---

## 6) ⚙️ Express Backend Setup

### 6.1 Project initialization

```bash
mkdir small-mapper-api && cd small-mapper-api
npm init -y
npm i express cors helmet morgan dotenv zod
npm i pg ioredis
npm i @clerk/express
npm i express-rate-limit
npm i -D nodemon
```

`package.json` scripts:

```json
{
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js"
  }
}
```

### 6.2 Folder structure (scalable)

```text
src/
  app.js
  server.js
  config/
    env.js
    db.js
    redis.js
  middleware/
    auth.js
    rbac.js
    errorHandler.js
  routes/
    health.routes.js
    locations.routes.js
    routing.routes.js
  controllers/
    locations.controller.js
    routing.controller.js
  services/
    user.service.js
    location.service.js
    routingService.js
  repositories/
    graph.repository.js
    location.repository.js
    user.repository.js
  utils/
    asyncHandler.js
```

### 6.3 Environment variables

`.env`

```env
NODE_ENV=development
PORT=8080
DATABASE_URL=postgresql://<user>:<pass>@<neon-host>/<db>?sslmode=require
UPSTASH_REDIS_REST_URL=https://<...>.upstash.io
UPSTASH_REDIS_REST_TOKEN=<token>
CLERK_SECRET_KEY=sk_live_xxx
CLERK_PUBLISHABLE_KEY=pk_live_xxx
CLERK_JWT_ISSUER=https://<your-clerk-domain>
```

### 6.4 Express app wiring

`src/app.js`

```js
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import rateLimit from 'express-rate-limit';

import { clerkAuthMiddleware } from './middleware/auth.js';
import healthRoutes from './routes/health.routes.js';
import locationRoutes from './routes/locations.routes.js';
import routingRoutes from './routes/routing.routes.js';
import { errorHandler } from './middleware/errorHandler.js';

const app = express();

app.use(helmet());
app.use(cors({ origin: ['https://your-frontend.vercel.app'], credentials: true }));
app.use(express.json({ limit: '1mb' }));
app.use(morgan('combined'));
app.use(rateLimit({ windowMs: 60_000, max: 120 }));

app.use(clerkAuthMiddleware);

app.use('/health', healthRoutes);
app.use('/api/v1/locations', locationRoutes);
app.use('/api/v1/routes', routingRoutes);

app.use(errorHandler);

export default app;
```

`src/server.js`

```js
import 'dotenv/config';
import app from './app.js';

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log(`API running on port ${port}`);
});
```

### 6.5 Protected route example

`src/routes/routing.routes.js`

```js
import { Router } from 'express';
import { requireAuthenticated, attachUserContext } from '../middleware/auth.js';

const router = Router();

router.post('/compute', requireAuthenticated, attachUserContext, async (req, res, next) => {
  try {
    const { fromNodeId, toNodeId, accessibleOnly } = req.body;
    const result = await req.services.routingService.compute({
      fromNodeId,
      toNodeId,
      accessibleOnly: !!accessibleOnly,
      requesterClerkId: req.auth.userId,
    });
    res.json(result);
  } catch (err) {
    next(err);
  }
});

export default router;
```

### 6.6 Admin route example

`src/routes/locations.routes.js`

```js
import { Router } from 'express';
import { requireAuthenticated, attachUserContext } from '../middleware/auth.js';
import { requireRole } from '../middleware/rbac.js';

const router = Router();

router.get('/', async (req, res, next) => {
  try {
    const data = await req.services.locationService.list(req.query);
    res.json({ items: data });
  } catch (err) {
    next(err);
  }
});

router.post('/', requireAuthenticated, attachUserContext, requireRole('admin'), async (req, res, next) => {
  try {
    const created = await req.services.locationService.create({
      ...req.body,
      createdByClerkUserId: req.auth.userId,
    });
    res.status(201).json({ id: created.id, message: 'Location created' });
  } catch (err) {
    next(err);
  }
});

export default router;
```

---

## 7) ⚡ Caching Strategy (Upstash Redis)

### 7.1 What to cache

1. **Route results** (high impact):
   - Key: `route:{from}:{to}:a11y:{0|1}`
2. **Location list queries**:
   - Key: `loc:list:{category}:{zone}:{q}`
3. **Graph snapshot** (optional):
   - Key: `graph:active:{version}`

### 7.2 TTL strategy

- Route cache: **30–120 seconds** for dynamic venues, or up to 10 min for static environments.
- Location list: **60–300 seconds**.
- Graph snapshot: invalidate on admin edge/node updates.

### 7.3 Node integration example

```bash
npm i @upstash/redis
```

`src/config/redis.js`

```js
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});
```

`src/services/routeCache.service.js`

```js
import { redis } from '../config/redis.js';

export async function getCachedRoute(key) {
  return redis.get(key);
}

export async function setCachedRoute(key, value, ttlSeconds = 120) {
  await redis.set(key, value, { ex: ttlSeconds });
}
```

Routing usage:

```js
const cacheKey = `route:${fromNodeId}:${toNodeId}:a11y:${accessibleOnly ? 1 : 0}`;
const cached = await getCachedRoute(cacheKey);
if (cached) return { ...cached, cached: true };

const computed = await computeShortestPath(...);
await setCachedRoute(cacheKey, computed, 120);
return { ...computed, cached: false };
```

---

## 8) 🚀 Deployment Guide

### 8.1 NeonDB setup

1. Create Neon project + database.
2. Copy pooled connection string with `sslmode=require`.
3. Run migrations (`psql` or migration tool like Prisma/Drizzle/Knex).
4. Seed buildings/zones/nodes/edges/locations.

### 8.2 Upstash setup

1. Create Redis database in Upstash.
2. Copy `REST_URL` + `REST_TOKEN`.
3. Add env vars to backend deployment platform.

### 8.3 Clerk setup

1. Create Clerk application.
2. Enable Google + GitHub + Email/password.
3. Add production frontend URL (Vercel) in allowed origins/redirects.
4. Add backend API URL to trusted origins.
5. Set `CLERK_SECRET_KEY` in backend environment.

### 8.4 Backend deployment (example: Render/Railway/Fly.io)

1. Push backend code to GitHub.
2. Create service and connect repo.
3. Build command: `npm ci`
4. Start command: `npm start`
5. Add env vars (`DATABASE_URL`, Upstash, Clerk, CORS origin).
6. Configure health check path: `/health`
7. Enable auto deploy on main branch.

### 8.5 Frontend deployment reference (Vercel)

1. Deploy frontend on Vercel.
2. Configure env:
   - `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
   - `NEXT_PUBLIC_API_BASE_URL`
3. Add Vercel URL to backend CORS allowlist.

---

## 9) 🔒 Security & Scalability

### 9.1 Security baseline

- Verify every protected request with Clerk JWT middleware.
- Never trust client role claims alone; fetch role from DB.
- Use `helmet`, strict CORS, JSON body size limits.
- Enforce HTTPS only in production.
- Keep secrets in platform env vars, never in git.
- Use parameterized SQL queries.

### 9.2 Abuse prevention

- Global rate limiting (`express-rate-limit`).
- Stricter rate limits on `/routes/compute` and admin endpoints.
- Add IP + user-id level throttling in Redis for precision.

### 9.3 Scalability pattern

- Stateless API containers (horizontal scaling).
- Cache route responses aggressively.
- Add async worker for graph rebuild/versioning if map updates are frequent.
- Use read replicas (Neon branching/replica strategy) for heavy read workloads.

---

## 10) 🧪 Testing & Extensions

### 10.1 API testing strategy

- **Unit tests**: routing algorithm correctness (`fromNode -> toNode`).
- **Integration tests**: route compute endpoint with test DB + mocked Clerk token.
- **Contract tests**: OpenAPI schema validation.
- **Load tests**: verify p95 latency under concurrent route requests.

Suggested tools:
- Jest or Vitest
- Supertest for API
- k6 for load

### 10.2 Minimum test checklist

1. Public location listing works without auth.
2. Protected route compute rejects missing/invalid JWT.
3. Admin location create rejects non-admin users.
4. Cache hit path returns `cached=true` and lower latency.
5. Route recomputes after edge deactivation.

### 10.3 Future enhancements

- **Indoor precision**: BLE beacons/WiFi RTT for live positioning.
- **Real-time updates**: WebSocket/SSE for closures/crowding reroutes.
- **Multi-modal routing**: accessible routes, stairs-avoid mode.
- **Admin dashboard**: map editor to update nodes/edges/POIs.
- **Analytics**: anonymous heatmaps for congestion-aware guidance.

---

## Production rollout checklist

- [ ] Neon schema + indexes migrated
- [ ] Seeded graph + locations
- [ ] Clerk OAuth providers enabled
- [ ] Backend JWT verification tested
- [ ] RBAC admin/user tested
- [ ] Upstash caching enabled + TTL tuned
- [ ] Rate limiting enabled
- [ ] Health checks + logs configured
- [ ] Load test baseline recorded
- [ ] Incident rollback plan documented

If you implement this structure exactly, you’ll have a backend that is secure, scalable, and practical for real-world small-scale smart navigation deployments.
