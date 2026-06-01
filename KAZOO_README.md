# Kazoo Technology — Typebot Self-Hosted Deployment

This is a fork of [baptisteArno/typebot.io](https://github.com/baptisteArno/typebot.io) maintained by [Kazoo Technology](https://kazootechnology.com) for our Turtle Learn product line.

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Tracks upstream `main` (read-only, sync periodically) |
| `kazoo-patches` | Our custom deployment patches on top of the latest stable release |

**Update process:**
1. Fetch latest tags from upstream: `git fetch origin --tags`
2. Merge new release into `kazoo-patches`: `git merge v3.x.x`
3. Test deployment before pushing
4. Evaluate if update is worth merging — skip if it breaks our setup

## Our Deployment

### Stack
- **Typebot:** Official Docker images (`baptistearno/typebot-builder:latest` + `typebot-viewer:latest`)
- **Database:** Dedicated Supabase project "Typebot" (ref: `qbykxptrcygxfrknrisc`, Singapore)
- **Hosting:** Azure VM (`ubuntu-playground-restored`) via Tailscale Funnel (public HTTPS)
- **Redis:** Local instance on the VM (protected-mode disabled for Docker access)

### Access URLs
- **Builder:** `https://ubuntu-playground-restored.tail695fce.ts.net:8443`
- **Viewer:** `https://ubuntu-playground-restored.tail695fce.ts.net:8444`
- **OpenClaw Gateway:** `https://ubuntu-playground-restored.tail695fce.ts.net` (port 443)

### File Locations (on VM)
- Docker Compose + .env: `~/typebot-docker/`
- Old source build (reference): `/tmp/typebot/`

## Patches Applied

### 1. Google OAuth PKCE + State Fix (source patch)
**File:** `packages/auth/src/lib/providers.ts`, `packages/auth/src/lib/nextAuth.ts`
**Why:** AuthJS v5 beta has an `iss missing` bug. Added PKCE and state checks.
**Status:** Only needed for source-based builds. Docker images work without this patch.

### 2. Onboarding Skip
**How:** Set `termsAcceptedAt` on the admin user in the database after first login.
**Why:** Docker Typebot's `NEXT_PUBLIC_ONBOARDING_TYPEBOT_ID` env var is required (>= 1 char). Any value triggers the onboarding redirect if `termsAcceptedAt` is null.
**Fix:** SQL: `UPDATE "User" SET "termsAcceptedAt" = NOW() WHERE email = 'admin@example.com';`

### 3. Dedicated Ports (not path-based routing)
**Why:** Tailscale Serve path-based routing (`/typebot/` → port X) does NOT work with Next.js apps. Internal links (OAuth callbacks, API calls) resolve relative to `/` and break.
**Fix:** Each service gets its own port:
- Port 8443 → Typebot Builder
- Port 8444 → Typebot Viewer
- Port 443 → OpenClaw Gateway

## Configuration Reference

### docker-compose.yml
```yaml
services:
  typebot-builder:
    image: baptistearno/typebot-builder:latest
    restart: always
    ports:
      - "8080:3000"
    env_file: .env
    environment:
      DATABASE_URL: postgresql://postgres.<project-ref>:<password>@aws-1-ap-southeast-1.pooler.supabase.com:5432/postgres
      REDIS_URL: redis://host.docker.internal:6379
    extra_hosts:
      - "host.docker.internal:host-gateway"

  typebot-viewer:
    image: baptistearno/typebot-viewer:latest
    restart: always
    ports:
      - "8081:3000"
    env_file: .env
    environment:
      DATABASE_URL: postgresql://postgres.<project-ref>:<password>@aws-1-ap-southeast-1.pooler.supabase.com:5432/postgres
      REDIS_URL: redis://host.docker.internal:6379
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

### .env (required)
```env
ENCRYPTION_SECRET=<32-char random string>
NEXTAUTH_URL=https://<your-domain>:8443
NEXTAUTH_SECRET=<random string>
ADMIN_EMAIL=<your-email>
NEXT_PUBLIC_VIEWER_URL=https://<your-domain>:8444
NEXT_PUBLIC_ONBOARDING_TYPEBOT_ID=none
GOOGLE_AUTH_CLIENT_ID=<google-client-id>
GOOGLE_AUTH_CLIENT_SECRET=<google-client-secret>
```

## Gotchas & Lessons Learned

1. **Supabase pooler ports:**
   - Port `5432` = session mode — **required for Prisma migrations** (first run)
   - Port `6543` = transaction mode — migrations hang, use for runtime only

2. **Redis in Docker:** Must disable protected mode so containers can connect:
   ```
   redis-cli CONFIG SET protected-mode no
   ```

3. **Docker user group:** After `usermod -aG docker $USER`, use `newgrp docker` or start a new session.

4. **Google OAuth redirect URIs:** Must include the exact port in the URL:
   - Authorized redirect URI: `https://<domain>:8443/api/auth/callback/google`
   - Authorized JavaScript origin: `https://<domain>:8443`

5. **Prisma schema:** Official Docker images have Prisma hardcoded to `public` schema. Cannot use custom schema isolation — use a dedicated Supabase project instead.

6. **Container restart vs recreate:** `docker compose restart` does NOT regenerate `__ENV.js`. Use `docker compose down && docker compose up -d` for env changes to take effect.

## Quick Deploy (fresh machine)

```bash
# 1. Install Docker
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker

# 2. Install Docker Compose plugin (if not included)
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose

# 3. Set up Typebot
mkdir -p ~/typebot-docker && cd ~/typebot-docker
# Copy docker-compose.yml and .env (see above)
# Edit .env with your values

# 4. Start
docker compose up -d

# 5. Wait for migrations (~2 min), then check
docker logs typebot-docker-typebot-builder-1 --tail=5

# 6. Expose via Tailscale
tailscale funnel --bg http://127.0.0.1:18789
tailscale funnel --bg --https=8443 http://127.0.0.1:8080
tailscale funnel --bg --https=8444 http://127.0.0.1:8081

# 7. First login — skip onboarding
# Sign in, then run SQL to accept terms:
# UPDATE "User" SET "termsAcceptedAt" = NOW() WHERE email = 'your@email.com';
```
