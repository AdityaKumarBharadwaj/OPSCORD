# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# OPSCORD API Gateway — Multi-Stage Build
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

# ── Stage 1: Dependencies ───────────────
FROM node:20-alpine AS deps
WORKDIR /app

# Copy all workspace package.json files for npm ci
COPY package.json package-lock.json ./
COPY apps/api/package.json ./apps/api/
COPY apps/ingestion/package.json ./apps/ingestion/
COPY apps/web/package.json ./apps/web/
COPY packages/db/package.json ./packages/db/

RUN npm ci

# ── Stage 2: Build ──────────────────────
FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate --schema=packages/db/prisma/schema.prisma
WORKDIR /app/apps/api
RUN npx tsc

# ── Stage 3: Runtime ───────────────────
FROM node:20-alpine AS runtime
RUN addgroup -g 1001 -S opscord && adduser -S opscord -u 1001 -G opscord
WORKDIR /app

COPY --from=build --chown=opscord:opscord /app/apps/api/dist ./dist
COPY --from=build --chown=opscord:opscord /app/apps/api/package.json ./
COPY --from=deps --chown=opscord:opscord /app/node_modules ./node_modules
COPY --from=build --chown=opscord:opscord /app/node_modules/.prisma ./node_modules/.prisma

ENV NODE_ENV=production
ENV PORT=4000

USER opscord
EXPOSE 4000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:4000/api/v1/health || exit 1

CMD ["node", "dist/index.js"]
