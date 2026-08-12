# FROM node:22-alpine

# RUN npm install -g npm@12.0.2

# LABEL maintainer="Nasir"
# LABEL project="DevTrack"
# LABEL version="1.0.0"

# WORKDIR /devtrack

# COPY package*.json ./

# RUN npm install -g npm@12.0.2

# RUN apk add --no-cache wget \
#     && addgroup -S appgroup \
#     && adduser -S nasir -G appgroup \
#     && npm ci --omit=dev

# COPY --chown=nasir:appgroup . .

# ENV NODE_ENV=production

# EXPOSE 5000

# HEALTHCHECK \
#   --interval=30s \
#   --timeout=5s \
#   --start-period=10s \
#   --retries=3 \
#   CMD wget --quiet --tries=1 --spider http://localhost:5000/api/health || exit 1

# USER nasir

# # CMD ["npm", "start"]
# CMD ["node", "src/server.js"]# =========================
# STAGE 1 — BUILDER
# =========================
FROM node:22-alpine AS builder

WORKDIR /devtrack

COPY package*.json ./

RUN npm install -g npm@12.0.2 \
    && npm --version \
    && npm ci --omit=dev

COPY . .


# =========================
# STAGE 2 — PRODUCTION
# =========================
FROM node:22-alpine

LABEL maintainer="Nasir"
LABEL project="DevTrack"
LABEL version="1.0.0"

WORKDIR /devtrack

RUN apk add --no-cache wget \
    && addgroup -S appgroup \
    && adduser -S nasir -G appgroup \
    && rm -rf /usr/local/lib/node_modules/npm \
    && rm -f /usr/local/bin/npm /usr/local/bin/npx

COPY --from=builder --chown=nasir:appgroup /devtrack/node_modules ./node_modules
COPY --from=builder --chown=nasir:appgroup /devtrack/src ./src
COPY --from=builder --chown=nasir:appgroup /devtrack/package*.json ./

ENV NODE_ENV=production

EXPOSE 5000

HEALTHCHECK \
    --interval=30s \
    --timeout=5s \
    --start-period=10s \
    --retries=3 \
    CMD wget --quiet --tries=1 --spider \
    http://localhost:5000/api/health || exit 1

USER nasir

CMD ["node", "src/server.js"]