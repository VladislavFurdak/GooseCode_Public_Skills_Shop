---
name: docker-compose-nextjs
description: Common pitfalls and fixes when building and running Next.js apps with Docker Compose on Windows/Linux. Covers .dockerignore, standalone output, port conflicts, init containers, volume mounts, and multi-platform builds.
---

# Docker + Next.js + Docker Compose: Pitfalls & Checklist

Use this skill BEFORE creating or modifying `Dockerfile`, `docker-compose.yml`, `.dockerignore`, or Next.js config for a containerized deployment.

---

## 1. `.dockerignore` — MANDATORY

Without it, `COPY . .` in the Dockerfile pulls in host `node_modules` (Windows `.exe` / macOS arm64 binaries) into the Linux container. This causes runtime crashes like:

```
You installed esbuild for another platform than the one you're currently using.
Specifically the "@esbuild/win32-x64" package is present but this platform needs "@esbuild/linux-x64".
```

**Minimum `.dockerignore`:**

```
node_modules
.next
.git
```

**Rule:** Always create `.dockerignore` before writing the Dockerfile.

---

## 2. Next.js Standalone Output

If the Dockerfile copies `.next/standalone` (which is the standard production pattern), `next.config.mjs` (or `.ts` / `.js`) **must** have:

```js
output: 'standalone',
```

Without it, `next build` does NOT create `.next/standalone/`, and the Docker build fails:

```
COPY --from=builder /app/.next/standalone ./: not found
```

**Checklist:**
- [ ] `next.config` has `output: 'standalone'`
- [ ] Dockerfile copies `.next/standalone`, `.next/static`, and `public`
- [ ] Dockerfile CMD is `["node", "server.js"]`

---

## 3. Port Conflicts on the Host

PostgreSQL default port 5432 is often occupied by a local PostgreSQL install. When Docker tries to bind it:

```
Bind for 0.0.0.0:5432 failed: port is already allocated
```

**Fix:** Map to a different host port in `docker-compose.yml`:

```yaml
ports:
  - '5433:5432'   # host:container
```

Inside the Docker network, services still connect to `postgres:5432` (the container port). Only the external mapping changes.

---

## 4. Database Migration & Seed Automation

Do NOT rely on manual `docker compose exec` commands after startup. Automate with an init service:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations/init.sql:/docker-entrypoint-initdb.d/0001_init.sql:ro
      # ↑ Postgres auto-runs SQL in this dir on FIRST start only (empty data dir)
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U user -d dbname']
      interval: 5s
      timeout: 5s
      retries: 5

  init:
    build:
      context: .
      dockerfile: Dockerfile
      target: deps            # ← use the stage with node_modules
    user: root
    restart: 'no'
    environment:
      DATABASE_URL: postgresql://user:pass@postgres:5432/dbname
    volumes:
      - ./drizzle:/app/drizzle:ro
      - ./src:/app/src:ro
    entrypoint: ['sh', '/app/docker/seed.sh']
    depends_on:
      postgres:
        condition: service_healthy

  app:
    depends_on:
      init:
        condition: service_completed_successfully
```

**Key points:**
- Postgres `docker-entrypoint-initdb.d/` scripts run **only** on first start (empty data volume). If you need to re-run migrations, do `docker compose down -v` to wipe the volume.
- The `init` service must use `target: deps` (not the final `runner` image) to have `node_modules`.
- Mount only source directories (`drizzle/`, `src/`), NOT the entire project root — see next section.

---

## 5. Volume Mounts in Init Containers — DO NOT MOUNT `.` OVER `node_modules`

The most common Docker-on-Windows mistake:

```yaml
# WRONG — mounts host Windows node_modules into Linux container
volumes:
  - .:/app:ro
```

This overwrites the Linux `node_modules` from `npm ci` with Windows binaries, causing esbuild/platform errors at runtime.

**Correct approach — mount only what the script needs:**

```yaml
volumes:
  - ./drizzle:/app/drizzle:ro
  - ./src:/app/src:ro
  - ./docker/seed.sh:/app/docker/seed.sh:ro
  - ./tsconfig.json:/app/tsconfig.json:ro
```

**Rule of thumb:** Never mount a directory that contains `node_modules` into a Linux container that already has its own `node_modules`.

---

## 6. `npx` vs Direct Command in Init Scripts

When a globally installed tool (e.g., `tsx`) is needed inside a container:

```sh
# WRONG — npx finds the LOCAL node_modules/.bin/tsx first (may have wrong platform binaries)
npx tsx drizzle/seed.ts

# CORRECT — uses /usr/local/bin/tsx (global install, correct platform)
tsx drizzle/seed.ts
```

`npx` resolves local `node_modules/.bin/` before global bins. If the local `node_modules` was contaminated by a host mount (see #5), `npx` picks up the wrong binary.

---

## 7. Alpine Images Lack CLI Tools

`node:20-alpine` does NOT include `pg_isready`, `psql`, `curl`, `wget`, or most system utilities.

**For health checks in non-Postgres containers**, use Node.js instead:

```sh
# WRONG — pg_isready not found in node:alpine
until pg_isready -h postgres; do sleep 1; done

# CORRECT — pure Node.js TCP check
until node -e "require('net').createConnection(5432,'postgres').on('connect',()=>process.exit(0)).on('error',()=>process.exit(1))"; do sleep 1; done
```

---

## 8. Full Checklist Before `docker compose up --build`

- [ ] `.dockerignore` exists with at least `node_modules`, `.next`, `.git`
- [ ] `next.config` has `output: 'standalone'`
- [ ] No port conflicts (check `netstat -tlnp | grep <port>`)
- [ ] Database migration SQL is in `docker-entrypoint-initdb.d/` OR in an init service
- [ ] Seed/migration init service uses `target: deps` (not `runner`)
- [ ] Init service volumes mount only source files, NOT the root `.` directory
- [ ] Init scripts use `tsx` (not `npx tsx`) for globally installed tools
- [ ] Init scripts use Node.js-based waits (not `pg_isready`) in Alpine images
- [ ] `docker compose down -v` before first run if you need a clean database
- [ ] `app` service depends on `init` with `condition: service_completed_successfully`

---

## Common Fix Patterns

### Port conflict
```yaml
ports:
  - '5433:5432'
```

### Standalone output
```js
// next.config.mjs
const nextConfig = {
  output: 'standalone',
  // ...
};
```

### Init service (clean pattern)
```yaml
init:
  build:
    context: .
    dockerfile: Dockerfile
    target: deps
  user: root
  restart: 'no'
  volumes:
    - ./drizzle:/app/drizzle:ro
    - ./src:/app/src:ro
  entrypoint: ['sh', '-c', 'npm install -g tsx && tsx drizzle/seed.ts']
  depends_on:
    postgres:
      condition: service_healthy
```
