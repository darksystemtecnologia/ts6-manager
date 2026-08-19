# syntax=docker/dockerfile:1.4
FROM node:20-slim AS base
RUN corepack enable && corepack prepare pnpm@9 --activate

# 1. Instalação do Python3 + FFmpeg + Deno (Runtime JS) + yt-dlp
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      openssl \
      ffmpeg \
      python3 \
      ca-certificates \
      curl \
      unzip \
    && curl -fsSL https://deno.land/install.sh | sh \
    && mv /root/.deno/bin/deno /usr/local/bin/deno \
    && curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp \
    && chmod a+rx /usr/local/bin/yt-dlp \
    && apt-get purge -y --auto-remove curl unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 2. Estágio de Build
FROM base AS build
RUN apt-get update && apt-get install -y --no-install-recommends make g++ python3 && rm -rf /var/lib/apt/lists/*

COPY . .
RUN pnpm install --no-frozen-lockfile
RUN pnpm --filter @ts6/common run build
RUN pnpm --filter @ts6/backend exec prisma generate
RUN pnpm --filter @ts6/backend run build

# 3. Estágio de Produção
FROM base AS production
ENV NODE_ENV=production
WORKDIR /app

COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./package.json
COPY --from=build /app/pnpm-workspace.yaml ./pnpm-workspace.yaml
COPY --from=build /app/packages/common/dist ./packages/common/dist
COPY --from=build /app/packages/common/package.json ./packages/common/package.json
COPY --from=build /app/packages/backend/dist ./packages/backend/dist
COPY --from=build /app/packages/backend/generated ./packages/backend/generated
COPY --from=build /app/packages/backend/prisma ./packages/backend/prisma
COPY --from=build /app/packages/backend/package.json ./packages/backend/package.json
COPY --from=build /app/packages/backend/node_modules ./packages/backend/node_modules

WORKDIR /app/packages/backend
RUN mkdir -p data /data/music /root/.config/yt-dlp && \
    echo '--extractor-args "youtube:player_client=ios,android,web"' > /root/.config/yt-dlp/config && \
    echo '--no-check-certificates' >> /root/.config/yt-dlp/config

EXPOSE 3001
ENV SIDECAR_URL=http://ts6-sidecar:9800
CMD ["sh", "-c", "npx prisma db push --skip-generate && (npx prisma db seed || true) && node dist/index.js"]